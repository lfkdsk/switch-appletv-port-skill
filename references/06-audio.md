# 06 · Audio: the two audren routes and debugging "no sound"

## 6.1 Two audren routes, both proven

Game audio goes through Horizon's AudioRenderer (audren): every frame the game produces a command buffer (voice state, mixes, effects); the host executes it and feeds PCM to AudioOut. Two host implementations:

| | Self-written audren (nx route) | SDK's in-process renderer (smbw route) |
|---|---|---|
| Approach | Command parsing and mixing implemented against Ryujinx (MIT, pinned commit), pure C11 | Don't implement it — run the game SDK's own renderer in-process (CPU rendering + manual execution); the host finds the DeviceSink in the command table and takes the mix buffer, upmixed to stereo |
| Strengths | Full control of behavior; fuzz-verifiable | No reverse engineering of mix semantics; SDK-faithful behavior |
| Costs | Must implement REV1–15 wire formats and semantics | Two SDK 20 adaptations (see 6.4, 6.5); render-thread races to handle |
| Verification | Semantic tests + fuzz: 58,362,601 updates without a crash | On-device RMS measurement (see 6.3) |

Selection guidance: first-party games on new SDKs with new command formats (ACNH uses REV15) mean a lot of semantics to reimplement — third-party games on old SDKs are cheaper to self-write. The pitfalls below apply to either route.

## 6.2 Self-written audren: the essentials

- Scheduling: 48 kHz, 240 frames, 5 ms quantum; CLOCK_MONOTONIC absolute-deadline pump; counts for rendered frames/quantums/events/underruns/overruns.
- Capability surface: PCM16 + GC-DSP ADPCM (predictor/scale/history/loop context, bit-exact against an independent golden per sample); 4 wave buffers per voice, finite/infinite loops, EOS, pause/resume/flush, zero-fill on starvation; voice→submix DAG→final mix; per-frame volume ramps; all 9 effect types parsed, dry-bypassed per processing_order.
- REV15 additions: splitter (destination + bias ramp, caps 256/1024), global mix-buffer budget 102 (ACNH's actual value), voice SRC/pitch (linear interpolation, spec-permitted), depop.
- REV14/15 revision numbers are capability bits only — they don't change the wire format. Don't be scared by the revision number; measure the actual section sizes first.

## 6.3 "Callbacks running ≠ there's sound": measure RMS

The most typical audio pitfall the ACNH port hit on Apple TV: ~358,000 sound-card callbacks, zero underruns, zero zero-fill ms — everything looked fine — but the log said "200 frames rendered, 0 with sound". Every frame's PCM was dropped at command validation; the sound card got silence.

Forensics (measured after the fix):

| Metric | Number |
|---|---|
| Frames with sound (302 s opening) | 58,586 / 60,000 |
| RMS | min −44.2 / median −28.9 / max −23.5 dBFS |
| Spectral flatness | 0.002 (white noise ≈ 0.56; musical tones well under 0.1) |
| L/R correlation | 0.94–0.96 |
| 240-sample frame-boundary step ratio | ≈ 1.0 (gapless frame stitching) |

Conclusion: callback counts and underrun counters prove nothing. Measure PCM RMS, check the spectral shape to confirm music rather than noise, and listen. Compare before/after with the same metrics.

The audio review caught two lessons more general than the bug itself:

- **No logging in real-time callbacks.** Calling `LOG` (`fprintf` plus a stdio lock) in AudioUnit's render callback can block the real-time thread on a lock — low log frequency doesn't change that. Fix: double-buffered atomic level stats — the callback only accumulates sum-of-squares/peak/sample count; a normal thread swaps buffers, waits for the old buffer's writers to drain, then computes and logs.
- **RMS numbers must be reproducible from a repo script.** The review recomputed independently from the retained PCM and found a reported "overall RMS −17.7 dBFS" that didn't match the raw PCM (actually −27.028 dBFS) — a line was mis-stitched when assembling the full-run value from 2 s log rows. Fix: the analysis script lives in the repo (reads s16le interleaved PCM, converts with `audioop.rms`'s integer semantics, without depending on `audioop` which was removed from modern Python); reports give the full command; reviews rerun it.

## 6.4 SDK 20's DeviceSink command number: 0x13 → 0x14

Symptom: no sound on Apple TV; sound-card callbacks running fine, zero underruns (see above).

