Suggestion: The security report a scan reads, and the issues it files from it, are of main's latest build, not of any land's
Type: Security
Summary: Seven of tonight's ten service security reports describe a commit that is not on main: gateway 5aab7fe (its 23 'new' cargo-deny findings), oauth 07e397c, llmproxy acc06ff, mailbox 278a8eb, tacplus 8051b3c (320 'new'), eventbus 2e877a5 and opa 896f975 (the looper/eventbus-2 and looper/opa-2 branches). security-report publishes every build's findings as the repository's 'latest', a red land's branch included, and the nightly scan files bugs from it and compares the next run with it. Publish 'latest' from builds of main only, and compare a land's report with main's last
Why: Five repositories' nightly scans have said so since 2026-11-12: llmproxy's three nights running (main's Dockerfile has no USER, and the report of branch b998d87 had no such finding, so main's checks went unreported), pigeon's (a report of 3ec8f8d, an unlanded commit whose land went red), eventbus's (0 findings, of looper/eventbus-28, not main). The baseline (looper#146) and one-issue-per-cause (looper#129) both rest on 'new since the previous run', which across branches is noise: 23 new in gateway and 320 in tacplus tonight are a branch against another branch. And a branch's clean report hides main's own regressions from the scan that files them. Also serves Correctness: the report names what it checked.

## Evidence
- https://github.com/tacplus-dev/looper/blob/main/README.md (Security checks: security-report 'publishes its findings as the repository's latest: the generic package security-<repo>, version latest, of tacplus-dev/build ... The next run compares with it, and the nightly agent scan reads it'; 'The nightly agent scan reads the repository's latest findings')
- https://github.com/tacplus-dev/gateway/compare/5aab7fe3a2badd8cbe273e38a58b619d5e7a6b6d...main (diverged: tonight's gateway report, 35 findings, 23 new, is of a commit not on main); likewise oauth 07e397cf, llmproxy acc06ff1, mailbox 278a8eb1 and tacplus 8051b3c1 (https://github.com/tacplus-dev/tacplus/compare/8051b3c17478e697bccb63d4e51a8751236635db...main: 1455 findings, 320 new, of the looper/tacplus-340 land)
- https://github.com/tacplus-dev/eventbus/issues/2#issuecomment-6891008011 (2e877a5, tonight's eventbus report, is the looper/eventbus-2 branch; its build https://gitlab.com/tacplus-dev/build/-/pipelines/3069912458) and https://github.com/tacplus-dev/opa/issues/2 (896f975 is looper/opa-2, rebased on main ffb03b2)
- https://github.com/tacplus-dev/llmproxy/blob/main/PROPOSALS.md (nightly scan 2026-11-12 and 2026-11-13: 'the report the nightly scan was handed is of b998d87, the head of the looper/llmproxy-25 branch, not of main ... That is why it has no semgrep missing-user-entrypoint finding even though main's Dockerfile still has no USER. The scan should read the report of main's latest build')
- https://github.com/tacplus-dev/pigeon/blob/main/PROPOSALS.md ('The nightly scan should get the security report of main's build, not of the latest land's', 2026-11-13: a report of 3ec8f8d, the unlanded looper/pigeon-2 commit) and https://github.com/tacplus-dev/eventbus/blob/main/PROPOSALS.md (2026-11-13: '0 finding(s), but it is for 6bfca94, the unmerged looper/eventbus-28 branch, not main')

## Cost
A day in tacplus-dev/looper: looper-scan report publishes security-<repo> version latest only from a build of main (the nightly, or a land's merge once it is on main) and a build of a branch as security-<repo> version <sha>; a land's 'new since the previous run' compares with main's latest; the summary names the commit's ref; the scan's run directory gets main's. Tests in looper-scan's: a branch build does not move latest, and a land's report compares against main.

## Touches
looper#49 (the deterministic floor), looper#129 (one issue per cause), looper#146 (the recorded baseline, Jody on tacplus#585), looper#108 (generated code scanned at its source), decisions/issue-flow-2026-10-27.md (a bug filed from evidence of main), decisions/throughput-first-2026-11-11.md

## Reuse considered
tacplus-dev/build's generic package registry already versions security-<repo> (latest); a version per commit is the same mechanism, and the previous run's package the comparison already fetches. The gates job's commit status names the pipeline, so main's last build is the status of main's head (looper#113 notes triggered pipelines set none). The recap reads tacplus-dev/build's jobs with the bot's read token (looper#85) and could pick main's build the same way. Considered: the scan rebuilding main itself (a 3 h tacplus build a night, for a report the pipeline already makes).

## Status
open
Rank: 1
Filed: 2026-11-15 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18042630451)
