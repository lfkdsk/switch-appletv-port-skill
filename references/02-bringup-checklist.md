# 02 · From NSP+keys to first frame: a checklist

Each stage lists the key actions and the done criteria. Numbers come from the ACNH project's measured reports. Pitfall details live in 03; this file is the checklist.

## Stage overview

| # | Stage | Key actions | Acceptance |
|---|---|---|---|
| 1 | Unpack | If hactool errors, switch to LibHac; overlay base game + update | Four NSOs + npdm all present |
| 2 | Survey | Imports, services, NVN APIs, textures, instruction scan | Two independent tools agree |
| 3 | Choose approach | Import-layer vs service-layer HLE; profile vs flags | Rationale written down |
| 4 | Rewrite + reconcile | Run the rewriter; loader handles DT_RELR | Scan count = rewrite count; unresolved strong imports = 0 |
| 5 | Synthetic guest | Build your own NSO, run the full pipeline | Pixel assertions pass on all three platforms |
| 6 | Service HLE | sm + HIPC framework + individual services | Every npdm service name gets a session |
| 7 | Audio core | Self-written audren or the SDK's in-process renderer | Semantic tests + fuzz green |
| 8 | Shaders | Full offline translation; GS fallback | All emitted, failures = 0 |
| 9 | Memory budget | Measure the jetsam limit with a probe | Gap between limit and working set is known |
| 10 | Texture pack | BC→ASTC/EAC | 0 failures; format-retyped views converted too |
| 11 | First present | Loading + services + NVN counting stub | First present returns status 0 |
| 12 | First frame | Real NVN→Metal replay | Frame is non-uniform; pixel assertions pass |
| 13 | Playable | Memory fit, controllers, save | Plays continuously on device without crashing |

## 1 Unpack

- Unpack NSP→NCA→ExeFS/RomFS with hactool. Update-package NCAs using the new BKTR layout make hactool fail with `Invalid BKTR layout!` — switch to LibHac (0.19.0) and write an equivalent CLI.
- Overlay base game + update with `OpenFileSystemWithPatch`; take the updated ExeFS and RomFS.
- Acceptance: ExeFS has all four NSOs (rtld/main/subsdk0/sdk) + main.npdm. ACNH measured: 103,400 RomFS files, 9.8 GB.
- Keys are passed to unpacking tools by path only — never printed, logged, or committed.

## 2 Survey

Measure before building. Survey items:

| Item | ACNH measured | Why it matters |
|---|---|---|
| SDK import symbols | 1,015 across 47 namespaces | Sizes the import-layer HLE work |
| NVN API usage | 521 (d2r base covers 216) | Sizes the graphics-layer work |
| npdm service access | 51 services | The service-HLE surface |
| Texture survey | 153,714 instances (BC 116,751 / ASTC 36,694) | Sizes texture-pack work and the VRAM budget |
| Instruction scan | x18/svc/sysreg site counts | Baseline for rewrite reconciliation |

