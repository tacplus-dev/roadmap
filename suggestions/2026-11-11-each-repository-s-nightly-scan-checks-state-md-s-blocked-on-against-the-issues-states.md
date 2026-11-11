Suggestion: Each repository's nightly scan checks STATE.md's Blocked on against the issues' states
Type: Documentation
Summary: Eight repositories' STATE.md 'Blocked on' lines name issues closed days or weeks ago: fleet-seatrials (looper#25, closed 2026-10-30; #18), oauth and pigeon (looper#12, closed 2026-10-29), messaging (tacplus#207), blobstore (tacplus#188), launcher (looper#106), coop and mailbox (tacplus#313). Add a scan item: a Blocked on naming a closed issue is a bug, filed approved with the line to rewrite
Why: STATE.md is the record agents and Jody read for 'what holds this up', and a stale line misleads. A stale STATE.md line is exactly what made this scan's first suggestion wrong (2026-11-03, 'Apply looper#25's narrowed credentials', rejected: 'the evidence was a stale STATE.md line'). fleet-seatrials' STATE.md, updated 2026-11-09, still says it is blocked on that same looper#25 and on #18, closed 2026-11-03. Jody accepted the same correction for the platform's own STATE.md yesterday (2026-11-10). Reports update STATE.md only when their own change changes it (repo-looper rule 6), so nothing updates a line when another repository's issue closes. The nightly scan of each repository already reads its open issues and files mechanical bugs as approved. Checking each '#N' and 'owner/repo#N' in the Blocked on line is one API call per reference. Also serves Operability: the platform ROADMAP and these suggestions read these lines to tell what is blocked.

## Evidence
- https://github.com/tacplus-dev/fleet-seatrials/blob/main/STATE.md ('Last updated: 2026-11-09 by the looper (#27)'; 'Blocked on: #18 (pipeline setup ...), tacplus-dev/looper#25 (the agents' GitLab token reaches this project's secrets), #1 (tacplus 2.4)': #18 closed 2026-11-03, looper#25 closed 2026-10-30)
- https://github.com/tacplus-dev/oauth/blob/main/STATE.md ('Blocked on: #5 on tacplus-dev/looper#12 and #1': looper#12 closed 2026-10-29, oauth#1 closed 2026-10-27) and https://github.com/tacplus-dev/pigeon/blob/main/STATE.md ('Blocked on: #2 on tacplus-dev/looper#12', updated 2026-11-08)
- https://github.com/tacplus-dev/messaging/blob/main/STATE.md ('#7 waits on tacplus-dev/tacplus#207', closed 2026-10-30), https://github.com/tacplus-dev/blobstore/blob/main/STATE.md ('tacplus-dev/tacplus#188 (gates in the looper)', closed 2026-11-06), https://github.com/tacplus-dev/launcher/blob/main/STATE.md ('tacplus-dev/looper#106 (the release workflow of #5, so v0.1.0)', closed 2026-11-08); coop's and mailbox's name tacplus#313, closed 2026-11-10
- https://github.com/tacplus-dev/roadmap/blob/main/suggestions/2026-11-03-apply-looper-25-s-narrowed-credentials-on-the-live-pipeline.md (rejected 2026-11-03: 'the evidence was a stale STATE.md line, corrected in looper cce1a9c')

## Cost
Hours in repo-looper's scan prompt: a new item, 'STATE.md's Blocked on: each issue it names (#N, owner/repo#N), its state (gh api repos/<owner>/<repo>/issues/N). One that is closed is a bug, one issue for the repository, filed approved (a mechanical rewrite of the line: drop it, or name what blocks now)'. Or deterministically: the scan wrapper lists the closed references into the run directory, as it does the security report. Then 8 small docs-only lands, minutes each once gate maps land (looper#48).

## Touches
decisions/repo-roadmap-files-2026-10-27.md (STATE.md 'overwritten to stay current, by loopers and agents'), decisions/issue-flow-2026-10-27.md (a gap between the docs and the facts is a bug; mechanical ones approved), looper#13 (planning-files-check), suggestions accepted 2026-11-10 (the platform's STATE.md as it stands)

## Reuse considered
Considered: (1) planning-files-check failing on a closed reference: it would make every repository's gates depend on GitHub and on the day, against 'every gate hermetic' (looper#103), and turn a land red for someone else's close. (2) A report updating the blocked repository's STATE.md when it closes its issue: it holds only its own repository. (3) The issue's 'Blocked-by:' lines already make the looper resume an issue when its blockers close, but STATE.md is prose that no looper rewrites. The scan already reads open issues and files approved mechanical bugs, so it is the cheapest place. Not in an open issue.

## Status
accepted (Jody, 2026-11-11): tacplus-dev/looper#132
Rank: 4
Filed: 2026-11-11 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17928642546)
