# 05 · Performance: what's slow and how it was fixed

## 5.1 First frame 236 s → 1.7 s: not PSO's fault

| Item | Number |
|---|---|
| First frame before the fix (Apple TV, fresh install) | 236,672 ms |
| After | 1,709 ms; second launch 1,392 ms |
| Mac comparison | 68,204 ms → 1,155 ms |

Diagnosis: measure PSO compile time first — before first frame, Metal compiled only 1–2 pipelines + 2 shader libraries (Mac 47.7 ms, 0.07%; Apple TV 8.5 ms, 0.2%). Ruled out. A 60 s folded stack sample showed both loading threads spending almost all their time in `getxattr` (5,620 calls).

Root cause: Horizon paths are case-sensitive; APFS isn't. The host's case-exact lookup, on "name doesn't exist as-is", lists the directory and asks `getxattr` per entry (~40 µs/entry). The game's opening probes for file existence, repeatedly full-scanning the same ten-thousand-entry directories.

Two-layer fix:

1. No-alias directory cache: a scan that finds no "physical name ≠ guest name" entries records the directory as clean, cutting 109 full scans (−232.6 s).
2. Ask xattr only for alias entries: 6 scans of 60,195 entries dropped from 2,455.6 ms to 44.7 ms (−2.35 s).

Together the two layers account for 99.3% of the total win.

Lesson: when first frame is slow, take a stack sample first — don't assume PSO. Hearing too many mobile-game PSO stories builds a mental rut.

## 5.2 Prewarm manifest vs offline pipeline archives

| Approach | Saves what | Numbers | Status |
|---|---|---|---|
| Prewarm manifest | Post-first-frame opening stutter, not first frame | Manifest 1,733 lines (651 shaders / 1,075 render pipelines / 7 compute pipelines; line order = first-use order); on Mac, 3 minutes on the island: synchronous pipeline creation 147 → 3 (−98%), synchronous metallib loads 59 → 0; cold-cache full prewarm 2.7 s (background; first-present costs only +20 ms), warm cache 84 ms | Shipped |
| Offline AOT archive | The cold-cache prewarm compile itself | tvOS 26 SDK's `metal-tt` can produce binary archives for A15 on Mac; minimal sample works; missing a pso_skey→mtlp-json generator (est. 1 day of work) | Feasible, not shipped |

Two caveats:

- The prewarm manifest does nothing for first frame — first frame uses 2 pipelines either way (1,392/1,412 ms with/without).
- Device-harvested archives only work on machines with the same GPU and OS build; on mismatch Metal silently falls back to runtime compilation.
- The manifest is recorded on Mac and applies to tvOS: ACNH's manifest merges multiple play sessions by first-appearance order with dedup; the old 980 lines are kept first in order, and island content is prewarmed in the background after the opening. On A15 hardware, the old manifest's 1,428 entries prewarmed in 473.8 ms; the full 1,733-entry on-device time wasn't measured.

The manifest has a second use (nxnative): record live-compiled shaders/pipelines while playing into `pso_user.tsv`, prewarm in the background at next launch; manifests are named by content key and portable across devices.

## 5.3 Frame rate and bottleneck location

| Scene | Machine | Numbers |
|---|---|---|
| Indoor dialogue | Apple TV | p50 25.5–26.1 ms / p95 30.3–31.2 ms (game present interval 2 = 30 fps cap) |
| Opening movie | Apple TV | p50 16.7 ms |
| Island outdoors, walking (before) | Apple TV | p50 84.4 ms / p95 93.2 ms (~11–12 fps) |
| Island outdoors (after the pool-translation fix) | Apple TV | p50 27.8–28.9 ms / p95 35–38 ms (30 fps) |
| Island dock (before) | Mac M5 Pro | p50 33.32 ms (30 fps cap), ~2,442 draws/frame, submit CPU 18.9 ms |
| Plaza (before) | Mac M5 Pro | p50 40.4 ms (~25 fps), 5,433 draws/frame, submit CPU 33.1 ms |

The outdoor bottleneck (Apple TV): ~3,300–4,300 draws per frame; before the fix each draw cost ~130k host instructions, 90% of it address translation — the replay layer linearly scanned all memory pools from scratch on every guest→Metal buffer translation (~3,000 pools on the ACNH island; 13–17 translations per draw). The dirty-page tracker's range lookup did the same linear scan over ~3,000 ranges. GPU p50 was only 26 ms — **the bottleneck was CPU submit, not the GPU**.

Fixes (all in the replay layer, invisible to the game):

