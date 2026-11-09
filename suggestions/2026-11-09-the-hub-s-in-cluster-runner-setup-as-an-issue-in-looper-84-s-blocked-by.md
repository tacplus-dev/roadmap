Suggestion: The hub's in-cluster runner setup as an issue, in looper#84's Blocked-by
Type: Operability
Summary: cluster#6 closed on 2026-11-04, but cluster's STATE.md says the in-cluster build and control runners are not registered or applied, and no cluster pipeline has run (no 'cluster' runner). Tonight all 8 of tacplus-dev/build's jobs failed with no matching runner. File the hub's steps (cluster README, What the hub does) as an issue and put it in looper#84's Blocked-by in place of the closed cluster#6
Why: looper#84 is the decided fix for the night's largest item: 126 issues waited 388.53 h for a VM, the fourth night. #84's Blocked-by still names cluster#6, which is closed, so #84 looks closer than it is. What actually stands in the way is the hub's registration of the runners, which no open issue tracks. cloud#26 closed on 2026-11-08 and started the build schedules, and each scheduled build now fails within the hour with stuck_pending_no_matching_runners. So the security floor (#49) still runs nowhere, two days after Jody accepted pinning every repository for it. The recap can't see cluster's pipeline either: its source gitlab:tacplus-dev/cluster returns 404. Also serves Security (the nightly floor) and Performance (the way off the VM queue).

## Evidence
- https://gitlab.com/tacplus-dev/build/-/jobs/17846235326 (recap 2026-11-08, failures/tacplus-dev/build/other:stuck_pending_no_matching_runners: 8 of 8 jobs, plan and pending, failed at 23:16Z with 0 recovered; also https://gitlab.com/tacplus-dev/build/-/jobs/17846206212, https://gitlab.com/tacplus-dev/build/-/jobs/17846204623, https://gitlab.com/tacplus-dev/build/-/jobs/17846165556)
- https://github.com/tacplus-dev/cluster/blob/main/STATE.md (Known gaps: 'the in-cluster runners are not registered or applied yet: the hub registers them as tacplus-dev group runners'; 'No pipeline has run: the hub owes the runner (tag cluster), the project's variables and environment')
- https://github.com/tacplus-dev/cluster/issues/6 (closed 2026-11-04) and https://github.com/tacplus-dev/looper/issues/84 (Blocked-by tacplus-dev/cluster#6 and tacplus-dev/tacplus#315, plus the pin list)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17834691878 (recap 2026-11-08, capacity/vm: 126 issues waited 388.53 h for a VM, queue p90 415.2 min over 294 jobs, 4 of 7 nights; sources: gitlab:tacplus-dev/cluster 'returned error: 404')

## Cost
An hour or two of the hub's time with Jody, since group runner registration needs an Owner. It follows cluster README's What the hub does, steps 1 to 7: the project runner tagged cluster, the protected environment and variables, the build and control runners registered as tacplus-dev group runners with their tokens in Secret Manager, and tacplus-looper-bot as a Reporter on tacplus-dev/cluster for the recap. No money: the runners use the existing dev cluster's normal pool and its spot pool, which scales from 0.

## Touches
decisions/only-vms-are-k8s-nodes-2026-11-04.md (CI in the cluster; its 2026-11-07 addendum: loopers all at once, builds async on spot), suggestions/2026-11-06-pin-every-repository-so-the-build-pipeline-and-its-security-checks-run-anywhere.md (accepted), looper#49 (the security floor), looper#84, looper#85 (the recap's sources), cloud#26

## Reuse considered
Everything is already built: cluster#6 rendered both runners, #13 gave the control runner its own tags, and cloud#26 started the schedules. Only the hub's registration and apply are left. A comment on #84 would not show up as a blocker, and an issue would, so the tracker's Blocked-by stops hiding the real gate. Not proposed: any change to the VM pools (Jody, 2026-11-06: they get no fix).

## Advice
Recap: 2026-11-08 failures/tacplus-dev/build/other:stuck_pending_no_matching_runners
Cause: tacplus-dev/build's jobs need the in-cluster build runner, which is not registered (cluster STATE.md), and cluster's own pipeline has no 'cluster' runner to apply it; no open issue tracks these hub steps since cluster#6 closed
Change: a tacplus-dev/cluster issue for the hub (README What the hub does 1–7, plus tacplus-looper-bot Reporter on tacplus-dev/cluster), linked as looper#84's Blocked-by in place of cluster#6
Gain: 8 failed build jobs a night to 0, and the nightly security floor running for every pinned repository. It also clears the last infrastructure step before #84, which removes the 388.53 h a night of VM waits (those waits remain #84's to remove)

## Status
accepted (Jody, 2026-11-09): tacplus-dev/cluster#18, now in looper#84's Blocked-by in place of cluster#6
Rank: 2
Filed: 2026-11-09 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17842078231)
