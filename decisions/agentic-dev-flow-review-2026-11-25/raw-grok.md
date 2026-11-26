# Review

Load: 10 projects, 50 MRs/day on the busiest, gates 30 min if split, 50–80 min as today. tacplus is one 8-vCPU spot VM; the other nine share one 4-vCPU spot VM.

## Bottlenecks

**1. Rebase-to-tip does not fit on one VM.** An MR opens only if gates were green on current main. One build at a time, attempts per land = e^(rT); the box saturates when rT·e^(rT) = 1 (rT ≈ 0.57). Ceiling ≈ 54 MRs/day at 15 min, ≈ 27 at 30, ≈ 10 at 80. Fifty is past every gate this repo runs (`pkg/**` is still everything). A second build or a preemption puts even 15 minutes under 50. "Nothing is queued" is the VM.

**2. The check is already full; a red makes it a queue.** Back-to-back: 48/day at 30 min, 24 at 60, 18 at 80. Fifty merges exceeds 48. Ten clean merges land in seconds during one check, so the next red is ~10 commits, not a handful. Isolating one of 10 is 4 serial full gates: 2–5 hours, plus the red run. That is the only slot, so merges during the bisect go unchecked. Main stays red for hours.

**3. One conflict blocks the only merger for a gate.** A conflict holds every later MR for 30–80 min and rebuilds on the VM in finding 1. One in five conflicting (manifests, `gates.yaml`, generated code) is ≈ 7.5 hours/day blocked.

**4. Nightly publish is a 24-hour queue on the check's slot.** Dev moves on the nightly unless someone fires build-now; if that is Jody, it is a person-queue this page denies. On the check's runners that build takes the slot finding 2 lacks. Production's version bump is a person-queue by design.

**5. Scale-to-zero, spot, and the shared VM.** The kicker wakes a VM for a ready issue, not an open MR, so idle MRs land later as a stale burst into finding 2. Preemption drops every in-flight gate; the checkpoint saves the branch, not the half hour. Nine repos share one 4-vCPU parent and no fair-share, so one full-gate report stalls the other eight.

## Correctness

**6. Undo-within-one-build does not hold.** Finding 2 is several builds, and only if one commit is solely and monotonically guilty. Otherwise the bad merge stays, or a good one is removed.

**7. The MR body is the evidence.** The container socket and model-written code can write it. The merger lands in seconds, unbuilt.

**8. Revert, then the same tree returns.** The issue is marked ready, the branch kept, the merger not frozen. A flake does this with no bug. Reports then build on that red HEAD.

**9. A bad bisect has no owner.** No rerun before revert, so a flake or a pair removes an innocent merge and leaves the culprit; the next check removes another. A 2% full-run flake at ~48 checks/day is a false red most days; 2% in each of 19 groups is ~32% if independent (1 − 0.98^19). A conflicting revert is a silent edit or a stop, and a stop names nobody. That blank failure-table cell is the common case.

**10. Green, and still wrong.** The merger can drop one side, pass every gate, and ship. No owner. Rollback covers a failed deploy, not a successful one; dev has no canary. Seatrials' apply is still open (fleet-seatrials#76). A release job that resolves `main` at start, not the green SHA, publishes commits the check never built.

**11. No fuse, and consumers are outside it.** Check down, revert agent wedged, or runners refusing jobs (this already happened): the merger keeps merging. A service on the latest patch of a pinned tacplus minor is never built here.

## Open questions

**12. Q1. Disagree that (c), with (b) only for conflicts, is enough.** GitHub must write main. The project VM must hold neither that credential nor the one that sets "accepted," or the report accepts itself. Merger on the small VM that runs no project code; protected status; GitHub auto-merge. No merge queue, no required build.

**13. Q2. No compile-on-merge.** It costs a gate, misses the break that matters, and becomes finding 3. One land per check that has started on that SHA. Pause while the check is red, down, or bisecting.

**14. Q3. An on-demand VM per heavy project, plus one for the shared org.** Not the project VM, not spot, not the release runners. Size it so 19 groups finish in the slowest group's time. Eight vCPU will not.

**15. Q4. Do not leave this open.** Rerun the red head once; green means flake, revert nothing. Revert only when one merge isolates, the revert applies clean, and its scoped gate is green. Otherwise freeze the merger and open one issue, "main red, fix forward," on a looper. No revert through dependents, no merge onto red main. Jody only if the freeze exceeds one full gate. Deploys stay on the last green release.

**16. Q5. A log is not evidence.** Signature over tree hash, base SHA, gate-set id, and result, from a key the report cannot mint, checked before accepted. Always rebuild conflict resolutions on the check VM.

**17. Q6. Fail closed.** Unmapped paths run full gates. A wrong map is not caught within one build while the check lags, and never if no test covers the edge. Diff `gates.yaml` against the package graph; file an issue on a missing edge.

**18. Q7. Do not block tacplus main on service builds.** Block the deploy: a release SHA triggers consumer checks on that digest; red does not go to seatrials. Split a two-repo change, or hold deploys across it.

**19. Q8. Parallel reviewers, a 15-minute cap, then proceed.** Two vendors rejecting on security or "does not do the issue" blocks; one does not. Off the project VM. Re-review only a conflict diff.

**20. Q9.** Findings 1–5 and 8–11. The empty owner is the row that will fire.

## Missing

**21. No single-flight per issue.** Preemption resumes a report, and the kicker can start another. Two MRs; the merger takes both.

**22. The security-gates exemption died here.** Nightly CI-only image re-pins were non-blocking. This check treats every red as a bad merge, so the revert agent reverts the pin or freezes every repo, every night.

**23. Jody is not told, and triage is an unnamed queue.** No metric for lag, freeze, or saturation. A person on triage is a queue not on the list he kept; an agent is an unreviewed approve.

## Do these first

**24. Freeze, and publish the exact SHA, before any unbuilt merge.** One land per check that has started on that SHA. Pause when the check or revert agent is red, down, or bisecting. Release jobs publish that SHA only.

**25. Split the gates, fail closed, measure, then allow a clean merge.** `pkg/**` must not mean everything. Until a scoped gate is under 15 minutes on this VM, measured, finding 1 is slower than the train.

**26. Quarantine, and one owner when the bisect fails.** A reverted tree hash cannot merge until the issue has a new commit. Anything short of one clean revert freezes merges and opens one fix-forward issue. Otherwise finding 8 spends what finding 24 saved.