- Pools stored sorted by start; binary search for translation. A 4,096-entry direct-map table in front (pool index per 64 KiB block; boundaries checked on hit). Pools don't overlap — an existing game invariant; a registered overlapping pool reports red and falls back to linear scan, with identical results.
- Dirty-page range lookup gets a per-thread small table (512 entries); the range table invalidates stale records by generation on register/unregister; hits return exactly what a linear scan would return now.
- Upload staging blocks reused across batches by slot, instead of a fresh 32 MB allocation per batch — island streaming textures re-upload every frame, and the old path paid a 32 MB allocation plus residency register/unregister per frame.

Measured after: on the Apple TV, instructions per outdoor draw 130k → ~11k; per-frame submit CPU 57–61 → 6.5–7.2 ms; frame time p50 84 → 28 ms; footprint peak 1,824 MiB. Mac island submit CPU 19.6 → 2.8 ms.

Two directions tried with no on-device gain, left off by default in the profile:

- **Translation worker thread** (per d2r-mobile v1.0.15: game threads only enqueue, a dedicated thread replays): ~6% on Mac with uncapped frame rate; no A/B gain on the Apple TV — once submit is down to 7 ms the submit thread is off the critical path (GPU p95 ~26–27 ms; game logic runs on other threads).
- **TBDR write-back speculation** (see 5.4): no GPU time reduction on the Apple TV.

A correction: an earlier suspicion that "the translation thread has QoS 0 (not on performance cores)" was wrong — the thread runs a real-time policy (priority 97), and real-time policies don't set QoS by design. Once submit got off the critical path, the QoS experiment was dropped.

Lesson: when frame rate is low, measure three things first — **submit CPU time, GPU time, host instructions per draw** — then act. Assuming GPU or PSO here would have sent the work in exactly the wrong direction.

### Per-encoder redundant-state and argument-address dedup

Once per-draw instruction count was down to ~11k, the next layer of waste was re-setting Metal state and argument-buffer addresses that hadn't changed since the previous draw in the same render-command encoder. Cache per live encoder: last-set pipeline, viewport/scissor, depth/stencil state, cull/winding, polygon mode, bind-group argument-buffer base addresses, vertex-buffer offsets. On encoder end, invalidate. For each draw, compare against the cache before issuing the Metal call; skip when equal.

A/B must be byte-identical. Freeze a scene, record a replay command stream, then run cache-ON vs cache-OFF against the same stream and compare presented frames pixel-for-pixel. Paired medians across four 300-frame windows: instructions per draw −14.03%, submit CPU −9.41%, GPU p50 −10.76%, GPU p95 −7.90%. Frames byte-identical in all four windows.

Two things to watch:

- An encoder boundary (`endEncoding`) is a hard invalidate — don't carry cache state across encoders, even if the next encoder's first draw looks identical.
- Argument buffer "same base address" is only safe when the backing storage hasn't been rebuilt. Tag with a generation counter; bump on each rebuild.

### Frame pacing at 30 Hz: hold interval-2 for two display ticks

The game runs its logic at a 30 fps beat (present interval 2). On a 60 Hz display the host must hold each presented frame for exactly two display ticks; a lazy "present at the next tick" lets consecutive 16.7 ms submits land on the same vsync and skip one, producing visible 1-frame hitches.

Pacing rule:

- When the game's requested present interval is 2 and the display is 60 Hz, schedule each present `presentDrawable(afterMinimumDuration:)` for two display intervals; set the layer's `preferredFrameLatency` to 2.
- When the interval is 1 (60 Hz), set `preferredFrameLatency` to 1 and present at the next tick.
- Hysteresis: downgrade from 60 Hz to 30 Hz only after 1 s of consistently missing the 60 Hz budget; upgrade back to 60 Hz immediately on the first in-budget window. Short dips don't flip the mode.

Measured result at 30 Hz (same scene, same 1,530-frame window): present-to-present stddev 0.603 ms → 0.301 ms (−50%); frames above 40 ms 4/1530 → 1/1530. Cost: +33.335 ms submit-ahead latency (the price of scheduling two ticks ahead). 60 Hz paths keep latency 1 — they already have no headroom to trade.

## 5.4 TBDR and bindless

Background: nxnative's TBDR (write-back speculation) optimization is off by default, citing "doesn't recognize bindless handles". ACNH's bindless share is **0 / 9,738**, so that reason doesn't apply here — hence this round of work.

Correctness pitfall (more important than performance): with TBDR on, the whole 3D background went black. Write-back speculation treated **clears under conditional rendering** as evidence that "the whole thing gets overwritten later" — but that clear never actually executed per the GPU counters. The write-back was saved; so was the clear. Fix: when the render-enable state is uncertain or known-off at record time, don't index the clear for write-back speculation.

Performance:

| Machine | Numbers |
|---|---|
| Mac M5 Pro | Same scene: GPU p50 down ~20%, p95 down ~25% |
| Apple TV | TBDR on vs off, 30+ windows each: GPU p50 21.8 vs 19.9–21.2 ms — no reduction; profile default stays off |

