Suggestion: The nightly scan files one issue per cause, not per finding, blocked from the start when the fix is upstream
Type: Performance
Summary: llmproxy's scan filed 22 approved issues (#34 to #55), one per cargo-deny 'unlicensed' crate, for a single cause, tacplus#368. Each became a report that waited about 15.6 h for a VM, only to add 'Blocked-by: tacplus#368, #390' and push nothing: 358.5 of the night's 953.9 VM-hours. Have the scan group findings by cause, and file an upstream cause with its Blocked-by lines already set
Why: Tonight 112 issues waited 953.93 h for a VM, and 37 of those waits (358.52 h, 38%) were llmproxy#34 to #55. They are 22 issues the nightly scan filed on 2026-11-10 (app/tacplus-looper, 08:48Z, all bug+approved), each naming one crate of the same generated Cargo.toml, with the same cause and the same fix: tacplus#368 and its 2.6 backport #390. Every report run ended the same way (llmproxy#34: 'Nothing to change here yet ... marked Blocked-by tacplus-dev/tacplus#368 and tacplus-dev/tacplus#390'). They crowded the shared pool's few slots: llmproxy had 24 reports at once, 73 jobs, and oauth, mailbox, messaging, cloud, operator and looper reports queued 938 min beside them. The scan's prompt asks for exactly this: 'File each finding ... as one issue (its tool, id and where in the title)' (repo-looper security_item). The same scan grouped blobstore's 213 generated-code findings into one issue (blobstore#17), so it is the rule, not the model, that splits them. Any other repository whose generated Rust chassis carries the same 22 crates gets the same burst from its first security report. Also serves Cost (22 model runs and VM slots a night for nothing) and Operability (one issue to watch and close instead of 22).

## Evidence
- https://github.com/tacplus-dev/llmproxy/issues/34 (filed 2026-11-10T08:48Z by app/tacplus-looper, bug+approved: 'cargo-deny HIGH: cargo-deny:unlicensed:am@0.1.0'. Its Cause section names tacplus-dev/tacplus#368. Its report, job https://gitlab.com/tacplus-dev/looper/-/jobs/17920823277, ended 'Nothing was pushed ... now Blocked-by: tacplus-dev/tacplus#368, Blocked-by: tacplus-dev/tacplus#390')
- https://github.com/tacplus-dev/llmproxy/issues?q=is%3Aopen+%22cargo-deny+HIGH%22 (22 open issues, #34 to #55, one per crate: am, approval, c2d, ... x509, all the same cause)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17892430272 (recap 2026-11-11, capacity/vm: 112 issues waited 953.93 h for a VM, queue p90 633.8 min. 37 of the waits were llmproxy#34 to #55, 358.52 h, e.g. llmproxy#55 938 min here and #54 938 min at https://gitlab.com/tacplus-dev/looper/-/jobs/17892430271. wait/llmproxy: 24 reports at once at most, 14 running, 73 llmproxy jobs. 76 of the 112 waits were on the tacplus-ci shared pool, 607.05 h)
- https://github.com/tacplus-dev/tacplus/issues/368 (open, approved: 'generated crates' Cargo.toml declare no license ... every Rust service's cargo-deny fails (22 crates)'; backport tacplus#390 open)
- https://github.com/tacplus-dev/looper/blob/53011292ebb197c35f72af5cb0b717929766cfb2/repo-looper#L2856-L2866 (security_item: 'File each finding that is new and not excepted and that the policy fails (blocking) or is HIGH or CRITICAL as one issue (its tool, id and where in the title)')
- https://github.com/tacplus-dev/blobstore/issues/17 (the same scan, one issue for 213 tacplus-generated findings: grouping works when the prompt allows it)

## Cost
Hours in repo-looper: security_item's rule becomes 'one issue per cause: findings that share a fix (the same upstream issue, the same generated template, the same dependency bump) are one issue listing them all, its title the cause'. A cause fixed in another repository's open issue is filed with its 'Blocked-by:' lines in the body from the start, so no report runs until that issue closes. A test in tests/repo-looper.test.sh: a stub report with 3 findings of one cause gives one issue. Closing llmproxy#35 to #55 as duplicates of #34 (with the list in #34) is a few minutes by hand or a report.

## Touches
decisions/issue-flow-2026-10-27.md (a finding is a bug; filed approved when mechanical), looper#49 (the deterministic security floor and its nightly filing), looper#107 (no caps: every ready issue gets a report at once, so a duplicate costs a VM slot), decisions/looper-job-per-issue-2026-11-03.md (one job per issue), tacplus#368/#390 (the cause), Jody on looper#57: 'I just want stuff to go faster'

## Reuse considered
The Blocked-by mechanism already holds an issue back until its blockers close (README, the outcomes table). The scan can write those lines when it files, as reports do when they find them. 'The other new ones, if any, as one issue that lists them' is already the rule for non-blocking findings (gateway#31, llmproxy#56), so the change extends an existing pattern. Considered: (1) a scale-up: rejected twice (only VMs are k8s nodes). (2) A cap on llmproxy's reports: never (Jody, 2026-11-07). (3) Dedup in the kicker: it would have to judge sameness from issue text, while the scan has the findings and their cause in hand. Not covered by an open issue: looper#108 is about the generated tree's SAST, not how issues are grouped.

## Advice
Recap: 2026-11-11 wait/llmproxy
Cause: the nightly scan's prompt (repo-looper security_item) files every blocking or HIGH finding as its own issue, so one cause in tacplus's Rust generator (tacplus#368) became 22 approved llmproxy issues, and with no caps each became its own report at once, only to find it Blocked-by tacplus#368
Change: repo-looper security_item: one issue per cause (the findings listed in it), filed with Blocked-by when the fix is another repository's open issue; close llmproxy#35 to #55 into #34
Gain: tonight 358.52 h of VM waiting (37 of 112 waits) and 22 model runs that pushed nothing; on later nights, the same for each Rust service whose security report carries tacplus#368's 22 crates, and room in the shared pool for the other 75 waits (17 repositories' reports, up to 938 min each) beside them

## Status
accepted (Jody, 2026-11-11): tacplus-dev/looper#129
Rank: 1
Filed: 2026-11-11 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17928642546)
