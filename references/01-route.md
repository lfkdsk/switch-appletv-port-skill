# 01 · Route: why no CPU emulation, and the design choices

Written for engineers or agents who have never done this. Concepts first, then reasons.

## 1.1 Native execution: the shared ISA is the big free win

The Switch CPU is an ARM Cortex-A57 (ARMv8-A); Apple Silicon is also ARM64. The machine code in a game NSO can be fetched and executed directly on the Apple CPU — no binary translation, let alone full-system emulation.

This one fact shapes the whole project:

| Decision | Choice | Why |
|---|---|---|
| CPU | Native execution + offline rewriting of a few instructions | Shared ISA; emulation's cost buys nothing |
| System services | Host HLE | Games don't touch the kernel; the service surface is fixed and documented |
| Graphics | Record at the NVN interface, replay to Metal at submit | NVN is the game's only graphics entry point — one interception point |
| Shaders | Fully offline translation, runtime lookup | Running the translation toolchain on phone is impractical; the library ships with the app or is built on first run |

The cost is real too: the game's code was compiled for the Horizon kernel and a Tegra GPU, and a set of instructions and conventions don't hold on Apple platforms. They must be rewritten before loading. Rewriting is offline and reconcilable, not a runtime patch.

## 1.2 Which instructions get rewritten

Games concentrate platform-specific behavior in a few instruction classes. The rewriter touches only these:

| Instruction / pattern | What the game uses it for | Problem on Apple | Rewrite |
|---|---|---|---|
| x18 accesses | Horizon uses x18 as TLS / shadow register | Apple reserves x18 with different semantics | Indirect through a key-page slot |
| `mrs/msr tpidr_el0`, `tpidrro_el0` | Horizon TLS/TLR pointers | Different user-space semantics on Apple | Redirect to host KTLS/KTLR slots |
| `svc #n` | Horizon syscalls | svc is an illegal instruction on Apple | Jump to a host gate function that dispatches |
| `mrs cntpct/cntvct_el0` | System counter reads | Different frequency | Replace with host monotonic clock scaled to 19.2 MHz |
| `mrs ctr_el0` | Cache-line geometry for cache maintenance | Not exposed to user space on Apple — SIGILL | Replace offline with the constant (128 B cache line) |

Untouched: `fpcr/fpsr/nzcv/dczid_el0` are accessible at Apple EL0.

A reconciliation lesson: an automated scan misreported function-alignment nop padding as over 100,000 "unknown system register accesses"; the real count was 220. Scan output needs human review of its shape, not blind acceptance.

Measured rewrite scale (ACNH, four modules): main — 52,095 x18 sites + 36 svc; sdk — 15,559 x18; subsdk0 — 9,784; rtld — 9. Every count reconciled with the scan.

## 1.3 Where HLE lives: both layers have been made to work

Games reach system services two ways: calling SDK-exported `nn::*` functions (which do IPC internally), or issuing svc/IPC themselves. Host substitution has two corresponding layers:

| | Import-layer HLE | svc/IPC service-layer HLE |
|---|---|---|
| Approach | Load the sdk NSO, but replace its service functions wholesale at the import layer with host implementations | Run the real sdk NSO as-is (including its IPC marshalling); the host implements the Horizon server side |
| Strengths | Fast start: third-party games have a relatively small SDK call surface | Behavioral fidelity: the SDK's internal state machines stay intact; the service surface is fixed (~60 services) and can be implemented against Ryujinx |
| Costs | Each SDK generation's differences need reverse engineering; bypassing SDK state machines risks behavioral drift | The IPC framework is new code with a large debug surface; requires HIPC/CMIF/TIPC marshalling |
| Proven by | smbw route: 394 of main's 1,015 imports hooked, playable on the island | nx route: all 1,007 strong imports resolved by real sdk/subsdk0, unresolved = 0, reached first frame |

