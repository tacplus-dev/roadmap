Suggestion: The manager takes queued lands by priority, then oldest
Type: Performance
Summary: tacplus's manager merged 3 lands tonight against a backlog of 43 (report to merge p50 2140 min, 372.65 h waited), taking them oldest first. Its 8 priority:high lands wait behind 30 older ones, among them #315 (the blocker of looper#84, queued since 2026-11-09 01:57Z) and #465 (gates red on main). Have the verdict job take priority:high lands first, then oldest
Why: Fourth night over the threshold: lands waited 372.65 h for tacplus's manager, which merged 3. With full gates on every land (no gates.yaml, tacplus#266) and a run of over 3 h each, a verdict job gets through a few lands, and oldest-first puts whatever was queued last at the back of a 38-land queue. The kicker and the plan already order reports by priority (looper#31), but the manager doesn't order lands. Among those waiting: tacplus#315 (rootless BuildKit), which blocks looper#84, the fix for the 421.8 h a night of VM waits; #465 (the expired-token member-port test, red on main); #284 (a tid read from the namespace claim); #376 (the security floor's generated recipe); #283 (images with fixed HIGH CVEs). Merge throughput is #86's, #266's and #300's to raise, and none of them is in progress. This reorders the same throughput so the lands that unblock the rest go first. No cap, and nothing serialized that wasn't.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17872608667 (recap 2026-11-10, manager/tacplus/merge-time: report to merge p90 2907.6 min over 3 merges, p50 2140; lands waited 372.65 h for the manager, 4 of 7 nights; managers.tacplus: 9 lands, 3 merged (#313, #369, #223), red gates 0; concurrency: merge backlog p90 43, max 43)
- https://github.com/tacplus-dev/looper/blob/main/README.md (the manager 'takes the queued branches oldest first, one at a time'; the split verdict job 'takes the queued lands oldest first')
- https://github.com/tacplus-dev/tacplus/issues?q=is%3Aopen+label%3Alooper%3Alanding (38 open issues looper:landing on 2026-11-10; priority:high: #283, #284, #315, #340, #376, #465, #475, #494; the oldest, #196, from 2026-10-27)
- https://github.com/tacplus-dev/tacplus/issues/315 (all of it queued to land in 4ff7e97c since 2026-11-09 01:57Z, no verdict; it is Blocked-by of https://github.com/tacplus-dev/looper/issues/84)

## Cost
Hours in looper: repo-looper's verdict (and the unsplit manager's) sorts the queued lands by their issue's priority label (high, none, low), then by the land's age, the order the plan already uses for reports. Tests in tests/repo-looper.test.sh: manager-order with a newer priority:high land merged before an older unlabelled one, and equal priorities still oldest first. No money.

## Touches
looper#31 (priority order), looper#70 (manager per repository), looper#86 (merge train: in queue order, which this sets), looper#84 and tacplus#315, tacplus#266 and #300 (gate time), Jody's 'I just want stuff to go faster' (looper#57)

## Reuse considered
The plan's priority ordering of ready issues (#31) gives the ranking to reuse. #86's batch takes 'every queued branch at once, in queue order', so this order carries into it. A batch that goes red bisects, and high-priority lands first keep them out of the bisection behind older ones. #266 (gate map) and #300 (parallel gates) raise merges per hour, and this doesn't replace them. Not proposed: a cap, or skipping any land.

## Advice
Recap: 2026-11-10 manager/tacplus/merge-time
Cause: the verdict job runs tacplus's full gates (over 3 h, no gates.yaml) once per land and takes the queue oldest first, so 3 lands merge a night out of 43, and the priority:high fixes (#315, #465, #284, #376) wait behind 30 older lands
Change: looper's repo-looper: the verdict/manager orders queued lands by their issue's priority label, then age (README 'The manager'); throughput stays tacplus#266, tacplus#300 and looper#86
Gain: the 8 priority:high lands merge in the next 1–3 nights rather than after the ~30 older ones (about 10 nights at 3 merges a night). #315 landing starts looper#84's way off the 421.8 h a night of VM waits. Total hours waited are unchanged until #86/#266/#300.

## Status
open
Rank: 3
Filed: 2026-11-10 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17872762621)
