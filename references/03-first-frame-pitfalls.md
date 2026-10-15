# 03 · Pitfall catalog before first frame (symptom → diagnosis → fix)

Every entry was actually hit and fixed during the ACNH port. Sources are engineering reports from both routes (nx route P2a/P2b/P2c, smbw route Z1/T2/T3/OPT1 and others). Look up by symptom; don't read in order.

## Loading / rewriting / kernel

| Pitfall | Symptom | Diagnosis | Fix |
|---|---|---|---|
| BKTR | hactool fails on update NCAs with `Invalid BKTR layout!` | Known hactool issue with new BKTR layouts | Switch to LibHac 0.19.0, write an equivalent CLI |
| DT_RELR | SIGSEGV after loading at link-time addresses; relocation count is 0 | New SDKs pack relative relocations with DT_RELR; the loader only knows RELA | Add RELR application to the loader; ACNH has 391,693 slots |
| ctr_el0 | The sdk's cache-line query code SIGILLs on Apple | Apple doesn't expose cache topology to user space | Rewrite to a constant (128 B cache line) offline, before signing |
| W^X | Anonymous pages can't be made executable on tvOS | tvOS code-signing constraints | Put rewritten modules in a signed Guest.framework / embed in `__TEXT`; vm_remap the signed RX pages at runtime |
| Scene watchdog | Killed by the system ~20 s after launch (0x8BADF00D) | First-time unpacking blocks the main thread | Return from the main thread immediately; run unpacking and the guest host on background queues (survived 80 s of unpacking after the fix) |
| Small-file transfer | Pushing 100k RomFS files to the device: 5 minutes with no progress, disconnects leave junk | CoreDevice's per-file transactions | Stream an archive in ≤500 MiB chunks + a readiness marker + atomic staging publish on the device |
| Wrong heap size | SIGBUS at launch building a heap at host code addresses | `GetInfo(UsedMemorySize)` ignores loaded modules; the game takes total−used as its heap | Used = heap + image bytes + 64 MB |
| core=-2 | Crashes at frames 15–63; the bad pointer is byte-identical across runs | `svcCreateThread(core=-2)` unresolved; `-2` used as an array index | Resolve IdealCoreUseProcessValue(-2) to real core 0 |
| Rewriter false suspicion | Suspected the rewriter broke atomic sequences / didn't zero the heap | Exclusive-range audit: zero rewrite sites inside LDXR/STXR ranges; bad values are deterministic | Rule out the rewriter/heap/randomness; point at logic bugs like core=-2 |

## Services / IPC

| Pitfall | Symptom | Diagnosis | Fix |
|---|---|---|---|
| Preselected user | Launch stalls at user selection | `PopLaunchParameter(PreselectedUser)` returns NotAvailable | Provide a local preselected-user IStorage |
| Strict CMIF ABI | Real-device requests rejected by the framework's ABI checks | The game's actual payload/PID/OutPointer sizes don't match the stubs | Fix per the real encoding; ABI-audit every command (105 audited for ACNH) |
| NVN API version | Null pointer | NVN API major 53 doesn't satisfy the game's requirement | Return major 55 |
| SetShadersExt bool | Null address | `nvnProgramSetShadersExt` is a bool; the generic stub returns 0 | SetShaders/Ext return 1 |
| NVN object table exhaustion | Crash during startup | The fixed 2,048-entry object table is too small | Dynamic chained hashing + Finalize reclamation |
| Periodic events never signal | Static image after window creation | VI vsync and audren events never fire | Wire periodic events (16.67 ms / 5 ms) deadlines into WaitSynchronization |
| Deadlock | Main thread waits on an address forever | Non-periodic events (audren cmd 12, nfp cmd 23) never fire; the guest waits forever | Add event-wake coverage; when a handle set contains periodic events, degrade to timed waits (mitigation) |
| Direct service connection | The game hipc-connects to olsc:u itself and aborts on failure | Direct connection bypassing the SDK service library + domain messages | Force-takeover at the three hipc entry points; answer CMIF in place with a fake session (including the 16-byte domain header) |
| Clock snapshot | nn::time abort | ClockSnapshot goes through the time service | Fill the snapshot per the 0xD0-byte layout + timezone rules |
| Applet resource ownership | Abort inside OpenAudioRenderer | SDK 20 requires GetAppletResourceUserId ≠ 0 | Return a fixed non-zero value |
| Shared font | nn::sm::Initialize abort | RequestSharedFontLoad/GetSharedFont not taken over | No-op + fill back at 0x18 stride; or a host font snapshot |
| Six-axis / friends ABI | hid cmd 66 UNIMPL; friend:u cmd 0 rejected | Missing StartSixAxisSensor; the friends entry wrongly requires a PID | Add cmds 66/67; drop the PID requirement |
| svcCloseHandle after HIPC Close | `svcCloseHandle 0xe401` | HIPC Close already deleted the session | One-time closed-session accounting |
| Offline service semantics | Online features hang | 35 online services need "no network" semantics | bsd Connect returns ENETUNREACH, nifm returns NoInternetConnection, ssl returns NoConnection, etc.; unknown commands respond, never stay silent |
| Abort after "Save and end" | `nn::diag` abort (Result 0) right after the save finishes | `CalculateStandardUserSystemClockDifferenceByUser` not taken over; the SDK aborts on a null time service object | HLE the function (semantics per Ryujinx `IStaticServiceForPsc` cmd 500: compare two snapshots for the user clock offset). Round the nanosecond offset to debounce — otherwise two snapshots differ by ±1 s out of nowhere and read as a clock change; use the host's boot time as the steady clock source |