Acceptance: cross-check key numbers with two independent tools (ACNH's BNTX texture survey was implemented twice, independently; agreement was the pass criterion).

## 3 Choose approach

Pick the HLE layer and multi-game strategy per the decision table in 01, and write the rationale into the design doc. The output of this step is one sentence: which layer, and why.

## 4 Rewrite + reconcile

- Run the rewriter over all modules. Unsupported instruction shapes must hard-fail — never silently skip.
- Reconcile rewrite counts against the survey scan, line by line (ACNH's four modules passed in one run).
- The loader must support DT_RELR: new SDKs pack relative relocations with it (ACNH has 391,693 slots). A loader that only knows RELA SIGSEGVs without it.
- Acceptance: zero unresolved strong imports in a load test.

## 5 Synthetic guest (P0)

Before touching any game file, build a minimal NSO of your own (clear the screen + draw a red triangle) and run the full chain: loader → instruction rewriting → HLE imports → guest thread → NVN→Metal output.

- Acceptance: pixel assertions pass on macOS, the tvOS simulator, and real Apple TV hardware (red center, fixed corner colors).
- This step exposes every base-layer pitfall on a minimal guest — an order of magnitude faster than debugging on the game.

## 6 Service HLE

- Build the HIPC/CMIF/TIPC marshalling framework + sm + handle table first, then fill in services one by one.
- Every service name in the npdm must return a session from sm. Unknown commands must get a response (an error code), not silence — silence makes the game wait forever.
- Online services get offline semantics across the board: return a no-network result the game accepts; don't implement real networking.
- Acceptance: ABI audit against real requests (ACNH audited 105 commands; all 7 blocking issues fixed).

## 7 Audio core

One of two routes (details in 06): self-written audren (implemented against a pinned Ryujinx revision), or the game SDK's in-process renderer. Acceptance: semantic tests + long fuzz green.

## 8 Full shader translation

- Don't run the full set until the chain works: synthetic samples → full translation → fix translator crashes → GS fallback.
- Acceptance: all shaders emitted. ACNH's two routes: nx line 6,141/6,141; smbw line 9,738/9,742 (4 GS failed, took the fallback).
- Archive translation outputs by toolchain version; the host loads only matching versions.

## 9 Memory budget

- Measure the per-app jetsam limit on device with a probe (Apple TV 4K 3rd gen ≈ 2.05 GiB; no entitlement raises it).
- Compare against the game's Switch application memory pool (ACNH ≈ 3.2 GB). The gap is what you must compress.
- Acceptance: a written budget — texture pack, memory pools, guest heap each take X, headroom Y.

## 10 Texture pack

- A15 and earlier have no BC texture decode. Convert all BC textures offline to ASTC/EAC (format choices in 04).
- Textures whose format was retyped in a view must be converted with their views. `GetStorageSize` keeps returning the original BC byte count.
- Acceptance: 0 failures. The lookup key is a hash of the texture content; the offline tool and the runtime implementation must agree byte-for-byte.

## 11 First present

Run NVN as a "counting + safe defaults" stub first. The goal is getting loading + services + the threading model all the way to the first `nvnQueuePresentTexture`.

- Acceptance: first present returns status 0. ACNH measured: Mac host 2,604 ms (177 NVN APIs and 111 IPC commands counted); real Apple TV 10,333 ms.

## 12 First real frame

Swap in the real NVN→Metal replay backend. This stage hits most of the graphics pitfalls in 03 — look them up by symptom.

- Acceptance: the frame is non-uniform. The nx route reached a 5-color UI at frame 300; semantic pixel assertions passed.

## 13 Playable

- Fit memory under the cap (on-demand residency + texture pack; see 04/05).
- Controllers map by position (see 04); saves land in Caches and handle disk-full.
- Verify the save loop end to end: play through day one → save and end → relaunch to the title → load back to the island, image matching the pre-save state (ACNH measured mean per-pixel diff 3.02/255 across the load; any visible change jumps to dozens). Device saves use transactional commit: clone the whole tree in a working directory, then atomically swap — a half-finished commit leaves trace files you can inspect.
- Acceptance: plays continuously on device without crashing. The smbw route measured 1,800 s continuous (self-exited on a timer).

## Milestone numbers at a glance

| Milestone | Machine | Number |
|---|---|---|
| First present (NVN stub) | Mac / Apple TV | 2,604 ms / 10,333 ms |
| First frame (non-uniform UI) | Mac | Frame 300 |
| First frame (fresh install) | Apple TV | 1,709 ms (optimized; see 05) |
| Indoor dialogue | Apple TV | p50 25.5–26.1 ms (≈30 fps cap) |
| Island outdoors | Apple TV | p50 27.8–28.9 ms (30 fps after PERF1; 84.4 ms before) |
| Continuous run | Apple TV | 1,800 s without a crash |