Selection criteria: first-party games (like ACNH) use `nn::*` throughout with a recent SDK — import-layer HLE would mean substituting a thousand-plus symbols, so the service layer is safer. Third-party games have a smaller SDK surface, so import-layer HLE is faster. When unsure, look at the survey numbers: few imports + old SDK → import layer; many imports + new SDK → service layer.

## 1.4 NVN→Metal: record-and-replay architecture

Games import exactly one NVN symbol, `nvnBootstrapLoader`, and fetch the other several hundred NVN APIs by name through it. Seizing that one symbol takes over the entire graphics entry.

Two layers:

1. **Interface layer**: when the game calls an NVN API, the host records the command (object creation, state setup, draw calls) into game-managed command memory. No Metal involved.
2. **Replay layer**: at submit, the host walks the command stream and replays it into a Metal command buffer.

Why split this way: NVN calls are multithreaded and high-frequency; doing Metal encoding at the submit point overlaps with game threads. Upstream later moved replay+encoding to a dedicated translation thread, overlapping game logic by one frame.

Key design points:

- **Zero-copy memory pools**: on unified-memory devices GPU address equals CPU address, so graphics memory pools are plain MTLBuffers — no copies. Unified-memory devices only (A14/M1 and up).
- **Object sizes follow the SDK headers**: host-recorded NVN object sizes must match the game's SDK headers (e.g. NVNtextureView is 16 bytes). Oversized writes bleed into adjacent memory, showing up as inf/nan on screen. Add a static assertion per object.
- **On-demand residency**: pools are not placed in the Metal residency set at creation; the first GPU address fetch makes them resident and commits the set immediately. Measured savings upstream: ~0.5 GB of wired memory.
- **Pipelines cached by state hash**, compiled in the background at launch from a prewarm manifest (see 05).

## 1.5 Shaders: fully offline translation

Switch game shaders are Maxwell machine code (BNSH containers), which Metal doesn't understand. The toolchain:

```
Maxwell machine code → Ryujinx translator (vendored, MIT) → SPIR-V → spirv-cross → MSL → xcrun metal/metallib
```

Key points:

- **The key is a hash of the control-section content.** The BNSH control section describes resource bindings (textures, samplers, constant-buffer slot numbers); the machine code alone can't recover them. Translation outputs are keyed by control-section hash and looked up at runtime. Short control sections (0x848/0x854 bytes) are hashed only over their self-reported length — otherwise load-time pointer rewrites make every hash different.
- **Facts the machine code can't recover go in a sidecar**: stage, slot numbers, compute workgroup sizes, as JSON shipped with the library.
- **Version the toolchain**: hash the chain's source + options + tool fingerprints; the host loads only libraries matching its version, so a translator change can't silently consume stale output.
- **Geometry shader fallback**: Metal has no geometry stage. ACNH's 4 GS (all ocean surface) go through a compute prepass: vs_prepass + gs_prepass + vs_passthrough, three entry points in one library.
- **The translator crashes**: the Ryujinx translator throws on some shaders (expression inlining, removed locals still used as output pointers). Fix it in the translator — don't route around it.

Alternative (the nx-port approach): skip the prebuilt shader table during bring-up; record shaders the game uses at runtime and translate afterward. The first run looks broken but reaches first frame fastest; return to the offline table for the real build.

## 1.6 Multi-game strategy: profiles vs flags

Where game-specific differences live when supporting multiple games:

- **Game profile (the nxnative approach)**: one directory per game with title ID, build, modules, RomFS markers, NVN constant tables, shader tables. Tools stop without a profile. Rewrite cut points and fixed-address behavior are all keyed by NSO build ID — never let a fixed address run against an unverified build.
- **Environment flags (the nx-port approach)**: all divergent behavior behind env flags, all on by default in bring-up scripts. Fast for experiments, messy long-term.

For frequently updated games like ACNH, the profile's build-union mechanism fits better.
