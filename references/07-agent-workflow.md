# 07 · Running the project with multiple agents

This kind of port spans a wide surface (kernel, services, graphics, audio, memory, real hardware); one agent doing it serially takes months. The ACNH project ran as a coordinator + task agents + review agents. These notes are for teams wanting to run the same way.

## 7.1 Spec-driven

Every task gets one spec document with four fixed sections:

1. **One-sentence goal**;
2. **Prerequisites**: which completed tasks it depends on, which reports to read first;
3. **Numbered steps**: what to do, which directories may be touched;
4. **Deliverables**: report path and format, final-line VERDICT (`DONE` / `PARTIAL — what's missing`; reviews use `PASS` / `FAIL — n items`).

Supporting rules:

- Commit after every step, or at least every 20 minutes — if a task is killed, the next one resumes from the commits;
- Reports lead with the conclusion, then tables/commands/numbers — no filler;
- Dependencies come in two layers: the spec's "prerequisites" are soft (read reports); dispatch parameters are hard (whose worktree to base on, whose work to review);
- Channels break: an execution agent's channel may be shut down mid-run (it happened on this project — an administrator killed the agent provider's channel), and tasks can disconnect. All working state lives in git commits and findings reports; after a disconnect, a new task resumes from the worktree (DAY2 is exactly this — picking up a previous task's disconnected state). This is the other reason for the 20-minute commit rule.

Tasks split by layer: base, IPC framework, services, shaders, audio, loading, integration; same-layer tasks can dispatch in parallel. Fix rounds open new tasks (never reopen old ones); after a fix, dispatch a fresh review round, closing the FAIL → fix → PASS loop.

## 7.2 Cross-family review: why it works

Reviews are independent tasks; the spec says "read the report + spec, rerun the tests yourself, don't fix code". Key design choices:

| Design | Why |
|---|---|
| Review agent from a different model family than the builder | The builder's wrong assumptions get written into both implementation and tests; same-family reviews tend to inherit them |
| Reviewers write their own harness/encoders, never linking the repo's test builder | Prevents production and tests sharing a wrong assumption that produces false PASSes |
| Protocols derived from public docs (Switchbrew/Ryujinx/libnx pinned revisions, with full commits and URLs) | Don't use the implementation as its own spec |
| Mutation self-check: deliberately break one piece of production code, confirm the test goes red, then revert | Proves the test can catch defects |
| Harness teeth: run the new harness against the pre-fix commit, confirm everything FAILs | Proves the harness itself works |
| Hygiene gate: diff scope, license notices, no game files/keys/large blobs, no banned trailers | Mechanically checkable |

Real problems caught (all "self-tests all green but the implementation is wrong"):

| Review | Verdict | Problem |
|---|---|---|
| 35 offline services | FAIL 7 | Tests built requests per the implementation's wrong protocol: friend service wrongly required a PID, BCAT returned the wrong type, NIFM missed a C buffer (real requests were rejected at the framework layer, never reaching handlers) |
| REV15 audio | FAIL 3 | Test packets were "REV15 magic + REV14 structure"; the real REV15 wire format (voice 0x188, SNDD 0xA8) was unimplemented, legal packets rejected |
| Ocean GS fallback | FAIL 1 | "Input primitive = triangles" was circular evidence: triangles was the test script's default, written into the sidecar and read back as observation |
| smbw route Z1 | FAIL 4 | Stale translation-toolchain header; shared-path fast path breaking case-exact semantics; commits carried banned trailers; game screenshots still in repo history |
| RAUD1 (AUD1 review) | FAIL 2 | Logging in real-time callbacks (stdio lock, violates AudioUnit real-time constraints); RMS numbers not reproducible from a repo script — the reported full-run value didn't match the retained PCM |
| RUP1b (UP1b review) | FAIL 3 | Copied the formula but not the wiring: the pacing pure function was right, but production read the main screen instead of the window's screen and never re-set it at runtime; copied half a pair: called `becomeFirstResponder` without `canBecomeFirstResponder`; A/B report not replayable (placeholder commands, unclear cold-start criteria) |

## 7.3 The review-of cwd trap

What happened: two review tasks were enqueued before their builders had started; the system resolved the default working directory at enqueue time and fell back to the main repo — the review agents reviewed nothing in the wrong directory, yet their results were marked complete.

Lessons:

- Dispatch reviews explicitly with `--cwd <builder's worktree>`, and only after the builder has started;
- Verify the cwd with your task system's status command after dispatching;
- Monitoring should treat "results written but process still alive" as still running.

## 7.4 Two-route race

The ACNH project ran two routes in parallel: a clean-room base (nx, written from scratch) and one built on a community framework (smbw, multi-game profiles). Rules:

- Fixes made on one side (NVN object sizes, preselected user, core=-2, unimplemented NVN calls, etc.) go to the other as patches or hints;
- On a rendering problem, first check whether the other route already has a solution;
- Switch primary/backup by evidence: smbw reached the island on Mac first and became primary; nx stayed as backup feeding fixes back;
- Track upstream community projects regularly (a 6-hour routine task) but **never merge wholesale** — cherry-pick the useful parts, keep your own structure.

Two routes cost twice the maintenance. The payoff is mutual insurance and a reference point: when one route is stuck, the other's progress proves "the road itself is open", preventing death-marches in the wrong direction.

## 7.5 Serialization rules for the Mac and the device

- With one MacBook and one Apple TV, build/device tasks carry an exclusive lease; only one runs at a time;
- Device operations (install, launch, file retrieval) are done only by the main agent, serially; read-only research, code reading, and review cross-checks can go to subagents (≤4, run synchronously in the foreground);
- Game content never leaves the dev machine: key contents are never printed, logged, reported, or committed — passed to unpacking tools by path only; game files are unpacked and used only on the dev machine; what comes back to the working machine is statistics, symbol names, and API names from logs;
- Game screenshots don't go in the repo (evidence images live in an ignored directory; reports cite paths and pixel statistics only);
- A "passes on device" claim requires the command, exit code, and retrieved log/screenshot paths; simulator passes and device passes are written separately.

## 7.6 Borrowing from upstream: hand-port item by item, reviews as the backstop

UP1b is the full sample of "track upstream without merging wholesale" (upstream = nxnative and d2r-mobile):

- **No merge — hand-port item by item.** Each item notes its source commit and license (MIT), and lands with unit tests and A/B. One round borrowed five: RomFS per-directory name index (replacing two old cache layers, and incidentally killing the "APFS case-insensitivity treats wrong case as a hit" hole at the root), translation-toolchain header fixes for two profiles, DmTV's controller hand-back to UIKit and display-beat integer-division fix, and one failure-catalog entry.
- **Deciding "doesn't need porting" on evidence is also a result.** The 18 `ec::ShowShop*` overloads upstream added for Engage are useless to ACNH — of 1,016 imports, the 5 `Show…` names are all already taken over, and swkbd imports are 0. No code ported; only the failure-catalog entry for "how to find missing Show… overloads" was borrowed.
- **Reviews specialize in catching "half-ported".** All three RUP1b failures were this class: the pacing formula ported but its production wiring (read the window's screen, re-set at runtime) didn't; `becomeFirstResponder` ported but its paired `canBecomeFirstResponder` didn't; A/B readings existed but the replayable command and cold-start criteria weren't written. Green pure-function unit tests can't prove the wiring — porting a function means porting its call sites, runtime re-setup, and paired protocols together.

## 7.7 Reusable checklist

1. Fix the spec template at four sections; machine-readable VERDICT on the final line so the coordinator can schedule by reading the last line only.
2. Reviews are independent tasks, cross-family, self-written harnesses, protocols derived from public docs.
3. Fix rounds open new tasks; re-review after the fix; loop until PASS.
4. Run two routes to the same goal in a race; exchange fixes regularly; switch primary by evidence.
5. Track upstream, never merge wholesale; cherry-pick only.
6. Single hardware is serialized by leases; device operations are main-agent-only.
7. Write the hygiene gate into review specs: no game files/keys/screenshots in the repo or its history, license notices preserved, blob size cap.
8. Evidence must be reproducible: command + exit code + retrieved paths; pixel assertions are semantic; audio is measured by RMS.
9. Clean history before publishing: spoofed author identities, banned trailers, game files that entered history — deleting from the working tree isn't enough; rewrite with filter-repo and push only the cleaned branch (see 7.8).

## 7.8 Pre-publish history cleanup: clean working tree ≠ clean history

Lessons from the ACNH project's pre-publish review (RZ1) and history cleanup (HIST1):

- **Upstream AGENTS.md can make agents commit under the upstream author's identity.** The upstream repo's AGENTS.md said "commit as the upstream author"; an agent complied and 15 commits landed with the wrong author. When forking, first check the repo's AGENTS.md for such clauses and fix them before starting; fix what's already wrong with mailmap + filter-repo.
- **`Co-authored-by` trailers**: one agent provider's channel auto-adds `Co-authored-by: <provider>`; the ACNH repo had 17 commits with trailers (project notes said 12; the real count was 17 — trust `git log --grep` over memory). A message callback strips them line by line.
- **Game screenshots in history**: 31 evidence screenshots were once committed under findings/, then untracked and .gitignored — but the blobs are still in history, reachable from `git log --all`, and pushing any old branch carries them out. Deleting from the working tree is not history cleanup.
- **How**: in a separate mirror clone (`--mirror --no-hardlinks`), use git-filter-repo to rewrite only the target range (`--partial --refs base..branch`), `--prune-empty never` to keep topology one-to-one. Acceptance: trees byte-identical before/after, single identity in the range, zero trailers, zero screenshot paths, upstream SHAs at and before the base untouched.
- **Push only cleaned branches**: other local branches still carry the pollution; check the reachable history of whichever branch you push. A cleaned branch is a snapshot in time — after the original branch moves forward, rerun the cleanup at the new tip before pushing.
