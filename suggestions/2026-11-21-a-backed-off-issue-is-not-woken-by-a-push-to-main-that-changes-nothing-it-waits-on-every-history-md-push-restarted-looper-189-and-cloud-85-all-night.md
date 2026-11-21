Suggestion: A backed-off issue is not woken by a push to main that changes nothing it waits on: every HISTORY.md push restarted looper#189 and cloud#85 all night
Type: Cost
Summary: A backed-off issue is ready again on any push to its repository's main (README, Backing off), so every HISTORY.md, STATE.md or PROPOSALS.md push the manager makes after a merge restarts every backed-off issue of that repository, each a model run that reads the issue and says nothing to change. Tonight: looper#189 14 runs (each 'waits until 2026-11-21T10:00Z' within a minute), cloud#85 10 ('Nothing to change: the push 222c118..1680be4 is HISTORY.md's #87 line, unrelated to #85'), tacplus#909 6, tacplus#830 5, fleet-seatrials#56 5, kernel#72 4 ('after main's push 4690915..892c9ec (PROPOSALS.md only: no Dockerfile/base change)'): 46 runs, and 79 control-runner pods that queued 5 to 11 minutes each (9.34 h) ahead of reports with work. Wake a backed-off issue on a push to main only when the push changes a file outside HISTORY.md, STATE.md and PROPOSALS.md, or one the run's checkpoint named; a member's comment, a label and a blocker closing wake it at once as today
Why: Jody wants things faster: 46 of the night's model runs and 79 of its control pods did nothing but re-read an issue, and each pod held one of the control runner's 12 slots while the reports that had work queued (the capacity/pods item, 4 of 7 nights). Cost first (the runs), Performance second (the queue). The open suggestion of 2026-11-19 has the recap count these waits as waiting by choice; this stops the runs.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/18241783654 (recap 2026-11-21, capacity/pods: 79 issues waited 9.34 h for room on the normal pool, the pool full for 0 of 1714 pods, queue p90 10.2 min, 4 of 7 nights; kernel#72 15 pods (10.9 min here), tacplus#909 13, fleet-seatrials#56 12, llmproxy#67 11, oauth#31 11, looper#189 9, tacplus#830 8, cloud#85 8: all backed-off or waiting-until issues; the concurrency item: 16 reports at once at most, 6.64 on average, over 93 passes)
- https://github.com/tacplus-dev/looper/issues/189 (14 'The looper of looper starts this issue' comments between 2026-11-20 10:00Z and 2026-11-21 10:01Z, each followed within a minute by 'The run waits until 2026-11-21T10:00:00Z on something outside the looper'; looper's main moved 13 times in the window, 6 of them HISTORY.md lines)
- https://github.com/tacplus-dev/cloud/issues/85 (10 starts; 03:58Z, 04:01Z and 04:04Z each 'Nothing to change: the push ... is HISTORY.md's #86 line' or '#87 line, unrelated to #85'; its wait is Jody's logging.configWriter grant)
- https://github.com/tacplus-dev/kernel/issues/72 ('Re-checked 2026-11-21 after main's push 4690915..892c9ec (PROPOSALS.md only: no Dockerfile/base change)', 10:04Z; its wait is a glibc fix in trixie, until 2026-12-04)
- https://github.com/tacplus-dev/looper/blob/main/README.md (Backing off: not ready 'until its until has passed or something changed after its since: a comment by a member ..., a label added or removed ..., a push to its repository's main (main is no longer that sha), or one of its Blocked-by: issues closing')
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md (A backed-off issue's reads every pass, and a first run that finds nothing, 2026-11-17, #181: the kicker's three reads an issue a pass; the wakes are the other half)

## Cost
Hours in looper-dispatch and repo-looper: the ready rule reads the push's changed paths (one compare from the sha the Backed-off line records to main's head, on the forge the kicker already reads) and ignores a push touching only HISTORY.md, STATE.md and PROPOSALS.md; a --wait-until run may name the paths it waits on (its Dockerfile, its lock) in the line, and only a push to those wakes it; a wake that ends with nothing to change keeps its until rather than resetting a day. Tests in tests/looper-dispatch.test.sh (backed-off-push: a planning-file push does not wake, a code push does) and tests/repo-looper.test.sh.

## Touches
looper#181 (the back-off), #57 and #72 (the recap's capacity and concurrency), decisions/looper-job-per-issue-2026-11-03.md (capacity is manual: slots go to work), Jody's rule of 2026-11-07 that loopers are never blocked (a member's comment, a label, a blocker closing still wake it at once), looper#162 (the models' tokens)

## Reuse considered
The Backed-off line already records main's sha at its since, so the compare is one request the kicker can make; the planning files are the set the manager itself appends (LOG_ATTRS: PROPOSALS.md and HISTORY.md, and STATE.md's last commit), so the kicker knows which pushes are its own; looper#181's tests cover the push case and need the two new ones. The open suggestion of 2026-11-19 (the recap reads the Backed-off line) is the count, not the cause.

## Advice
Recap: 2026-11-21 capacity/pods
Cause: the kicker's ready rule for a backed-off issue (README, Backing off) counts any push to main as a change, so the manager's own HISTORY.md, STATE.md and PROPOSALS.md pushes after each merge restart every backed-off issue of the repository: 46 model runs and 79 control pods tonight for eight issues waiting on a date, a grant or an upstream fix
Change: looper-dispatch's ready rule and repo-looper's back-off: a push to main wakes a backed-off issue only when it changes a file outside HISTORY.md, STATE.md and PROPOSALS.md, or one the run named in its Backed-off line; a run that wakes to nothing to change keeps its until; tests/looper-dispatch.test.sh backed-off-push split in two
Gain: about 46 model runs and the 79 no-op control pods a night (9.34 h of queue tonight), the control runner's 12 slots left to reports with work; the wait items of looper (4.48 h), cloud (3.31 h), oauth (3.11 h), llmproxy (2.51 h), fleet-seatrials (2.11 h) and kernel (1.25 h) are these issues' own back-offs, which the open suggestion of 2026-11-19 has the recap count as waiting by choice

## Status
open
Rank: 4
Filed: 2026-11-21 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18244259587)
