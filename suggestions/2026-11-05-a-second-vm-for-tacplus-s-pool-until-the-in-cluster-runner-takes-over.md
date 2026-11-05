Suggestion: A second VM for tacplus's pool until the in-cluster runner takes over
Type: Performance
Summary: tacplus's 16-CPU pool ran all 32 of its jobs on 1 VM, one at a time, and its reports waited 123.7 of the night's 135.53 VM-hours. The in-cluster runner that ends this (looper#84) is blocked on tacplus#315, which is itself waiting in that queue. Jody's call: one more VM in that pool now, removed when #84 lands
Why: This is the second night over the threshold, and it grew from 22.39 h to 135.53 h waited for a VM (queue p90 227.3 min, max 418.9). Five tacplus reports (#259, #274, #282, #284, #300) each waited about 7 h in a single plan. Last night's 'Fill the tacplus VM' suggestion opened with packing four jobs per VM, but decisions/only-vms-are-k8s-nodes-2026-11-04.md has since replaced 'pending work that adds or tunes runner VMs (e.g. packing jobs per VM)' with the in-cluster runner. That leaves the scale-up, which is Jody's to decide. The way out is circular: looper#84 is Blocked-by tacplus#315 (rootless BuildKit), and #315 waited 20.87 h at tacplus's cap behind the same VM. A second VM shortens the queue that holds the fix. Also serves Operability: the in-cluster plan sizes the normal pool for 3 to 4 report pods across all repositories, and tonight tacplus alone kept 6 reports busy 99% of the night. These are the numbers Jody asked to see before that cut-over.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17747893169 (recap 2026-11-05, capacity/vm: 61 issues waited 135.53 h for a VM, 123.7 h of it tacplus's; queue p90 227.3 min over 220 jobs, max 418.9; tacplus#300 418.9 min here, #284 418.7, #282 418.6, #274 418.4, #259 417.8 in the same plan; one more t2a-standard-8 about $7.39 a day)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17757254999 (recap 2026-11-05, runner_pools: the 16-CPU autoscaler, MIG runner-arm64-tacplus-big, ran 32 jobs on 1 VM, disk p50 67%, max 84%; the other pool ran 188 jobs on 9 VMs, and its longest VM wait was 57.8 min, looper#85)
- https://github.com/tacplus-dev/looper/issues/84 (approved, Blocked-by tacplus-dev/cluster#6, closed, and https://github.com/tacplus-dev/tacplus/issues/315, open; recap 2026-11-05 caps.tacplus: #315 waited 20.87 h at the cap of 6, all reports busy in 99% of 410 kicker passes)
- https://github.com/tacplus-dev/roadmap/blob/main/decisions/only-vms-are-k8s-nodes-2026-11-04.md (Consequences: packing jobs per VM replaced by the in-cluster runner; capacity stays manual, scale-up on Jody's yes with numbers)
- https://github.com/tacplus-dev/looper/blob/main/README.md (Two pools: tacplus's runner 58307468 on runner-arm64-tacplus-big, t2a-standard-16, 'up to 2 VMs'; changing max_instances needs a runner restart, which kills running jobs)

## Cost
One more VM a day: about $7.39 per t2a-standard-8 (the recap's capacity.one_more_vm, on-demand list price). The pool's own t2a-standard-16 is about twice that, and the pool's spot template (cluster-node-pools-2026-11-03.md) costs less. That is for the days until looper#84 lands, and scaling down needs no ask. The hub sets the MIG's and runner 58307468's max to 2 in a quiet window, since a runner restart kills running jobs.

## Touches
decisions/looper-job-per-issue-2026-11-03.md and only-vms-are-k8s-nodes-2026-11-04.md (capacity is manual: scale-up on Jody's word, with numbers; down without asking), decisions/merge-train-2026-11-04.md addendum (start with 3 normal nodes and learn from the recap), looper#72 (the recap's capacity), looper#84 and tacplus#315 (the way off runner VMs), suggestions/2026-11-04-fill-the-tacplus-vm-before-adding-one-four-reports-a-vm-on-the-16-cpu-pool.md (its packing step since superseded)

## Reuse considered
Considered: (1) raising tacplus's cap (the recap's cap/tacplus/raise, 544.91 h) adds pending jobs, not running ones: 5 of the 6 reports at the cap were themselves queued for the VM. (2) Packing jobs per VM (last night's suggestion) is ruled out by the 2026-11-04 decision. (3) looper#84's in-cluster runner is the decided end state, but it waits on tacplus#315. (4) Moving tacplus's manager and scan to the 9-VM pool (filed tonight) frees this VM of 2 of its jobs at no cost, but it leaves the reports one at a time. Only a second VM runs two reports at once. The scale-up itself is a setting on an existing MIG, and nothing new is built.

## Advice
Recap: 2026-11-05 capacity/vm
Cause: every tacplus job (reports, its manager, its scan) is tagged for runner 58307468's 16-CPU pool, which runs one job per VM and ran 1 VM all night, so tacplus's cap of 6 runs one job at a time and the rest queue for hours
Change: Jody's scale-up: max 2 VMs on MIG runner-arm64-tacplus-big and runner 58307468 (the README's 'up to 2 VMs') until looper#84 moves tacplus's jobs in-cluster, then back to none. Bring tonight's numbers to the normal pool's sizing at #84's cut-over.
Gain: about half of the 123.7 h tacplus's jobs waited for a VM tonight, so roughly 60 h a night, since two reports run at once instead of one. #315, the blocker of #84, gets a slot sooner.

## Status
open
Rank: 2
Filed: 2026-11-05 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17765898686)
