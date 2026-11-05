Suggestion: The recap's cap advice counts running reports, not queued ones
Type: Operability
Summary: Keep tacplus's cap at 6. The recap advises raising it (544.91 h waited at the cap, 2 nights running), but 5 of the 6 reports holding the cap were themselves queued about 7 h for the VM. Have looper-recap advise a raise only when the reports at the cap were running, and name the pool's queue as the cause otherwise
Why: Jody asked the recap to say when a cap needs adjusting (looper#72: 'recap should let us know if we need to adjust'). Tonight it says raise tacplus's, for the second night running, so the scan has to answer it. caps_of in looper-recap counts a pass as 'all busy' when the repository's created, pending and running report jobs reach the cap. The plan counts them the same way (free_reports). Tonight tacplus's six were mostly pending: #259, #274, #282, #284 and #300 queued 417.8 to 418.9 min in one plan. A raise would add more pending jobs on the same one VM and change nothing, and the advice repeats every night until someone sees why. The scan_holds_repo cause has the same stale basis: its comment says 'GitLab runs one job of a repository at a time', which ended with #70's per-issue resource groups. Its suggested parallel, 'a scan that does not hold the repository's lock', points at a lock that no longer exists, when the real cause is the shared VM. Also serves Correctness: advice the recap gives should follow from the numbers it shows.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17747893165 (recap 2026-11-05, cap/tacplus/raise: ready issues waited 544.91 h at the cap of 6, all reports busy in 99% of 410 passes (20.87 h), merge backlog p90 1; in the same plan, capacity/vm: #259 417.8 min queued for a VM here, #274 418.4, #282 418.6, #284 418.7, #300 418.9, https://gitlab.com/tacplus-dev/looper/-/jobs/17747893169)
- https://github.com/tacplus-dev/looper/blob/main/looper-recap (caps_of: a pass is at the cap when k >= cap, with no regard to whether those jobs ran; 'raise' is advised when issues waited there and the backlog is low; the scan_holds_repo comment: 'GitLab runs one job of a repository at a time')
- https://github.com/tacplus-dev/looper/blob/main/looper-dispatch (free_reports counts created, pending, waiting and running report jobs against the cap; reports are resource_group looper-<repo>-<issue>, the scan looper-<repo>, so a scan holds no report back by lock since #70)
- https://github.com/tacplus-dev/roadmap/blob/main/suggestions/recap/2026-11-04.md (the first night of cap/tacplus/raise; the night's 16-CPU pool ran 9 jobs on 1 VM)

## Cost
A few hours in looper-recap. The kicker's capacity lines already list each repository's reports; add how many of them are running (it has the statuses from active_job_statuses). Advise raise only when the running reports reach the cap. When most at-cap reports are pending, the item becomes 'capacity/<pool>' with the pool's wait instead. Attribute a queued report to scan_holds_repo only when the scan shares its resource group, and otherwise count it as a VM wait on the scan's pool. Tests in tests/looper-recap.test.sh: a cap full of pending jobs gives no raise; a scan on another pool holds nothing.

## Touches
looper#72 (the recap's caps and capacity; Jody: 'recap should let us know if we need to adjust'), looper#57 (Jody: 'I am looking for actionable advice'), decisions/looper-job-per-issue-2026-11-03.md (caps in a committed config, adjusted on the recap's word two nights running), looper#70 (resource groups per issue)

## Reuse considered
The kicker's capacity lines and the plan's job statuses already hold the running/pending split, so no new source is needed. looper#84 changes the measure to pod pending time, but it is blocked on tacplus#315. Until then, this advice is what Jody reads every morning.

## Advice
Recap: 2026-11-05 cap/tacplus/raise
Cause: the recap counts pending report jobs as using the cap, and 5 of tacplus's 6 were pending for the pool's one VM, so 'raise the cap' is advised where the cap is not what limits tacplus
Change: caps.yaml: tacplus: 6 (unchanged: no raise). looper-recap caps_of: at the cap only by running reports, else a capacity item for the pool; scan_holds_repo only for a shared resource group
Gain: no hours by itself: it stops a raise that would add pending jobs and no throughput, and points the 544.91 h at the VM (capacity/vm, 123.7 h of tacplus's queue tonight), where a change removes waiting

## Status
open
Rank: 4
Filed: 2026-11-05 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17765898686)
