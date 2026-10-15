---
name: switch-appletv-port
description: Distilled playbook for porting Nintendo Switch games to run natively on Apple platforms (iPhone/iPad/Mac/Apple TV) without CPU emulation — covering ARM64 instruction rewriting, HLE at the import or IPC layer, NVN→Metal, offline shader translation, and tvOS bring-up, with a symptom-indexed pitfall catalog for black-screen, crash, and no-audio failures; use when starting such a port or debugging one.
---

# Porting Switch Games Natively to Apple Platforms

## When to use this skill

- Running a Switch game (NSO/NSP) natively on iPhone, iPad, Mac, or Apple TV
- Debugging a port: black screen, crashes, no audio, jetsam kills, missing text, low frame rate
- Wiring NVN graphics calls to Metal, or translating Maxwell shaders offline
- Learning how projects like d2r-mobile and nxnative work and avoiding their pitfalls

Not applicable: full-system emulators (a different technology stack); GameCube/Wii-era games (no shared-ISA advantage; see the bellpad comparison in references/08).

## Mental model

1. **No CPU emulation.** The Switch CPU and Apple Silicon are both ARM64, so the game's NSO machine code runs as-is. Emulation's performance cost is unnecessary here. The real work is rewriting the handful of instructions that break under the Apple kernel (see 01).
2. **Rewriting is offline and reconcilable.** The rewriter touches only a few instruction classes (x18 shadow register, TLS pointers, svc syscalls, counter reads, cache-type registers). Rewrite counts must reconcile exactly against a disassembly scan.
3. **System services come from host HLE.** Games never touch the kernel directly; they get services through Horizon svc/IPC. The host can substitute at two layers: the SDK import layer, or the svc/IPC service layer. Both have been made to work; see 01 for the decision criteria.
4. **Graphics is record-and-replay, not just-in-time translation.** Games fetch every NVN function through a single symbol, `nvnBootstrapLoader`. The host records NVN commands into game-managed memory and replays them as Metal at submit time. Shaders are translated fully offline; runtime looks them up by hash.
5. **tvOS is a different animal.** Per-app memory cap around 2.05 GiB, code pages must be signed, only Caches/tmp are writable, A15 has no BC texture decode, and the simulator cannot run the graphics layer. Chapter 04 covers this.
6. **The content boundary is a hard line.** No game code, assets, or keys in the repo; engineering experience, API names, and service names are fine. This is both a compliance rule and a constraint on what evidence can appear in reports.

## Workflow: from NSP to first frame

| Stage | What to do | Acceptance |
|---|---|---|
| Unpack | NSP→NCA→ExeFS/RomFS; if hactool errors on new BKTR layouts, switch to LibHac | Four NSOs + npdm all present |
| Survey | Import symbols, service access, NVN APIs, texture formats and counts | Numbers from two independent tools agree |
| Choose approach | Import-layer vs service-layer HLE; single-game vs game profile | Write down the rationale; 01 has the decision table |
| Rewrite + reconcile | Run the rewriter; loader must handle DT_RELR | Unresolved strong imports = 0 |
| Synthetic guest | Build your own minimal NSO and run the full pipeline | Pixel assertions pass on all three platforms |
| Service HLE | sm + HIPC framework + individual services; online services get offline semantics | Every service name in the npdm gets a session |
| Shaders | Full offline translation; GS via compute fallback | All shaders emitted, failures = 0 |
| Texture pack | BC→ASTC/EAC (mandatory on A15) | 0 failures; format-retyped views converted too |
| First present | Loading + services + NVN counting stub first | First present returns status 0 |
| First frame | Real NVN→Metal replay | Frame is non-uniform; semantic pixel assertions pass |
| Playable | Fit memory under 2.05 GiB, controllers, save | Plays continuously on device without crashing |

Details and per-stage pitfalls are in references/02.

## Symptom lookup

| Symptom | Read first |
|---|---|
| Black screen, but presents happen | 03 graphics section: unrecorded CallCommands, NEAR_IS_MINUS_W, ReportCounter, conditional clears, typed views |
| Crashes at launch or after a few frames | 03 loading section: DT_RELR, core=-2, wrong heap size; 03 services section |
| Text missing on TV, fine on Mac | 03: `texture2d_array` vs 2D (A15 samples 0) |
| No audio, but the audio callback is running | 06: callback counts prove nothing; measure RMS; SDK 20 DeviceSink is 0x14 |
| Killed by the system on device | 04: 2.05 GiB cap, on-demand residency, texture pack |
| First frame takes minutes | 05: probably not PSO — likely case-insensitive filesystem lookups |
| Frozen on a black frame, no image | 04: CAMetalDisplayLink never fires on Apple TV |
| Low frame rate | 05: measure submit CPU / GPU / host instructions per draw first; ACNH outdoor bottleneck was a linear scan of ~3,000 memory pools per draw (130k→11k instructions, p50 84→28 ms) |
| Game runs at double speed, music doesn't speed up | 05: pin the present interval to `game` (30 fps beat); logic is frame-counted, so uncapping is a fast-forward switch |
| Black moiré on near objects (tree canopies) | 03: depth-EQUAL prepass needs `-fpreserve-invariance`; MSL `invariant` is ignored without the compiler flag |
| Abort right after "Save and end" | 03: `CalculateStandardUserSystemClockDifferenceByUser` must be HLE'd; the SDK aborts on a null time service object |

## references map

| File | Questions it answers |
|---|---|
| 01-route.md | Why the route is chosen this way |
| 02-bringup-checklist.md | What to do first, and how each step counts as done |
| 03-first-frame-pitfalls.md | How someone fixed this exact symptom |
| 04-apple-tv.md | How tvOS differs from iOS/macOS |
| 05-perf.md | What's slow, why, and how it was fixed |
| 06-audio.md | The audio side of the port |
| 07-agent-workflow.md | How to run this kind of project with multiple agents |
| 08-community.md | Who in the community did what, and what was borrowed |

## Hard rules

- **No game content.** No game code bytes, disassembly, shader source, or keys in the repo, reports, or logs. API names (`nvn*`, `nn::*`), service names, and command numbers are fine.
- **Only measured numbers.** Any cited number must trace to a specific report; mark in-flight work as such; don't write numbers you can't stand behind.
- **Device and simulator results are separate.** Simulator pass is not a device pass. A device-pass claim needs the command, exit code, and paths to retrieved logs/screenshots.
- **Look at every screenshot.** Retrieved PNGs must be opened and checked with semantic pixel assertions (a region should be a specific color), not compared by file hash.
- **Measure audio levels.** A running callback and zero underruns prove nothing; measure PCM RMS, and ideally listen.

## Operational tips for agents

- Read a project report's final VERDICT line first (DONE/PARTIAL/PASS/FAIL), then the tables.
- When porting a new game, run the survey scripts before writing implementation code; 02 has the checklist.
- With one device, do all device operations serially — parallel installs evict each other (see 04).
- Graphics-bug drill: TEXDUMP first (are textures present?), then PASSDUMP (were draws issued?), then match view types to shader declarations (03 has a full case study).
