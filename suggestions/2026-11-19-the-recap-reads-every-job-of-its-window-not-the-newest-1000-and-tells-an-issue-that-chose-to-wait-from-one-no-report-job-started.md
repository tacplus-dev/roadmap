Suggestion: The recap reads every job of its window, not the newest 1000, and tells an issue that chose to wait from one no report job started
Type: Operability
Summary: looper-recap reads a project's jobs 10 pages of 100 (LOOPER_RECAP_PAGES), so tonight it saw exactly 1000 of tacplus-dev/looper's: 'failures: {}' while the window held 57 failed looper jobs (the 23:53Z runner_system_failure burst of cluster#37: 7 kicks, a plan, 15 reports), and 9 kicks and 97 passes over 4.87 h against 22 kicks and 355 passes the night before, so every wait and concurrency figure covers a fifth of the night. And of the 28.59 h it reports as 'ready issues waited with no report job', 23.1 h are five issues that backed off by choice (looper#181's Backed-off line: tacplus#830 until 2026-11-23, fleet-seatrials#56 until 11-20, llmproxy#67 until 11-21, looper#189 and tacplus#273 until 11:00Z and 12:00Z), which the kicker in the pinned image (f489822, before #181) still lists as ready and starts: 153 llmproxy jobs, 144 fleet-seatrials, 120 oauth, most ending in 28 s. Have looper-recap page until the window's start (and say when a project's read was cut), read an issue's Backed-off line and count it as waiting by choice with its reason, and report the no-op jobs an issue got
Why: Jody: 'recap should let us know if we need to adjust', 'if builds are backing up somebody tells me'. Tonight's recap says looper had no failures and little waiting on the night the runners failed every job for seven minutes, and its advice would send the kicker's and plan's fixes after five issues waiting on Debian, the hub or a time. The requests item reads the same no-op pods (report CPU p50 468m tonight against 130m the night before). Also serves Correctness (the recap's contract is the 24 hours) and Cost (the no-op pods it would name)

## Evidence
- recap 2026-11-19 projects.tacplus-dev/looper: jobs 1000, failures {} (the night before 997, 2026-11-16 676); control.looper-kick.jobs 9 (22 to 32 on the nights before), concurrency.passes 97 over 4.87 h (355 over 18.44 h on 2026-11-18) (/builds/tacplus-dev/looper/looper-runs/20261119T093741Z-scan/context/recap.json); /builds/tacplus-dev/looper/looper-runs/20261119T093741Z-scan/context/failed-jobs-tacplus-dev_looper.json: 57 looper jobs failed inside the window, 51 of them runner_system_failure at 23:53Z (https://gitlab.com/tacplus-dev/looper/-/jobs/18165241641 looper-kick, https://gitlab.com/tacplus-dev/looper/-/jobs/18165240458 looper-plan, https://gitlab.com/tacplus-dev/looper/-/jobs/18165239856 tacplus#273); https://github.com/tacplus-dev/cluster/issues/37
- https://github.com/tacplus-dev/looper/blob/main/looper-recap ('LOOPER_RECAP_PAGES (10 pages of ...)': jobs?scope[]=success&scope[]=failed&scope[]=canceled&per_page=100&page=N for page in 1..10, and nothing past them)
- recap 2026-11-19 concurrency: waiting_hours 28.59; wait/tacplus #830 4.87 h (https://github.com/tacplus-dev/tacplus/issues/830: 'Backed-off: until 2026-11-23T09:00:00Z'), #273 4.11 h (waits until 12:00Z behind #327); wait/fleet-seatrials #56 4.87 h (https://github.com/tacplus-dev/fleet-seatrials/issues/56: 'waits until 2026-11-20T00:00:00Z... The hub has not yet answered'); wait/llmproxy #67 4.87 h (https://github.com/tacplus-dev/llmproxy/issues/67: 'until 2026-11-21T00:00:00Z' for a Debian digest); wait/looper #189 4.87 h (https://github.com/tacplus-dev/looper/issues/189: 'until 2026-11-19T11:00:00Z'); loopers.llmproxy.jobs 153 with reports max 0, fleet-seatrials 144 with 0, oauth 120 with 1; projects.tacplus-dev/looper run_s p50 28 s
- https://github.com/tacplus-dev/looper/issues/181 (closed 2026-11-17 18:53Z: the Backed-off line, 'the kicker, plan and --list don't treat it as ready again'); https://github.com/tacplus-dev/looper/commit/97af996 (#174: the control jobs' image pinned at f489822 of 2026-11-17 03:44Z, before #181)
- recap 2026-11-19 requests/report: cpu used p50 468m, p90 582m over 694 pods, against p50 130m, p90 144m over 401 pods on 2026-11-18 (suggestions/recap/2026-11-18.json)

## Cost
Hours in looper-recap: page until a job older than the window (and a line in the recap when a project's read was cut), the issues' Backed-off lines read beside the kicker's passes, a waits-by-choice table with each reason, and the no-op job count per issue; fixtures in tests/looper-recap.test.sh

## Touches
looper#57 and #72 (the recap's purpose), looper#107 (waits with no report job), looper#181 (back-off), the open 2026-11-18 suggestion (the image pin, which ends the kicker's no-op starts), decisions/looper-job-per-issue-2026-11-03.md (capacity decided on the recap's numbers), decisions/throughput-first-2026-11-11.md

## Reuse considered
looper-lib.sh's ready (backoff_over, #181) already reads the Backed-off line: the recap can call it rather than infer readiness from the kicker's lines. The failed-jobs read this scan gets (7 days, by finished_at) pages the same API and found the 57. Considered: a bigger LOOPER_RECAP_PAGES (the busiest nights are the ones that overflow); waiting for the image pin alone (it ends the no-op starts, not the page cap)

## Advice
Recap: 2026-11-19 wait/llmproxy
Cause: llmproxy#67 backed off until 2026-11-21 for a Debian digest (looper#181); the kicker in the pinned image lists it ready and starts 153 jobs that end in 28 s, and the recap, reading the kicker's lines and the newest 1000 jobs, counts 4.87 h of waiting and no failures
Change: tacplus-dev/looper looper-recap: page to the window's start; read the issue's Backed-off line (looper-lib.sh ready) and report waits by choice apart, with the no-op jobs each issue got; the control jobs' image pin (the 2026-11-18 suggestion) ends the no-op starts
Gain: no hours of waiting removed: 23.1 of the 28.59 h the recap calls waiting are five issues waiting by choice, so its advice goes to the 5.5 h that are not; the 57 failed jobs and the 23:53Z outage appear in the recap instead of 'failures: {}'

## Status
open
Rank: 5
Filed: 2026-11-19 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18176428236)