Bindless adaptation: infer which textures a draw reads, uniformly; resolve bindless handles from recorded UBs; give up this pass's optimization when unresolved. Zero hot-path overhead.

Another observation: ACNH indoors runs 65 render passes per frame (13 at 1920×1080, 23 at 960×540) — a lot of full-screen passes; pass merging is future work.

## 5.5 Shader translation counts

| Route | Numbers |
|---|---|
| nx line | 23,739 variants dedup to 6,141 unique Maxwell programs (VS 3,117 / PS 2,991 / CS 29 / GS 4), all emitted; full-run wall time 75.3 s (10 workers) |
| smbw line | Shader table 6,536 → 9,742 (3,206 short-control-section entries added); 9,738 compile (4 GS failed, took the fallback); tvOS rebuilt 14,020 metallibs, 0 failures |

Short control sections are the usual reason tables don't line up: agl/gsys package control sections are only 0x848/0x854 bytes, and hashing a fixed 0x880 includes load-time-rewritten pointers, so every key differs (see 03).

## 5.6 Memory performance

- On-demand residency: pools stay out of the residency set until the first GPU address fetch, then commit. ~0.5 GB wired memory saved upstream (~550 MB / 25% for ACNH with the same mechanism).
- Texture pack: same resident working set 43.96 GiB → 7.31 GiB; the pack is pread-backed and costs no memory (see 04).
- Thread-exit release: fixed a leak where TLS/TLR/thread objects weren't freed on exit (600 blocks × 9.5 KB).

## 5.7 60 fps and game speed: fix fast-forward first, then talk speedups

The 60 fps outdoor experiment on the Apple TV concluded: **not possible right now — neither route holds**. 60 fps is a game-logic problem before it is a GPU problem.

**Game side: uncapping the present interval means the whole game runs at double speed.** ACNH's logic is frame-counted — typewriter dialogue, HID polling, and time conversion all advance per frame; only audio runs on wall clock (~199 updates/second, identical across runs). A replay of the same input script on Mac: uncapped, 180 s produced 5,100 → 10,500 frames; after one A press, characters typed in 0.35 s went 5 → 13; music didn't speed up. Normal speed would need changes on the scale of the community SpeedFix: ~760–7,381 code patches + ~1,300 `StaticParam.pack` values (1.5× for the 45 fps version, 4× for 120 fps) — converting frame-counted logic to time-stepped.

**GPU side: even the best outdoor mode exceeds the budget.** The bar is GPU p95 ≤ 15 ms and submit CPU ≤ 6 ms. Apple TV outdoor measurements:

| Combo | GPU p95 | Submit CPU |
|---|---|---|
| 1080p TBDR off | 26.62 ms | 7.37 ms |
| 1080p TBDR on | 26.33 ms | 8.47 ms |
| 720p TBDR off | 19.21 ms | 7.58 ms |
| 720p TBDR on (best mode) | 18.65 ms | 8.12 ms |

The best mode is still 12% over the 16.7 ms full-frame budget for 60 fps. The bottleneck is **draw count, not pixels**: 1080p→720p cuts pixels 55.6% but GPU time only 25%; TBDR saves ~10% of write-back bandwidth for 0% frame-time gain — A15's bandwidth isn't this frame's bottleneck. Outdoor scenes run 3,500–4,200 draws per frame; the counter scene only ~100, so light scenes do hit 60 on device (counter at 720p measured p50 16.81 ms = 59.5 fps, GPU p95 14.2 ms, submit CPU 7.4–8.5 ms) — but that doesn't extrapolate outdoors. The cost-effective order for chasing 60: dedicated replay thread (already implemented) → cut draw count (batching/culling) → game-side SpeedFix. 900p isn't reachable: the game has no external hook for "scale the 3D target proportionally", and the host has no generic scaler.

**Fixing on-device fast-forward matters more than 60 fps.** The present-interval mode implicitly followed "is there a window": Mac with no window used the game's own interval (30 fps beat); device with a window presented one frame per display beat — so the game ran at "actual fps / 30" speed (the pool-translation fix's 23 ms frames ≈ 1.4×; the earlier 84 ms frames ≈ 0.4×), while music stayed on wall clock. Fix: the present interval is a profile entry, pinned to `game` (30 fps beat) for ACNH. The uncap switch is a diagnostic tool only — it's not a frame-rate switch, it's a fast-forward switch; don't ship it to users as "experimental 60 fps".

Community cross-check: emulator/Atmosphere 60 fps patches say the same thing (uncapping alone = double speed; SpeedFix required); "real hardware 1080p only manages ~45 fps, force 720p for stable 60" matches the resolution findings here.
