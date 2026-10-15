# 08 · Community projects: how others did it, what we borrowed

Four public projects, all in the "console games running natively on Apple platforms" space or adjacent engineering. The ACNH project's two routes both build on their experience. This file documents each project's approach and the borrowed pieces, so newcomers can follow the trail. All licenses are MIT or more permissive; keep the original copyright notices when citing.

## 8.1 d2r-mobile (jhaizhou-ops)

| Item | Content |
|---|---|
| What it is | Native compatibility layer running Diablo II: Resurrected (Switch) on iPhone/iPad/Mac/Apple TV; a 44-device closed beta cleared all three difficulties; 60 fps measured on iPhone 17 Pro Max |
| Route | No CPU emulation + offline rewriting of a few instructions + host substituting Horizon + NVN record/replay to Metal + fully offline shader translation (Ryujinx translator → SPIR-V → spirv-cross → MSL) |
| Content boundary | The repo contains no game code/art/text/shaders — everything is generated on the user's machine from their own game files; a tool byte-scans the whole tree before export and blocks on any hit |
| tvOS | Official community-ported DmTV target (separate Xcode target, shared host and graphics layer); reached the class-selection screen on Apple TV 4K 3rd gen hardware; the simulator can't run the graphics layer — hardware only |
| License | MIT |

Borrowed:

