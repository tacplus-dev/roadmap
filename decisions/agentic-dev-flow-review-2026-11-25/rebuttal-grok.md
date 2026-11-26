# Rebuttal

## Adopt

**Codex 1, 4, 6.** 50×4 cores×30–80 min = 100–267 core-hours vs 192; nine repos share four. Nineteen groups need cores the looper VM lacks. Kicker adds P/2; spot, model, and forge limits are unbudgeted. Admit, isolate check capacity, publish queue age.

**Codex 2; Gemini F2 (the loop only).** Park conflicts; keep merging clean MRs.

**Codex 3, 5, 7, 9 (window), 11, 16, 17.** Inclusion plus verdict is 60–160 min before bisect; diagnosis can hit 7×80 min. Coalesce dev releases; a button is a person. Revert does not undo a migration. Bisect may not name one commit. Idempotent claims on merge, revert, deploy. Own green SHA per release branch; no older digest deployed last; pipeline files out of candidate code. One in-flight check of unverified clean merges; pause if red, down, bisecting, or older than one gate.

**Codex 12, 14; 15's hours.** A signature proves origin, not gates. Release authority is an independent rebuild. Pin the tested digest. 3×10 min×50 = 25 reviewer-hours/day.

**Gemini F4, F5, F6, F7, F9–F15.** 46 of 82 *failed* child pipelines were GitLab refusals under a ~500-job cap, not 56% of runs. Preemption restarts every build at once. Two clean MRs can fail to compile together; a clean `git revert` of a helper breaks a later caller. Bounce conflicts: an LLM can drop a side and still pass file-scoped gates. Forged MR text, shared cache. `release/X.Y` cherry-picks are unchecked. A red release needs an owner; a running VM is not rescanned. Rebase on last-green. Cap builds per issue. 25s will not checkpoint ten worktrees on one git store.

## Disagree

**F1's 87%.** That is not 30 min on 8 cores (~200 core-hours). Use Codex 1. Keep the semaphore.

**F2's O(N²) and 30%.** Not measured. Bouncing conflicts moves the cost onto that VM.

**F3.** With last-green, reports do not stall. The merger freeze does.

**Q2.** A cross-crate typecheck is not under 2 min, it serializes a full VM, and it misses every break that still compiles. No merge-path compile.

**Q6's auto-edit.** A red check must not rewrite `gates.yaml`. Fail closed; file an issue.

**Q7.** A pre-release per MR publishes commits the check has not kept. Build-now a SHA already green.

**Q8, and Codex 15 if a timeout holds the MR.** 90s and 2-of-3 lets one fast model decide, or the timeout waits. A wait is the train. Two completed security or scope rejects block; one does not; silence does not.

**Codex 8 and 10.** No LLM resolution. Reserved runners cannot be the GitLab project that already refused jobs.

**Both Q4s on every red.** The 2026-11-23 decision: CI-only image and tool findings do not fail a land, and a nightly job re-pins those images. Reverting every red reverts that pin, or freezes every repo, nightly.

## Missed by all three

**Option (c) cannot revert.** If only GitHub writes main, the revert has no writer unless the controller pauses auto-merge and fast-forwards one revert, or one fix-forward MR if that revert will not build. That exemption cannot be a label the report token can set, or the repair deadlocks behind the freeze.

**The merge queue is the train.** It rebases, waits, and will not park conflicts. Non-rebasing auto-merge, or a controller fast-forward. The author's "or" is two designs.

**The cache disk outlives the VM.** Cloud-init remounts a readable disk. Preemption clears neither a full disk nor a poisoned cache. No size, eviction, or owner.

**Metadata.** Build containers can mint the VM service-account token from the metadata server. That account already reads Secret Manager, including the model token. A signing daemon on the same VM is not a boundary unless those containers cannot reach metadata.

## Open questions

1. **Q1.** Control VM, no project code, no shared metadata; non-rebasing auto-merge writes main; conflicts bounce; only a revert and one fix-forward may land during a freeze.
2. **Q2.** No merge-path build. One in-flight check of unverified clean merges; pause if the check is red, down, bisecting, or stale.
3. **Q3.** Non-spot VM per heavy project, plus one for the org, own cache, more than 8 vCPU; not the looper VM, not GitLab.
4. **Q4.** Rerun the failed group once; ignore CI-only pin reds; revert only if one shipped-code commit isolates and the reverted tree builds green; else one repair looper; no retry of the original; loopers stay on last-green; Jody only after one full gate.
5. **Q5.** Trust only a status the report cannot mint or fetch via metadata. Release authority is an independent rebuild bound to the published digest.
6. **Q6.** Generate the map from the package graph; unmapped paths and policy edits run full gates; never auto-patch the map.
7. **Q7.** Do not block tacplus main and do not pre-release per MR. Pin the tested digest; a red consumer check does not reach seatrials.
8. **Q8.** Parallel, off the looper VM, capped. Two completed security or scope rejects block. Timeout and a single reject do not. Not mandatory until the reviewer quota exists.
9. **Q9.** Admission queues, a rescan while the VM is up, a per-issue build cap, an owner for a red release, a cache-disk generation.

## Bottleneck

One check lane, still the slowest full gate: 30 min if split and staffed, 50–80 if not. A shipped-code red freezes landing for inclusion, the red build, and the bisect — 2–5 hours — while 12–30 MRs arrive on an 8-vCPU VM already past 192 core-hours once retries count. Last-green lets loopers compile. It does not merge. Nothing adopted adds a second lane or shortens `current-generation`. That freeze is the stall this page was written to remove.

