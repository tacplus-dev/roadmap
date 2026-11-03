Suggestion: The kicker and plan on a runner of their own
Type: Performance
Summary: Move looper-kick and looper-plan off the 5-worker autoscaled pool (tags arm64, autoscale) onto a small always-on runner tag, so dispatch never queues behind hours-long looper jobs
Why: The kicker and the plan share the pool with every repository's looper and scan jobs: one job per worker, at most 5 workers, jobs up to 11 h. When the pool fills, the job that starts all the others waits too. Tonight four looper-kick jobs queued 155–162 minutes, and the queue p90 was 123 minutes. Ready work across the org isn't even planned meanwhile. Also serves Cost: the kicker holds a t2a-standard-8 for 57 minutes of every hour for a few gh/curl calls (PROPOSALS).

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17655033492 (recap 2026-11-03, queue/tacplus-dev/looper: queue p90 7381 s (123 min) over 150 jobs, p50 5 s, max 249 min; looper-kick queued 161.7 min here, 160.2 min https://gitlab.com/tacplus-dev/looper/-/jobs/17656141833, 157.7 min https://gitlab.com/tacplus-dev/looper/-/jobs/17657610061, 155.3 min https://gitlab.com/tacplus-dev/looper/-/jobs/17659375890)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17658507015 (recap 2026-11-03: looper-scan-tacplus queued 249 min; pool 'tacplus-dev arm64 autoscaler' ran 149 jobs on 5 VMs)
- https://github.com/tacplus-dev/looper/blob/main/ci/tacplus-dev.gitlab-ci.yml (looper-plan and looper-kick: tags [arm64, autoscale], the looper jobs' pool) and README Infrastructure (capacity_per_instance = 1, max_instances = 5)

## Cost
Hours plus a few dollars a month: a second runner entry with a docker executor on gitlab-runner-mgr (already always on) or an e2-small, tag e.g. 'dispatch', concurrency 2. Then tags: [arm64, dispatch] on looper-kick and looper-plan in ci/tacplus-dev.gitlab-ci.yml, which the hub copies to the project. It frees a t2a-standard-8 the kicker holds 57 min/h, about 0.95 of a worker.

## Touches
looper#24 (the kicker as a scheduled pipeline), looper#55/#56 (per-repository runner tags, looper-dispatch plan), looper#31 (priority: firm only if dispatch is not starved), looper#57 (the recap)

## Reuse considered
The PROPOSALS entry 'The kicker holds a worker around the clock' (#24) proposes the same runner for cost, and 'Priority within a capacity-short pipeline' (#31) a high-priority tag. This puts numbers on both and adds the plan job. Raising max_instances above 5 would also cut the queue but costs a t2a-standard-8 per worker. Moving dispatch off the pool is cheaper, and the first step either way. No open issue holds it.

## Advice
Recap: 2026-11-03 queue/tacplus-dev/looper
Cause: orchestration jobs (kicker, plan) share the one-job-per-worker, 5-worker pool with looper jobs of up to 11 h, so a full pool starves dispatch itself
Change: a 'dispatch' runner tag on a small always-on runner (gitlab-runner-mgr or an e2-small); tags: [arm64, dispatch] for looper-kick and looper-plan in ci/tacplus-dev.gitlab-ci.yml
Gain: the kicker's 155–162 min queues (4 tonight) drop to seconds; about one t2a-standard-8 worker (57 min/h) returned to looper jobs

## Status
open
Rank: 5
Filed: 2026-11-03 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17692416595)