- **The entire native-execution route** is the ACNH project's template: no CPU emulation, rewrite a few instructions, host substitution, NVN record/replay, offline shader translation.
- **On-demand memory residency** (e2a0c9a): graphics pools enter the residency set only on first GPU use, saving ~0.5 GB of wired memory — the key optimization for 4 GB devices.
- **sRGB presentation fix** (#36): render targets are written through format-retyped views, and the sRGB swapchain is sampled for presentation through a non-sRGB view; otherwise the image is too dark.
- **Translation threading** (v1.0.15): command translation moved to a dedicated thread, overlapping game logic by one frame.
- **Toolchain versioning**: hash the chain's source + tool fingerprints; the host loads only matching-version libraries.
- **"Porting to other games" checklist**: the README enumerates host defaults hard-coded for a single game (render scale, save seed, fixed resolution, etc.) — these must be turned off or moved into per-game config before porting.
- **Content-compliance framework**: a manifest of "what's in the repo and why it must be" + pre-export byte scanning.

## 8.2 nxnative (seedds, formerly smbw-mobile)

| Item | Content |
|---|---|
| What it is | Multi-game Switch native-port toolchain: one script packages NSP + prod.keys into an unsigned IPA for sideloading; supports Super Mario Bros. Wonder (playable at 60 fps on iPhone) and Fire Emblem Engage (in bring-up) |
| Relation to d2r-mobile | Started as a copy of d2r-mobile's generic parts; fixes flow both ways |
| License | MIT (inherited from d2r-mobile) |

Borrowed:

- **Game profile architecture**: one directory per game (title ID, builds, modules, RomFS markers, NVN constant tables, shader tables); tools stop without a profile. Rewrite cut points and fixed-address behavior are all keyed by NSO build ID — never let a fixed address run against an unverified build. ACNH updates frequently; this mechanism fits exactly.
- **Survey first**: without a profile, the script auto-produces a survey report (modules/build IDs/SDK version/engine markers/containers/rewrite sites/import comparison) — measure before building.
- **RomFS per-directory name index**: the read-only mount builds a "guest name → physical name" index on first lookup. Caused by Engage's single 38,000-file directory, where per-entry getxattr for case mapping made the launch black-screen for 17 s; 1.8 s after indexing. ACNH RomFS has 100k+ files — same problem.
- **Present interval as a profile entry**: `game` (present at the game's own interval, same as Switch) vs `display` (one frame per display beat). ACNH locks 30 fps and its logic is frame-driven — the `game` profile.
- **DLC packaging**: DLC NSPs mounted by index; the host reports ownership by index.
- **docs/agents failure catalog**: five documents (extraction, survey, bring-up, graphics, debugging); the bring-up failure catalog (rewriter, memory, threads, services, NVN object sizes, GPU hangs, audio) is the most valuable failure index in the whole project — this skill's 03 owes it heavily.
- **Shader key improvements**: hash control section + code together (Mario has 6,367 pairs sharing a control section with different code); the container survey scans all RomFS files, not just the main shader package.
- **Prewarm manifest feedback**: record live-compiled shaders/pipelines while playing; prewarm in the background at next launch.

## 8.3 nx-port (Johnoooob)

| Item | Content |
|---|---|
| What it is | The second game-generalization fork of d2r-mobile: strips the single-game-specific parts to test how far the same native-execution route goes; one NVN game brought up |
| License | MIT |

Borrowed:

- **Runtime shader recording**: the alternative to the offline table — record shaders the game uses while running, translate afterward. The first run looks broken but reaches first frame fastest; return to the offline table for the real build. Worth considering in early bring-up.
- **.nro dynamic modules**: the host implements `nn::ro` (including RELR relocations and TLS descriptors) — a ready reference when the target game ships dynamic modules.
- **Divergent behavior behind flags**: the second game's graphics behavior all sits behind env flags, on by default — the opposite of nxnative's profile route; lighter during fast experimentation.
- **Geometry shader → vertex variant**: Metal has no geometry stage; a VS-variant approach for point-sprite-to-quad expansion.

## 8.4 bellpad (chrissotraidis and the ac-decomp community)

| Item | Content |
|---|---|
| What it is | A **source-level** native port of the GameCube original Animal Crossing: based on ac-decomp's decompilation, the game core compiles to native 64-bit code and runs on macOS and iPhone/iPad via Aurora (source-level SDK compatibility layer) + SDL3 + Dawn/WebGPU |
| License | Mixed (game code CC0, compatibility layer MIT) |

How it differs from this skill's route:

| Dimension | bellpad (decomp rebuild) | This skill (binary native execution) |
|---|---|---|
| Prerequisite | High-quality decompiled source | Game files + reverse-engineered interface facts |
| Runtime form | Game source compiled to a native 64-bit program | The original aarch64 binary runs directly, with a few instructions rewritten |
| Platform binding | Architecture-agnostic | Bound to aarch64 (the shared Switch/Apple ISA is the prerequisite) |
| Up-front cost | Extreme (decompiling a whole game) | Medium (interface survey + per-game bring-up) |

Borrowed:

- **Route comparison**: a living sample of two engineering routes for the same IP — the GC generation relies on decomp; the Switch generation can only rely on native execution (no public decompilation projects exist for Switch games). Useful when arguing for "why we chose this".
- **60 Hz frame-pacing contract**: game simulation is driven by a fixed retrace; even on ProMotion it must request 60 Hz — 120 Hz doubles game speed. "Simulation tick count never changes with present rate" is the same iron rule for 30 fps-locked ACNH.
- **Game data verification**: pinned Redump hashes + header/size/streaming fingerprints + atomic install + bad-save isolation — the strictest implementation of "user-supplied data".
- **Ship recipes, not apps**: releases publish build recipes; users build themselves, avoiding the risk of shipping an app containing game code.
- **Compliance stack**: clean-room conventions + all upstreams pinned + third-party notice lists — suited to projects that need to publish externally.

## 8.5 Quick reference

| | d2r-mobile | nxnative | nx-port | bellpad |
|---|---|---|---|---|
| Route | Binary native execution | Binary native execution | Binary native execution | Decomp source rebuild |
| Multi-game mechanism | Single game + porting checklist | Game profile + build ID keying | env flags | Single game |
| Shaders | Fully offline, control-section hash key | Fully offline, control-section+code hash key | Runtime recording + post-translation | Source-level GX→WebGPU |
| tvOS | Yes (DmTV community preview) | No | No | No |
| Relevance to the ACNH port | Direct (route + tvOS template) | Direct (profile/survey/failure catalog) | Indirect (alternative strategies) | Indirect (route comparison + compliance) |

## 8.6 Acknowledgments

The two ACNH-on-Apple-TV routes directly fork/borrow code and docs from d2r-mobile and nxnative (both MIT); the shader translator and audren implementation reference Ryujinx (MIT, pinned commit); the GC-generation comparison references bellpad and the ac-decomp community (CC0/MIT). Thanks to all the original authors for their open-source work. Keep each project's copyright notice and license when citing them.
