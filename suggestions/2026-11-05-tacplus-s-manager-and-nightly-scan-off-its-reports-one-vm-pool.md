Suggestion: tacplus's manager and nightly scan off its reports' one-VM pool
Type: Performance
Summary: LOOPER_JOB_TAGS_tacplus sends tacplus's manager and its 2-hour scan to the same one-job-per-VM pool as its reports. Last night lands waited 15.94 h for the manager (report to merge p90 511.5 min) and six reports queued 2.05 h behind the scan. Have the plan apply the per-repository tags to reports only, so the manager and the scan go to the 9-VM pool
Why: Since looper#70 a repository's jobs no longer share a lock: reports are resource_group looper-<repo>-<issue>, the manager looper-<repo>-main, the scan looper-<repo>. What still serializes tacplus is the pool. plan_job gives every tacplus job, the manager and the scan too, the 16-CPU pool's tags, and that pool runs one job per VM on 1 VM. A running report (up to 4.5 h) holds the VM, so the manager waits behind it even though the plan puts managers first (#75), and branches that are already done sit unmerged while later reports start from a stale main. The nightly scan held the same VM 123.9 min for the third night running. Neither job needs 16 CPUs: the full gates use about 1.5 of them (tacplus#300), and the scan changes no code. This is the item's own parallel: the manager and the scan run beside the reports instead of in turn.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17753137170 (recap 2026-11-05, manager/tacplus/merge-time: report to merge p90 511.5 min over 3 merges, p50 419, max 511.5, threshold 120; lands waited 15.94 h for the manager; merges of #274 https://gitlab.com/tacplus-dev/looper/-/jobs/17726416792, #275 https://gitlab.com/tacplus-dev/looper/-/jobs/17746311210 and #213 here)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17729413354 (recap 2026-11-05, scan/tacplus: the scan ran 123.9 min, threshold 60, the third night of 3; held/tacplus/scan_holds_repo: #213, #259, #274, #275, #282 and #284 queued 2.05 to 2.07 h while it ran, 12.29 h in all, e.g. https://gitlab.com/tacplus-dev/looper/-/jobs/17735829800)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17757254999 (recap 2026-11-05, runner_pools: the 16-CPU pool ran 32 jobs, all of tacplus's, on 1 VM, disk max 84%; the 8-CPU pool ran 188 jobs on 9 VMs, disk max 48%)
- https://github.com/tacplus-dev/looper/blob/main/looper-dispatch (plan_job: tags from LOOPER_JOB_TAGS_<repo> for every job of the repository, scans, managers and reports alike; ci/tacplus-dev.gitlab-ci.yml sets LOOPER_JOB_TAGS_tacplus to 'arm64, autoscale-16')
- https://github.com/tacplus-dev/tacplus/issues/300 (full gates use ~1.5 of 16 CPUs)

## Cost
An hour or two in looper-dispatch: LOOPER_JOB_TAGS_<repo> applies to report jobs only. A manager uses LOOPER_MANAGER_TAGS_<repo>, and a scan LOOPER_SCAN_TAGS_<repo>, each falling back to LOOPER_JOB_TAGS. Add a plan-tags scenario to tests/looper-dispatch.test.sh. The risk is memory: the manager runs the full gates in its job until #71 or #86 (Kani's 12 GB containers fit on a t2a-standard-8's 32 GB). If they don't, keep only the scan off and leave the manager.

## Touches
decisions/looper-job-per-issue-2026-11-03.md (the manager alone merges; the recap's report-to-merge score), looper#55 (per-repository runner tags), looper#75 (managers first), looper#70, looper#72 (the recap), looper#84 (in-cluster pods share nodes, which ends this), tacplus#300

## Reuse considered
The 9-VM pool and its tags already exist, and so does the per-repository override (#55). This narrows that override's scope, with nothing new to build. looper#84 removes the cause for good, since pods share nodes, but it is blocked on tacplus#315, which waits in this very queue. A second VM for the pool (filed tonight) adds capacity. This takes 2 jobs off the VM at no cost, and the two are complementary. Shortening the scan does not help: the scan would still hold the whole VM for its run.

## Advice
Recap: 2026-11-05 manager/tacplus/merge-time
Cause: the plan tags tacplus's manager and scan for the 16-CPU pool (LOOPER_JOB_TAGS_tacplus), which runs one job per VM on 1 VM, so the manager waits for a running report to end and the scan holds the VM for 2 h, while the 9-VM pool has room
Change: looper-dispatch plan_job: LOOPER_JOB_TAGS_<repo> for --issue jobs only; managers and scans use LOOPER_MANAGER_TAGS_<repo> / LOOPER_SCAN_TAGS_<repo>, else LOOPER_JOB_TAGS (arm64, autoscale)
Gain: up to 28.23 h a night: the 15.94 h lands waited for tacplus's manager and the 12.29 h six reports queued behind the scan; the VM's ~2 h a night of scan time goes back to reports

## Status
accepted (Jody, 2026-11-05): tacplus-dev/looper#91
Rank: 3
Filed: 2026-11-05 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17765898686)