Root cause: SDK 20 changed the DeviceSink command number from 0x13 to 0x14. The host hard-coded 0x13 and read a different command (0x13 appears dozens of times per frame); field validation failed and the whole frame was dropped. The protective validation blocked the out-of-bounds read, so the result was silence, not a crash.

Fix: the command number is a game-profile entry (ACNH profile: 0x14; default 0x13 keeps old games working). Measured command table: 74 entries; DeviceSink is the last one (0x148 bytes); the table header layout matches SDK 16; only the command number changed.

One more check: when the Apple TV has a Bluetooth audio device connected, sound goes to Bluetooth only and the TV speakers stay silent — for "no sound", check `AVAudioSession`'s `currentRoute.outputs` first, not just the host.

## 6.5 SDK 20 audio render-thread race

Symptom: minutes of island walking, then a guaranteed crash in the audio render thread, with command-header magic in the stack.

Root cause: under SDK 20 the host's chase-render (manual render execution) calls the public `ExecuteAudioRendererRendering` without the host interception layer — no lock, no render counting, concurrent writes to the same command buffer as the render thread.

Fix: on the SDK 20 path the render thread holds the lock, counts renders, and Update is mutually exclusive; chase-render dropped from every update to 3 times total. 2,632 s crash-free after the fix.

## 6.6 REV15 wire format and the test trap

ACNH uses REV15. The self-written route's real REV15 wire format differs from the old revision: voice sections are 0x188 (old 0x170), SNDD 0xA8 (old 0x90) — 0x18 larger each. Parsing with the old layout rejected legal REV15 packets; the game went through SetTerminateResult + svcBreak and crashed before first frame.

Fix: select v1/v2 layouts by update/behavior revision; derive the splitter section's stride from the "SNDD" magic; during transition, keep a logged silent fallback on the host side (one count line per 1,000 hits).

The test trap (worth more than the bug itself): the REV15 test fixtures were actually REV14-structured. Fixture and production parser shared the "voice is always 0x170" assumption, so tests passed self-consistently and fuzzing couldn't mutate a legal 0x188 packet. Fix:

- Physically separate test layout tables from production constants;
- Byte-level assertions on size/offset/stride;
- State fields tested across at least two updates (semantics survive an update);
- Mutation self-check: break one piece of production code and confirm the test goes red.

## 6.7 Movie audio: the game opens AudioOut itself; the host only decodes PCM

ACNH's opening movie is WebM (VP9 + Vorbis). Key fact: **the game doesn't expect the system to play it for it** — it hands the audio track to the host as a second MediaCodec stream, takes the PCM16 back, and opens its own `nn::audio::AudioOut` to append it; video is aligned to the host's timestamps too. So the host's job isn't "play sound" — it's demuxing one extra audio track and one extra PCM decoder.

Implementation points:

- The two WebM tracks interleave into one sample table in file order; report honestly which track each sample belongs to. The cursor must never rest on a non-selected track, or the game's frame-fetch thread chokes on an unknown track number.
- Vorbis decodes to PCM16 with vendored libvorbis (BSD); Configure takes the three setup headers from the track config.
- Safety switch: codec not A_VORBIS, missing CodecPrivate, wrong channel count/sample rate — any of these falls back to "report video only" (movie plays, silent). Reporting a track you can't decode makes the game wait forever; better not to report it.
- The movie player calls `nn::audio::SetAudioOutVolume` every frame; not taking it over crashes (the host's AudioOut is an opaque handle, and the SDK's own implementation null-derefs on it). After takeover, volume is applied on the way to the sound card.

Pitfall hit: the decode thread unconditionally VP9-decoded every stream — the audio track took 203 packets in and produced 0 frames, and the game gave up on the movie after 3 s. Routing by stream type to the Vorbis decoder fixed it.

Measured (Mac, full 65.47 s): RMS −23.3 dBFS, peak −8.6 dBFS, spectral flatness 0.000 (music, not noise); video 1,959 frames, audio 4,564 packets — not one missing; during the movie the game itself stops the audren mix (cutscene audio comes from the movie, same as on original hardware), and it resumes normally before and after. The tvOS build compiles; on-device listening isn't verified — the one thing Mac can't check is underruns and Bluetooth routing with two AudioOut streams coexisting; left for on-device re-test.

## 6.8 hwopus

ACNH imports 8 `nn::codec` Hardware variants (voice chat). Two options: the rewrite layer replaces the constructor with the SDK's own software decoder (same vtable interface, object fits), or host libopus HLE. During the offline-semantics phase, UNIMPL is fine.