## NVN / graphics

| Pitfall | Symptom | Diagnosis | Fix |
|---|---|---|---|
| CallCommands not replayed | First present has only 3 passes / 0 draws | `nvnCommandBufferCallCommands/CopyCommands` not recorded | Record as OP_CALL for recursive replay; grow command memory in 4 KiB host segments and reject out-of-range segments |
| Runtime shader entry name | Shader link error | The packager takes the SPIR-V reflection entry `main` as the metallib entry | Take the real entry from the MSL vertex/fragment/kernel declarations |
| Shaders outside the manifest | 2 shaders missing | Runtime capture beyond the static manifest | Runtime capture → translation → runtime package |
| DrawElementsBaseVertex | Only 1 draw per frame | `nvnCommandBufferDrawElementsBaseVertex` missing | Add indexed/base-vertex recording and replay |
| Vertex attribute numbers | All shader resources wrong | Vertex attributes used SPIR-V ids as NVN locations; cbuf/C1/driver bank, texture, and storage slots don't match the NVN ABI | Rebuild reflection per the NVN ABI; fill c1/driver0/driver2/ubo_ptr at runtime |
| TextureView 16 B | inf/nan in the synthetic matrix | The game's NVNtextureView is only 16 B; the host's 36 B defaults overwrite adjacent viewport rectangles | Shrink view_t to a strict 16 B + static_assert the ABI |
| NEAR_IS_MINUS_W | The synthetic quad is clipped by Metal | NVN's default depth convention accepts z∈[-w,w]; Metal only accepts [0,w] | When packaging all VS, do z=(z+w)*0.5 before returning (3,117 VS repackaged for ACNH) |
| ReportCounter {0,t} | UI is drawn but the final frame is still black; a conditional clear passes everything and wipes same-frame UI | TIMESTAMP wrongly written {t,t}; Maxwell ZERO reports should write {0,t} | Write back {counter=0, timestamp=t} in the post-op; don't invert the condition or skip the clear |
| Conditional clears | The counter background goes fully black for a few frames at a time; with TBDR on, the whole 3D background is black | Counter reset to 0 happens in the GPU completion callback, so the next frame's encode still reads 1 and the clear runs twice; write-back speculation treats clears under conditional rendering as "will be overwritten later" | Write counter fields to 0 in order at encode time; the completion callback writes only timestamps. Don't index clears under conditional rendering for write-back speculation |
| Snapshot race | In-frame snapshots occasionally read stale content | The snapshot runs ahead on the legacy queue, before the replay command buffer is submitted | Keep the snapshot copy in the same replay command buffer |
| Font CPU-write linear textures (ruled out) | Dialogue Chinese text and nameplate glyphs don't show | Linear texture / CPU-write visibility was suspected; five GPU readback paths ruled it out one by one — linear textures are fine | The real cause is the next entry (typed view). Don't spend time here |
| texture2d_array vs 2D | Text missing on TV, fine on Mac; 3D scenes 15–30% darker | The text fragment shader declares the glyph cache as `texture2d_array` and samples layer 0; the host bound a 2D view. Type mismatch is undefined behavior in Metal — M-series happens to read it right, A15 reads 0, alpha≤0 discards everything | Create a second view on the same storage matching the shader's declared type (2D↔2D array, cube↔cube array, cube→2D) |
| Alpha test | Tree canopies / grass cutouts render as black blocks | NVN ColorState fixed-function alpha test (GEQUAL 0.16) has no Metal equivalent | Inject discard into fragment shaders offline (per function constant); the replay folds the alpha function and reference into the pipeline key |
| Short control-section shaders | 47 "shader not in map" errors | agl/gsys package control sections are only 0x848/0x854 bytes, and a load-time-rewritten pointer follows the section; hashing a fixed length gives a different key every time | When a control section self-reports a length under 0x880, hash only that (table 6,536→9,742) |
| Compute shaders without a header | Compute shaders can't be split out | ACNH's BNSH compute code starts at +0 with no 0x30 header | The splitter allows headerless compute stages |
| Ryujinx translator crashes | 18 CS throw InvalidCastException, 1 PS throws KeyNotFound | AstOptimizer inlines expressions back into Call arguments; removed locals still used as call output pointers | CallArgumentPrepass: materialize non-local expressions into temporary locals first; re-add pointer parameters to Locals |
| GS won't compile | 4 GS produce MSL with `unknown` types | Metal has no geometry stage | Three-entry compute prepass fallback (see 01) |
| texpack won't mmap | The 7.3 GiB texture pack can't be mmaped whole | Without EVA, tvOS process address space totals ~7 GB | Read-only index (header + entry table ≈ 3.9 MB) + on-demand pread for data |
| Texture view format retype | The easiest conversion to miss | The game calls nvnTextureViewSetFormat thousands of times; BC formats retyped in views must be converted too | Build views on the underlying ASTC/EAC format; GetStorageSize keeps returning the original BC bytes |
| BC5_SNORM can't go ASTC | Normal maps sample [-1,1]→[0,1] after ASTC; n.z is all wrong | ASTC LDR has no signed mode; fixing sampling means changing shaders | Use EAC_RG11Snorm (same block bytes, exact semantics, supported on all Apple GPUs, no shader changes) |
| Mid-render-pass depth/stencil clears | Red line "mid-pass depth/stencil clear unimplemented"; later draws test against stale depth | The game issues depth/stencil clears in the middle of a pass | End the current pass and attach the clear to the next pass (color attachments still load); keep the red line when the clear scissor doesn't cover the depth target |
| Premature pool residency | Unused pools still hold wired memory; 4 GB devices killed at launch | Pools enter the residency set at creation | Make resident only on first GPU address fetch, and commit the set then (~0.5 GB saved upstream) |
| Dirty-page tracker race | Game threads spin forever in the page-fault handler | Marking read-only sets PROT before mprotect; concurrent "unprotect before write" scrambles the state | State machine NONE→BUSY→mprotect→PROT; waiters recheck after BUSY clears; report red after a 2 s cap |
| Display link never fires | Frozen on a black frame; output stops at the 3rd present | CAMetalDisplayLink callbacks never arrive on Apple TV | The output thread self-fetches drawables (env switch to revert) |
| Depth-EQUAL prepass invariance | Diagonal black moiré on near canopies/objects (looks like z-fighting); ACNH's old library had 10.92% pure-black pixels in the same scene | The game draws a depth-only prepass, then draws color with a *different* vertex shader at depth EQUAL; MSL marked `[[position, invariant]]` but the metal compile didn't pass `-fpreserve-invariance`, so Metal ignores invariant — the two VS positions differ in the last bit, and failed-EQUAL pixels keep the clear color | Add `-fpreserve-invariance` to metal compiles on Mac/iOS/tvOS; rebuild the whole library and bump the toolchain version. Same scene after the fix: 0.00% pure black |

