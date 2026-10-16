# 04 · tvOS deep dive: where it differs from iOS/macOS

Apple TV is the tightest target in this port. Every claim carries a measured number.

## 4.1 Memory limit: a hard 2.05 GiB cap

| Item | Number | Notes |
|---|---|---|
| Per-app jetsam limit | ≈ 2.05 GiB | Without entitlements; two touch runs were both killed on the next step after 2,048 MB cumulative allocation — reproducible |
| Entitlements | Can't raise it | No entitlement on tvOS raises the physical limit: `com.apple.developer.kernel.increased-memory-limit` is iOS/iPadOS-only per Apple docs, and `com.apple.developer.kernel.extended-virtual-addressing` only enlarges the process address space, not the resident-memory ceiling. Unified memory means every Metal resource counts against the same budget |
| Runtime query | `os_proc_available_memory()` | Trustworthy on tvOS 27: reports 2.041 GiB at launch, 37.8 MiB one step before the kill |
| Virtual address (no EVA) | 6 GB per mmap, 7 GB cumulative | The 7.3 GiB texture pack can't be mmaped whole — read-only index + pread instead |
| Switch side comparison | ACNH application memory pool ≈ 3.2 GB | A ~1 GB gap that must be compressed/streamed into 2 GB |

Measured usage (Apple TV 4K 3rd gen, A15):

| Scene | Peak | Notes |
|---|---|---|
| First present (NVN counting stub, no real VRAM) | 710 MiB | A lower bound; doesn't predict final feasibility |
| Indoors (BC runtime decode path) | 1,377–1,494 MiB | Almost all post-first-present growth is Metal |
| 30-minute play session (texture pack enabled) | 1,744 MiB | Flat after reaching the island, no growth; minimum headroom 353 MiB |

The two big memory wins: on-demand pool residency (~0.5 GB saved, see 03) and the BC→ASTC/EAC texture pack (same resident working set 43.96 GiB → 7.31 GiB, see 4.5).

### Host 16 KiB pages vs 4 KiB guest pool protection

The dirty-page tracker calls `mprotect` on individual guest pool pages to catch CPU writes to GPU-visible pools (see 03 "Dirty-page tracker race"). Guest pools are 4 KiB-aligned; the Apple host page is 16 KiB. When four 4 KiB pools happen to share one 16 KiB host page, `mprotect`ing one changes the protection of all four, and the three neighbors silently take write faults they didn't ask for — the handler spins, or marks the wrong pool dirty and re-uploads unchanged bytes.

