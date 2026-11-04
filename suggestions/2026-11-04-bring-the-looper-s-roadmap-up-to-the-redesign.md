Suggestion: Bring the looper's roadmap up to the redesign
Type: Documentation
Summary: looper's ROADMAP.md still says the kicker runs on the Mac and the image is the 2.4 build environment, and its Next lists 10 closed issues, among them #32's parent-and-children design that #70 retired; the platform ROADMAP still says one looper per repository. Move the done items to Now and mark #32 retired
Why: The loopers build a feature only when a ROADMAP.md links it, and they and the scans read the roadmaps before they open an issue (looper#14). A Next that still lists 'Parent looper runs child loopers in parallel; the parent decides concurrency (#32)' tells them a retired design is still wanted, a day after decisions/looper-job-per-issue-2026-11-03.md replaced it. Jody's review reads the same file. Only an agent at Jody's request may change a ROADMAP.md, so it waits on this review. Also serves Correctness: the roadmap matches what runs.

## Evidence
- https://github.com/tacplus-dev/looper/blob/main/ROADMAP.md (Now: 'looper-dispatch on the Mac starts the GitLab pipeline' and 'the image is the tacplus 2.4 build environment (#1)'; Next: #13, #24, #31, #32, #40, #55, #57, #60, #68 and #70, all closed, read 2026-11-04)
- https://github.com/tacplus-dev/looper/blob/main/STATE.md (2026-11-04: the kicker runs as the GitLab pipeline's schedule, 'nothing on the Mac' (#24); the image is built FROM build:2.5; the parent and its in-job children (#32), conflict holds (#39) and in-job lander are retired by #70)
- https://github.com/tacplus-dev/looper/issues/32 (closed 2026-10-30; superseded by https://github.com/tacplus-dev/looper/issues/70, closed 2026-11-03)
- https://github.com/tacplus-dev/roadmap/blob/main/ROADMAP.md (Now: 'one issue-driven looper per repository, working approved issues'; decisions/looper-job-per-issue-2026-11-03.md: one job per issue and a manager per repository)

## Cost
Under an hour for an agent at Jody's request: looper ROADMAP.md's Now rewritten from STATE.md (kicker on GitLab, a job per issue with a manager per repository, image FROM build:2.5), its closed Next items moved to Now as done with their dates, #32 marked retired by #70; one line in the platform ROADMAP's Now.

## Touches
decisions/repo-roadmap-files-2026-10-27.md (ROADMAP is Jody's, STATE the loopers'), decisions/looper-job-per-issue-2026-11-03.md, looper#14 (features only when a ROADMAP.md links them), looper#13 (the planning files)

## Reuse considered
STATE.md already says what is done and true, so the edit is a copy from it, not new writing. planning-files-check (#13) checks only that the four files exist, not that a closed issue left Next. A check that lists a ROADMAP's Next items whose issues are closed (one gh call each) could go in the nightly scan, so this doesn't recur; the scan could also propose the edit in PROPOSALS.md, since it may not make it itself.

## Status
open
Rank: 5
Filed: 2026-11-04 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356)
