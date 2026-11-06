Suggestion: tacplus's cap from 6 to 8, now that its reports were running at it
Type: Performance
Summary: For a second night counted by running reports only (#92's rule), tacplus kept all 6 reports running for 5.85 h while 29 ready issues waited (168.75 issue-hours), with a merge backlog p90 of 1. Raise its cap in caps.yaml to 8 and read the next recap: the pool, not the cap, becomes the bound again where its VMs are full
Why: Last night's suggestion kept the cap at 6 because 5 of the 6 reports at it were queued for a VM (accepted, looper#92). Since #92 the recap counts a raise only for waits while the reports at the cap were running. By that count tacplus waited 168.75 h at the cap, with all 6 running in 54% of the kicker's 195 passes. It is the night's largest item, on the platform's critical path (tacplus#163 onward). The manager kept up (merge backlog p90 1), so more reports will not pile up unmerged. It is not free throughput: the same pool also left reports of other repositories waiting 78.6 h for a VM. In the hours that pool was full, 2 more tacplus reports take VMs other repositories would have used. A raise of 2, not more, puts tacplus's critical-path work first without crowding out the rest, and the next recap shows which way it went. The lasting fix is looper#84, where pod requests replace caps.yaml.

## Evidence
- https://github.com/tacplus-dev/looper/blob/main/caps.yaml (tacplus: 6). Recap 2026-11-06, cap/tacplus/raise: ready issues waited 168.75 h at the cap with all its reports running (29 issues, #190 to #323, 5.85 h each but #318's 4.97), 122.37 h more with some queued for a VM; all busy 97% (10.22 h), all running 54% (5.85 h) of 195 passes; merge backlog p90 1, max 6; running-basis nights: 2
- https://gitlab.com/tacplus-dev/looper/-/jobs/17795301568 (recap 2026-11-06, capacity/tacplus-dev arm64 autoscaler: its reports waited 78.6 h for a VM, e.g. tenantecho#4 216.2 min, doppelganger#5 213.1 min, coop#1 209.1 min; the pool ran 346 jobs on 9 VMs)
- https://github.com/tacplus-dev/roadmap/blob/main/suggestions/2026-11-05-the-recap-s-cap-advice-counts-running-reports-not-queued-ones.md (accepted as looper#92: keep 6 while the cap's reports were queued; raise only on waits while they ran)

## Cost
Minutes: one line in tacplus-dev/looper's caps.yaml (tacplus: 8), landed as any change. No money. The risk is the pool: 2 more tacplus reports can hold 2 of the shared pool's VMs, so other repositories' VM waits may grow by up to 2 × 5.85 h a night. The next recap's capacity/<pool> item shows it, and the cap goes back to 6 on its word.

## Touches
decisions/looper-job-per-issue-2026-11-03.md (caps in a committed config, adjusted on the recap's word two nights running; capacity manual), decisions/only-vms-are-k8s-nodes-2026-11-04.md (no new VMs), looper#72 and #92 (the recap's cap advice), looper#84 (pod requests replace caps.yaml), the platform ROADMAP's Next 1 (tacplus#163, the Stratum path)

## Reuse considered
caps.yaml is the decided mechanism, so nothing new is needed. Considered and not proposed: a scale-up (rejected twice, nodes-only), and raising every busy repository's cap (eventbus's and looper's raises are one night each). Priority labels (#31) already order ready issues within tacplus, so the 2 extra slots go to its highest-priority issues.

## Advice
Recap: 2026-11-06 cap/tacplus/raise
Cause: tacplus's cap of 6 in caps.yaml: all 6 of its reports ran for 5.85 h while 29 ready issues waited, a second night on the running-reports count (#92), with the manager keeping up (backlog p90 1)
Change: caps.yaml: tacplus: 8
Gain: up to 11.7 issue-hours a night (2 more reports over the 5.85 h all 6 ran) of the 168.75 h waited at the cap, less what the shared pool's full hours turn into VM waits (its reports waited 78.6 h for a VM tonight)

## Status
held (Jody, 2026-11-06): keep tacplus at 6 until the in-cluster runners replace the VM pool (looper#84); revisit on the recap after that
Rank: 3
Filed: 2026-11-06 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17802225217)
