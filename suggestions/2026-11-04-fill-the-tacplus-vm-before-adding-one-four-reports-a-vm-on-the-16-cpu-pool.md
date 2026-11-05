Suggestion: Fill the tacplus VM before adding one: four reports a VM on the 16-CPU pool
Type: Performance
Summary: tacplus's cap is 6 reports, but its 16-CPU pool gives each job a whole VM (capacity_per_instance = 1, set for the retired parent's 5 children a job), so its reports run one at a time and waited up to 4.7 h for a VM; pack four a VM first, then decide on a second VM with the numbers
Why: Since looper#70 each tacplus issue is its own job, but the tacplus runner still holds one job per t2a-standard-16 (64 GB): a single report, typically 4–12 GB, has the whole VM. Tonight that pool ran 9 jobs on 1 VM, and tacplus#284 and #275 queued 280.5 and 277.1 minutes behind #259 and #274. The kicker saw all tacplus reports busy in 97% of its passes, so the cap of 6 in caps.yaml is not what limits tacplus: the VM is. The decision already says containers share VMs (decisions/looper-job-per-issue-2026-11-03.md), and cloud#5's shared pool implements it, but that pool is registered paused until the move off jody1-328023. Also serves Cost: the speed-up comes before any new VM.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17719405071 (recap 2026-11-04, capacity/vm: 22 issues waited 22.39 h for a VM, queue p90 39.6 min, max 280.5; tacplus#284 280.5 min here, #275 277.1 min https://gitlab.com/tacplus-dev/looper/-/jobs/17719405070, #274 56.7 min https://gitlab.com/tacplus-dev/looper/-/jobs/17719405069, #259 53.9 min https://gitlab.com/tacplus-dev/looper/-/jobs/17719405068: 11.14 h after the redesign went live; the rest were the old parents' jobs on the 5-VM pool before 2026-11-03's cut)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356 (recap 2026-11-04 runner_pools: '16-CPU autoscaler (MIG runner-arm64-tacplus-big)' 9 jobs on 1 VM, disk p50 30%, max 63%; caps.tacplus: cap 6, all reports busy in 97% of 167 kicker passes (8.15 h), 195.7 issue-hours waited at the cap; one more t2a-standard-8 is about $7.39 a day)
- https://github.com/tacplus-dev/looper/blob/main/README.md (Two pools: the tacplus runner 58307468 'up to 2 VMs; 5 children a job (64 GB / 12; the CPUs allow 8)', and the loopers' entry capacity_per_instance = 1, 'one job on a worker at a time: a job's children share its VM')
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('Caps against the VM', #70: caps.yaml caps each repository's reports, not the VM; the runner's limit bounds it)

## Cost
Packing costs nothing: an edit to runner 58307468's [runners.autoscaler] in /etc/gitlab-runner/config.toml on gitlab-runner-mgr (capacity_per_instance = 4, max_use_count kept), applied by the hub in a quiet window, since a restart kills running jobs. The risk is memory: 4 reports at looper-docker's 12 GB Kani limit is 48 of 64 GB, and the manager's gates run unlimited (#41). Hold at 3 until tacplus#275/looper#74 take Kani out of the gates and reports go back to 4 GB. The scale-up, if packing is not enough, is Jody's call: one more t2a-standard-8 is about $7.39 a day (the recap's figure), and a t2a-standard-16 about twice that.

## Touches
decisions/looper-job-per-issue-2026-11-03.md (containers share VMs; capacity is manual, scale-up on Jody's word with numbers), looper#70 (job per issue), looper#72 (the recap's caps and capacity), looper#41 (report memory limits), looper#42 (fresh VM per job, which packing trades back), cloud#5 (the shared pool: two jobs of 4 CPUs and 14 GB per t2a-standard-8)

## Reuse considered
Considered: raising tacplus's cap in caps.yaml (the recap's cap/tacplus/raise, one night so far) changes nothing while the VM holds one job. Moving onto cloud#5's shared pool is the decided end state, but it waits on the cloud bootstrap and the runners' move. A second VM only doubles one-at-a-time. looper-docker's per-container memory limit (#41) already gives each report a bound, which is what makes packing safe enough for now.

## Advice
Recap: 2026-11-04 capacity/vm
Cause: the tacplus runner (58307468, 16-CPU, 64 GB) holds one job per VM (capacity_per_instance = 1), sized for the retired parent that ran 5 children in a job; with one job per issue a single report holds the whole VM, so the cap of 6 runs one at a time
Change: Before buying: capacity_per_instance = 4 (3 until tacplus#275 lands) in runner 58307468's [runners.autoscaler] on gitlab-runner-mgr. Then, if tacplus still waits over 1 h a night for a VM, Jody's scale-up: one more VM for the tacplus pool, about $7.39 a day per t2a-standard-8
Gain: up to the 11.14 h a night tacplus reports waited for a VM after the redesign (#284 280.5 min, #275 277.1, #274 56.7, #259 53.9), and most of the 195.7 issue-hours tacplus waited at its cap, since 4 reports run at once instead of 1

## Status
rejected (Jody, 2026-11-05): the only VMs are Kubernetes nodes; pods are packed by their requests on the in-cluster runner (tacplus-dev/looper#84)
Rank: 3
Filed: 2026-11-04 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356)
