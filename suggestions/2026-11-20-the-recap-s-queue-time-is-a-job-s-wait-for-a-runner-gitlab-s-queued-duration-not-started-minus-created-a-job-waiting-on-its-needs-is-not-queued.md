Suggestion: The recap's queue time is a job's wait for a runner, GitLab's queued_duration, not started minus created: a job waiting on its needs is not queued
Type: Operability
Summary: looper-recap counts every job's queue as started_at minus created_at, so a job created with its pipeline that waits on its needs (report-pass, report-fail, gates-failed, security-report, security-failed: needs gates) is 'queued' for the whole gates run. Tonight's five longest queues in tacplus-dev/build, 676 to 727.5 min, are all such jobs, behind gates jobs that ran 11 h to their timeout (the security-report of gates 18169753889's pipeline 'queued' 676.2 min), and they are in the item queue/tacplus-dev/build (p90 107.6 min over 705 jobs, over its 15-min threshold 7 of 7 nights) and in capacity.queue_minutes (p90 61.7, max 727.5). Read GitLab's queued_duration, the wait for a runner only, keep created-to-started as a second figure, and recompute the 7 nights
Why: Jody reads the recap for 'if builds are backing up somebody tells me': a queue p90 of 107 min that is largely the gates' own hour, seen from the jobs behind them, says the runners are short when they may not be (the spot pool refused 0 of 1385 pods tonight), and last night's suggestion to raise the build runner's concurrent was argued from that figure. Also serves Correctness: one definition of queue across the three projects.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/18215309726 (recap 2026-11-20, longest.queues: report-fail 727.5 min (job 18169088848), report-fail 680.5 (18170815178), security-failed 677.9 (18169827115), security-report 677.9 (18169827114), security-report 676.2 (18169753892); projects.tacplus-dev/build: queue p90 6457 s, max 43653 s, run max 39625 s; queue/tacplus-dev/build p90 107.6 min, 7 of 7 nights; capacity.queue_minutes p90 61.7 over 1731 jobs, max 727.5; pool_full 0 of 1385)
- https://gitlab.com/tacplus-dev/build/-/jobs/18169753889 (gates, job_execution_timeout, 2026-11-19, one of the night's three 11-h timeouts; security-report 18169753892 of its pipeline 'queued' 676.2 min behind it)
- https://github.com/tacplus-dev/looper/blob/main/looper-recap (c, s, f = when(created_at), when(started_at), when(finished_at); q = secs(c, s) for every job, needs or not; no use of queued_duration)
- https://github.com/tacplus-dev/looper/blob/main/looper-build (gates-failed, security-report and security-failed: needs: [{job: gates ...}], when: on_failure or always; ci/build.gitlab-ci.yml: report-fail when: on_failure, after the gates trigger)

## Cost
Hours in looper-recap: read each job's queued_duration from GitLab's jobs API for queue_s and the longest queues, keep created-to-started as a per-job 'waited' figure; tests/looper-recap.test.sh's fixtures get the field; the 7-night history recomputed from the recap's kept JSON.

## Touches
looper#57 and #72 (the recap says when builds back up), decisions/looper-job-per-issue-2026-11-03.md (capacity is manual, decided on the recap's numbers), the 2026-11-19 suggestion 'Fewer full builds and more at once' (its queue figure), looper#185 and #195 (slots sized from the recap)

## Reuse considered
GitLab's jobs API already returns queued_duration for every job, so no new source or token; looper-recap's dist() and the item thresholds stay as they are. Considered: leaving jobs with needs out of the queue, which loses their real wait for a runner once their needs are met.

## Advice
Recap: 2026-11-20 queue/tacplus-dev/build
Cause: looper-recap measures a job's queue as started_at minus created_at, so a job that waits on its needs (report-pass, report-fail, gates-failed, security-report, security-failed) counts the gates' whole run as queue
Change: looper-recap: queue_s and longest.queues from each job's queued_duration (GitLab's projects/<p>/jobs), created-to-started kept as a separate 'waited' figure; tests/looper-recap.test.sh
Gain: no false queue item: the 676 to 727 min 'queues' of tonight's five longest become the gates' run, and the build project's p90 is the runners' real wait, so a concurrent change or a node (capacity.one_more_node, about 2.14 USD a day) is asked for from it; 0 hours of waiting removed by itself

## Status
open
Rank: 5
Filed: 2026-11-20 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18215309726)
