Suggestion: The manager closes a land whose change is already on main, without running its gates
Type: Performance
Summary: A land the hub hand-landed (throughput first, point 3) stays queued with its manager, which runs the full gates on it again. tacplus#351 and #554 each spent a 3-hour full gate run per pass, on a land that, once rebased, changes nothing. The manager's 'nothing to land' check counts commits before the rebase, and a cherry-pick has a new SHA. After the rebase, choose_gates maps 'no changed path' to full. Check after the rebase, and close such a land as already landed
Why: Under throughput first the hub lands the merging fixes by hand (tacplus#266 tonight, looper's 15 commits at 07:00-07:57Z), and each one leaves its report's land queued. The hub then has to drop them one by one: '#351: each pass spent a full 3-hour gate run on nothing', the same for #554, and 'Looper: drop the queued land of looper/tacplus-266'. In tacplus that is the one manager's time, serialized, against a backlog of 43 (report to merge p90 5288.6 min tonight). It recurs with every hand-land until the merge train (looper#86) is in, and after that whenever a fix is cherry-picked or two reports make the same change. git rebase already drops an applied patch. The manager only has to look at what is left. Also serves Cost: no build-pool time spent on main itself

## Evidence
- https://github.com/tacplus-dev/tacplus/issues/351#issuecomment-6845948662 (hub, 2026-11-13 06:29Z: 'already on main as 822e15f0 ... git cherry shows it as applied. It was still queued for landing, where each pass spent a full 3-hour gate run on nothing'); the same for #554, https://github.com/tacplus-dev/tacplus/issues/554#issuecomment-6845949736 (5380efab)
- https://github.com/tacplus-dev/tacplus/issues/266#issuecomment-6841300275 (hub-landed 0d66c1b..4205075 at 01:45Z: 'Looper: drop the queued land of looper/tacplus-266; it is on main')
- https://github.com/tacplus-dev/looper/blob/423a103c4bf657a1b722ecc8e69a1b67502f310f/repo-looper#L880 (land_work: 'nothing to land' only when rev-list --count ONTO..HEAD is 0, before rebase_on_main), and #L832 (choose_gates: 'no changed path' keeps GATES_SET=full)
- recap 2026-11-13, manager/tacplus/backlog: merge backlog p90 43, max 43; lands waited 439.42 h for the manager; tacplus merged 2 of 5 lands, report to merge p50 5241.5, p90 5288.6 min (https://gitlab.com/tacplus-dev/looper/-/jobs/17956849015)

## Cost
Hours in repo-looper, by a looper. In land_work, right after rebase_on_main, if git diff --quiet ONTO HEAD -- ':!.looper' then log 'nothing to land: main has this change already' and exit 0, so the existing rc 0 path gives 'The manager had nothing to merge ... main has its commits already' and closes the land. The same check in judge_land before land_fits, so such a land is never deferred for want of time. Tests in tests/repo-looper.test.sh: a branch whose patch was cherry-picked to main lands nothing, runs no gates and gets the nothing-to-merge verdict. No money

## Touches
decisions/throughput-first-2026-11-11.md (point 3: the hub lands fixes by hand; done when tacplus merges 10 a day), decisions/merge-train-2026-11-04.md and looper#86 (batches: an empty member would cost a batch build too), looper#134 (build once per commit: a different commit each pass, so it does not catch this), looper#114 (a verdict kept when main moved by planning files only: the same spirit)

## Reuse considered
The manager already has an rc 0 'nothing to merge' verdict and comment (repo-looper L1326), so only the check moves after the rebase. Considered: (1) the hub dropping each queued land by hand, as tonight, which is what costs the hub and the queue. (2) looper#134's cache of (repo, commit, gate set), which misses because each rebase is a new commit. (3) git cherry before the rebase, which misses a patch that applies with a different context; the diff after the rebase is the exact test

## Advice
Recap: 2026-11-13 manager/tacplus/backlog
Cause: a land whose change the hub already put on main (a cherry-pick, a new SHA) passes the manager's commit-count check, is rebased to no change, and choose_gates runs full on 'no changed path': a 3-hour full gate run per pass on main itself, in tacplus's one serialized manager
Change: tacplus-dev/looper repo-looper: land_work and judge_land stop with 'nothing to land: main has this change already' (rc 0) when git diff ONTO HEAD, .looper/ excluded, is empty after rebase_on_main; a test in tests/repo-looper.test.sh
Gain: 3 h of tacplus's manager per already-landed land per pass (tacplus#351 and #554 at least one pass each tonight, #266 queued too), so about 6 to 9 manager-hours tonight, out of the 439.42 h its lands waited; every later hand-land costs nothing

## Status
open
Rank: 2
Filed: 2026-11-13 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17999306134)
