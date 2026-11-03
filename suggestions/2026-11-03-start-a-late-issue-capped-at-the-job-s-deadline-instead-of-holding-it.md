Suggestion: Start a late issue capped at the job's deadline instead of holding it
Type: Performance
Summary: When a free slot opens but an issue's limit outlasts the job's time left, start it anyway with its limit capped at the job's deadline (the #35 path); it checkpoints and resumes in the next job
Why: Tonight 61 holds of 24 tacplus issues (58.76 issue-hours) were 'limit would outlast this job': free capacity sat idle until the job ended and the kicker started another. Checkpoints and resume (#20, #21), the deadline cap (#35) and land-pending (#38, #44) already make a run cut by the deadline safe. The work just has to be allowed to start. This is the item's own parallel: work instead of idling.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17666176518 (recap 2026-11-03, held/tacplus/limit_outlasts_job: 61 holds, 24 issues, up to 1.78 h each, 58.76 h in all; e.g. #222 and #256 waited 1.78 h for this reason alone)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17657507272 (recap 2026-11-03: the same holds in tacplus's earlier job; also https://gitlab.com/tacplus-dev/looper/-/jobs/17679641121)
- https://github.com/tacplus-dev/looper/blob/main/README.md (the parent's plan, 'the job's time': no child starts that would outlast the job, less its hour; a child's limit grown after sizing is already capped at the deadline and stopped with a checkpoint, #35)

## Cost
Hours to a day in repo-looper's plan: replace 'would outlast the job, hold' with 'start capped at the deadline' when at least LOOPER_MIN_CHILD_MINUTES (say 60–90, enough for a useful step) remains, else hold as now. Tests: a late issue starts capped, checkpoints at the deadline, and the next job resumes it. It costs model time on runs that only get partway, which is cheap next to idle slots.

## Touches
looper#33 (job sizing), looper#35 (the deadline cap), looper#38/#44 (land-pending: a capped run's land is landed first next job), looper#59 (re-plan fills free slots), looper#57 (the recap)

## Reuse considered
The PROPOSALS entry 'Size the job for what fills a slot later' (#33) would lengthen jobs for a second round instead, trading this wait for a job held longer when a run hangs, and it needs the 11 h ceiling. Capping reuses #35's existing stop-and-checkpoint path, with no change to job sizing. 'A newly ready issue in a job with no free slot' (#59) is the no-slot case, not this one.

## Advice
Recap: 2026-11-03 held/tacplus/limit_outlasts_job
Cause: the plan never starts a child whose issue limit (4 h default) exceeds the job's time left, so slots freed late in a job stay empty until the job ends
Change: repo-looper's plan: start such an issue with its limit capped at job_deadline when LOOPER_MIN_CHILD_MINUTES remain (the #35 cap), instead of holding it for the next job
Gain: most of the 58.76 issue-hours a night held by this reason in tacplus (up to 1.78 h per hold): the issue makes a checkpointed start instead of waiting for the next job

## Status
rejected
Rank: 4
Filed: 2026-11-03 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17692416595)
Decided: 2026-11-03 by Jody: Superseded by the looper redesign (Jody, 2026-11-03: one job per issue, a manager per repo; decisions/looper-job-per-issue-2026-11-03.md): there are no conflict or time-left holds to tune any more.
