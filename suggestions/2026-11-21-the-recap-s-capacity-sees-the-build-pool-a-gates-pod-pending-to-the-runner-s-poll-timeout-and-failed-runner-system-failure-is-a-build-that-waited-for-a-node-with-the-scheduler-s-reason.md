Suggestion: The recap's capacity sees the build pool: a gates pod Pending to the runner's poll_timeout and failed runner_system_failure is a build that waited for a node, with the scheduler's reason
Type: Operability
Summary: Tonight 114 tacplus-dev/build jobs failed runner_system_failure (55 gates jobs of the last 100; 28.79 h of waiting, the night's second-largest item, 4 nights running) and the recap's capacity line said the pool was full for 0 of 1714 pods and listed control-runner pods alone: it reads a pod's Pending time from the looper project's job logs and nothing of the build runner's, whose gates pods wait Pending 15 minutes (cluster's poll_timeout = 900) for a spot node, fail, and are retried twice by ci/build.gitlab-ci.yml. The hub read the cause by hand three times in a day (cloud#85 21:30Z: the spot pool stuck at 2 of 5 nodes; cloud#87 00:31Z: 'about 20 of them ... 0/5 nodes are available: 2 Insufficient cpu, 2 Insufficient memory'; looper#209 06:32Z). Have looper-recap read the build project's jobs' Pending time and the scheduler's reason, count a job failed at poll_timeout as a build that waited for a node, and give the capacity item the spot pool's nodes against its max and one more node's cost, so builds backing up reaches Jody from the recap
Why: Jody: 'if builds are backing up somebody tells me'. The recap is the one place that would, and it said the opposite on the night tacplus merged nothing of 16 lands. Operability first; Performance second, since the scale-up (cloud#85's max 5 against 2 running, 'no capacity change' Jody's word) becomes a decision with the morning's numbers rather than three by-hand reads a day.

## Evidence
- https://gitlab.com/tacplus-dev/build/-/jobs/18231780359 (recap 2026-11-21, failures/tacplus-dev/build/runner_system_failure: 114 jobs, 16 recovered on retry, 28.79 h, seen 4 of 7 nights; of the last 100 failed build jobs 55 are gates, 3 pending, 3 plan, all on ref main between 00:11Z and 09:34Z; runner_pools: the in-cluster build runner 715 jobs, pods_pending 0, pool_full 0; capacity: 'the pool was full for 0 of 1714 pods', its 79 waiting_for_room jobs every one a control-runner pod; projects.tacplus-dev/build: 290 jobs canceled, 162 failed, 544 succeeded)
- https://github.com/tacplus-dev/cloud/issues/87 (closed 2026-11-21 04:02Z: 'Every tacplus gates job there failed runner_system_failure after 15 minutes Pending, about 20 of them between 22:00Z and 00:30Z, with 0/5 nodes are available: 2 Insufficient cpu, 2 Insufficient memory ... Preemption is not helpful'; read by the hub, not the recap)
- https://github.com/tacplus-dev/cloud/issues/85 (open: 'the gates pod waits Pending for 15 minutes ... and the job times out'; the spot pool at 2 of 5 nodes in one zone; its part 2, merged 03:41Z, reads the autoscaler's NotTriggerScaleUp events into cloud's audit artifact, which the recap could quote)
- https://github.com/tacplus-dev/cluster/blob/main/gitlab-runner/runners/build.yaml ('A spot node from 0 takes minutes to come up. poll_timeout = 900': a pod with no node is a runner_system_failure after 15 minutes)
- https://github.com/tacplus-dev/looper/blob/main/ci/build.gitlab-ci.yml (default retry: max 2 on runner_system_failure: a Pending pod costs a build 45 minutes and three failed jobs before its requester reads no verdict)
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md (The pod's Pending time from timestamps, not the preparation, 2026-11-15, #84: the recap reads a pod's wait for room from the job log's 'status is Pending' lines, the looper project's jobs)

## Cost
A day in looper-recap: read tacplus-dev/build's jobs (tacplus-looper-bot is its Reporter since cloud#12) for the kubernetes executor's Pending lines and the Unschedulable reason, class a job failed at poll_timeout as waited for a node with that reason, sum it per repository and per pool beside the control runner's, and read the spot pool's nodes against its max (cloud's audit artifact, or the cluster API through the audit's container.viewer); looper-recap classes names it capacity/<pool> with one more node's cost; a test with a stub job log.

## Touches
looper#57 and #72 (the recap: the capacity Jody scales by hand on), decisions/looper-job-per-issue-2026-11-03.md and only-vms-are-k8s-nodes-2026-11-04.md (capacity is manual, a scale-up on Jody's word with numbers), cloud#85 (the pool's max 5), looper#209 and cluster#42 (a land's build always has a node)

## Reuse considered
looper-recap already reads the build project's jobs (queue, run and failure class) and the control runner's Pending lines, so the build runner's are the same read on the other project; cloud#85's audit now lists the autoscaler's events and what fills each spot node, which the recap can quote instead of inferring; the accepted suggestion of 2026-11-20 (looper#206, queued_duration) measures a job's wait for a runner, not a pod's wait for a node, which is this.

## Advice
Recap: 2026-11-21 failures/tacplus-dev/build/runner_system_failure
Cause: gates pods with no spot node to run on (the 3500m request that fit no node until cloud#87's 3 CPU at 04:02Z, the pool stuck at 2 of 5 nodes, cloud#85, and a red round's 24 builds at once, looper#209) wait Pending to the runner's poll_timeout and fail runner_system_failure, three times with the retry; the recap reads no Pending pod of the build runner, so its capacity line said no pool was full on the night 114 builds failed for want of a node
Change: looper-recap: the build project's jobs' Pending time and scheduler reason, a job failed at poll_timeout counted as waited for a node, the spot pool's nodes against its max and one more node's cost in capacity/<pool>; the pool itself is cloud#85's (open, backed off until 2026-11-24) and the round's size looper#209's
Gain: no hours by itself: the 114 failures a night become one capacity line with their reason the morning after, and the scale-up (a third spot node where the autoscaler stalled, about $2.14 a day for a c4a-standard-2, more for the c4a-standard-4 a tacplus gates pod needs) is Jody's decision on numbers instead of three by-hand reads a day; the 28.79 h themselves go with cloud#85 and looper#209

## Status
open
Rank: 5
Filed: 2026-11-21 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18244259587)