## Audio

| Pitfall | Symptom | Diagnosis | Fix |
|---|---|---|---|
| REV15 wire format | svcBreak (0xc99) before first frame after merging main | Real REV15 voice sections are 0x188 and SNDD 0xA8 — 0x18 larger than the old revision | Select layout by update/behavior revision; derive splitter stride from the "SNDD" magic; during transition, fall back to a logged silent compatibility packet |
| Tests share production's wrong assumption | The REV15 test suite passes self-consistently but the wire format is wrong | Fixture and production share the assumption "voice is always 0x170"; fuzzing can't mutate a legal 0x188 packet | Physically separate test layout tables from production constants; byte-level assertions on size/offset/stride; state fields tested across at least two updates |
| DeviceSink command number | No audio on Apple TV; PCM all zero | SDK 20 changed the DeviceSink command number from 0x13 to 0x14; the host reads a different command, fails field validation, drops the whole frame | Make the command number a game-profile entry (default 0x13 keeps old games working) |
| Render thread race | Minutes of island walking, then a guaranteed crash in the audio render thread | Under SDK 20 the host's chase-render calls the public ExecuteAudioRendererRendering without the host interception layer — no lock, no render counting, concurrent writes to the same command buffer as the render thread | Render thread holds the lock, counts renders, Update is mutually exclusive; 2,632 s crash-free after the fix |

## Debugging drill

The standard order for a graphics black screen (from nxnative's docs/agents and ACNH practice):

1. **TEXDUMP**: is the texture data there? If not → check texture upload / the replacement pack / view formats.
2. **PASSDUMP**: were draw commands issued? If not → check recording/replay, CallCommands, DrawElementsBaseVertex.
3. **Match view types to shader declarations**: issued but nothing draws → check typed views (2D vs 2D array), NEAR_IS_MINUS_W, alpha test.
4. **Read VS output**: geometry wrong → check vertex attribute numbers, depth conventions.
5. **Conditional rendering/clears**: intermittent → check the write-back timing of ReportCounter and conditional clears.

Keep evidence for every ruled-out item (readback byte counts, pixel statistics) — don't rule things out by feel. ACNH's text bug wasted a round on "linear textures" when the real cause was typed views.