Fix: before `mprotect`-ing a pool page, walk the pools sharing its host page and wrap them (track their current PROT state, protect/unprotect the host page as a unit, and restore each neighbor's logical state when it next writes). The state machine stays NONE→BUSY→mprotect→PROT per logical pool page; the physical `mprotect` is keyed on host-page granularity. Costs a per-host-page hash and one extra pointer per pool.

### Re-measure on device after enabling an AOT pipeline archive

Shipping an AOT pipeline archive (see 5.2) saves cold-cache prewarm compile time but adds resident memory: on the Apple TV the measured footprint grew by **+177 MiB** after enabling it, dropping minimum headroom to **79 MiB** — close enough to the jetsam cap that another regression anywhere in Metal would start killing the app. Mac savings don't transfer: the Mac has gigabytes of headroom; the Apple TV doesn't. Always re-measure footprint on real hardware after enabling an archive, not just launch time; keep the "disable archive" env switch wired so the device can boot back to a known-good baseline during triage.

## 4.2 W^X and signed code pages

tvOS won't let you turn anonymous data pages into executable pages. The two routes:

- Put the four rewritten NSO modules into a signed Guest.framework (Mach-O section + vm_remap of signed RX pages).
- Or embed the rewritten modules in the executable's `__TEXT` and vm_remap the signed pages at runtime (this route needs no Guest.framework).

The ctr_el0 rewrite must happen offline, before signing — it can't be done at runtime on tvOS.

## 4.3 Watchdog and writable directories

- **Scene watchdog**: blocking the main thread for ~20 s gets you killed with 0x8BADF00D. `didFinishLaunching` returns immediately; unpacking and the guest host run entirely on background queues (survived 80 s of unpacking after the fix).
- **Only Caches/tmp are writable**: the app container's Documents is not writable (writing `/var/mobile/containers/...` fails in practice). Write everything to `Library/Caches` (NSCachesDirectory) or tmp.
- **Caches gets purged**: Caches is system-purgeable. The app runs a preflight at launch and fails loudly when critical data is missing — never assume it's still there.
- **env override order**: `Caches/env.txt` is read first; same-named keys, first one wins, and it overrides the bundled env wholesale. When multiple people share a device, someone else's env.txt left in Caches can silently disable your input scripts and screenshot configs — check this first when "config didn't apply".

## 4.4 Where RomFS lives

ACNH RomFS: 103,400 files, 9.78 GiB. Three options:

| Option | Verdict | Numbers |
|---|---|---|
| `Library/Caches/romfs` | Used during development | 21 chunks × ≤500 MiB + readiness marker; push 461–616 s; on-device expand 78.7–80.3 s; staging verifies file/byte counts, then publishes atomically |
| Inside the app bundle | Installable, readable, not purged by Caches; costs bundle size and install time | 10.7 GB bundle; build 79 s; install 593 s |
| LAN HTTP | Not implemented | The first two work, so it wasn't needed |

Plain per-file copy is not viable: CoreDevice's small-file transactions are slow, and dropped connections leave junk behind.

## 4.5 A15 has no BC texture decode → the ASTC/EAC pack

A15 doesn't support BC textures. Without replacement, the only option is runtime decode to uncompressed pixels, which blows the VRAM budget.

| Item | Number |
|---|---|
| Source scale | 116,751 BC instances; 81,741 unique after content dedup, 11.41 GiB |
| Output | balanced pack 7.315 GiB, 81,741 entries, 0 failures; encode 376 s (15 parallel) |
| Key | Original texture content hash (the NVN layer can't see BNTX/paths, only pool bytes); offline tool and runtime agree byte-for-byte |
| VRAM win | Same resident working set: runtime BC decode 43.96 GiB → pack 7.31 GiB (6.0×), 1.56× smaller than the original BC |

Format strategy:

| Source | Target | Why |
|---|---|---|
| BC1 | ASTC 8x8 | PSNR p50 42.52 dB |
| BC3/BC7 | ASTC 5x5 | PSNR p50 44.40 dB |
| BC5_SNORM (normal maps, 41% of BC bytes) | EAC_RG11Snorm | ASTC LDR has no signed mode; ASTC would crush [-1,1] to [0,1]. EAC matches block bytes, keeps exact semantics, needs no shader changes |
| BC4 | EAC_R11 | PSNR p50 52.79 dB |

Two easy-to-miss points: textures with format retyped in a view must get views built on the underlying format (see 03); the pack can't be mmaped whole — read-only index + pread.

## 4.6 CAMetalDisplayLink never fires

Symptom: the framework's default switches output to CAMetalDisplayLink at frame 3. On Apple TV the callbacks never arrive; the game idles on the fallback clock, output stops at the 3rd present, and the image freezes on a black frame.

Fix: on tvOS, the output thread self-fetches drawables by default (env switch to revert). This class of bug — "Apple TV only, simulator and Mac both fine" — can only be found on real hardware.

## 4.7 Controllers

- **Map by position**: Apple's GCExtendedGamepad face-button properties are fixed geometric positions, not cap letters. Mapping: `buttonB`→A, `buttonA`→B, `buttonY`→X, `buttonX`→Y, `buttonMenu`→+, `buttonOptions`→−. PS5: ○=A, ×=B; Switch Pro: by position (matches its labels).
- **Siri Remote fallback**: only when no full controller is connected — touchpad→left stick, click→A, Menu→B, play/pause→+.
- **Rumble**: HD Rumble values are sanitized (amplitude [0,1], low/high frequency ranges) → intensity/sharpness, delivered per grip via Core Haptics.
- **Verification status (in progress)**: mapping, HD Rumble, and hot-plug pass Linux unit tests, the tvOS simulator, and a shared-memory on-device self-test. On real hardware only the Siri Remote has been connected so far; full-button input and actual rumble feel on PS5 DualSense / Switch Pro controllers are **not yet verified** (pairing path: Settings → Remotes and Devices → Bluetooth). Simulator-only verification doesn't count.

## 4.8 Getting screenshots off the device

Apple TV has no screenshot command. The flow:

1. The app reads back its latest Metal frame and writes PNG + JSON to `Library/Caches` (frame number/interval/output size configurable via env).
2. Retrieve with `devicectl device copy from` (~1.4 s per file, app unaffected).
3. **Open every retrieved PNG yourself** and run semantic pixel assertions (a region should be a specific color/shape) — never compare by file hash alone.

Helper: the app exits on its own after writing screenshots or on a timer, so the runtime devicectl file service isn't slowed down — copy-from while the game runs gets slower and slower until it stalls (CPU pegged + ~18 GB of freshly written Caches).

## 4.9 Contending for one shared device

With one Apple TV:

- **Lease serialization**: build/device tasks carry an exclusive lease; only one task builds on the Mac or uses the device at a time.
- **Same bundle ID evicts**: when tasks share a bundle ID, `devicectl --terminate-existing` kills the other side's running app. Parallel work needs distinct bundle IDs. One task's device run was evicted by another's same-bundle-ID install during the ACNH project.
- **Shared build directories collide**: multiple tasks in one build directory overwrite each other's output (an unsigned build evicts the signed one, caught by codesign). Use per-worktree directories.
- **Check before installing**: the deploy script checks whether another "ours" app is running on the device before installing, and waits (default 600 s) or explicitly doesn't.
- **Leases can't stop scripts that don't check them**: during a release packaging task, another task used an old deploy script with `FORCE_OTHER=1` to install test builds over the same bundle ID — 4 of 6 install self-tests were evicted. Keep the mutual-check logic in the deploy script itself; don't leave `FORCE_OTHER`-style backdoors.
- **Identify processes by bundle ID, not executable name**: liveness probes and timeout kills use the "installed app URL → bundle ID" mapping (parameters ending in `.` match by prefix). Matching on a `…/DmTV.app/DmTV` string mistakes someone else's same-named executable for yours.
- **Device state**: `devicectl list devices` flips between `available (paired)` and `connected`; install/copy/launch work in both, and the gate only blocks unavailable.

## 4.10 The simulator can't run the graphics layer

Per d2r-mobile's official DmTV notes (it ships a Metal 4 launch-detection screen on purpose): the tvOS simulator GPU lacks Metal 4, isn't unified memory, and only supports argument buffers tier-1 — the NVN→Metal layer can't come up. All of our results come from real hardware; the graphics layer has never run in the simulator. Graphics acceptance is device-only; the simulator is only good for loading/services/logic.

## 4.11 Release build: one command for package + install + self-test

The ACNH project's release flow (`tv_release.sh`) is worth copying:

- **One command does everything**: incremental shader rebuild → package (~350 MB) → install on device → file-by-file RomFS verification (103,400 files; missing/sized-wrong chunks auto-pushed) → clean test residue out of Caches → self-test → confirm return to the home screen. RomFS stays in Caches, out of the bundle (see 4.4): the texture pack is in Caches anyway, so bundling RomFS only turns ~1 minute of packaging into ~10 minutes, and a failed install costs more.
- **Self-test verdicts are fully scripted**: first-frame time, timer exit, counter screenshot semantic assertions (7/7), audio levels (56 of 59 level rows above −60 dBFS), no input scripts in the log — all PASS or the build doesn't ship.
- **Pacing check**: the self-test also requires the present-interval mode to be `game`, and the median of per-300-frame-window p50s to be above 31 ms. A fast-forward build (counter p50 16.9 ms = double speed) fails this check outright. The verdict logic is validated both ways against real logs and deliberately corrupted ones (real logs PASS; display-mode/16.9 ms logs FAIL).
- **Reinstall keeps saves**: same-bundle-ID over-the-top install doesn't delete Caches. Saves live in `Caches/save*`, with tmp backups + NSUserDefaults receipts. Before backgrounding or exiting, the app asks the system for background time and waits for save backup to finish; if backup and receipt disagree, the next launch redoes it (`present-differs-from-receipt`). A save is only lost if both locations are purged.
- **Final sweep**: one more check outside the script — app not running, correct display name on device, `Caches/env.txt` byte-identical to the release version, input script is an empty file. The user's build sits at the home screen; tapping it starts a new game or their own save.
