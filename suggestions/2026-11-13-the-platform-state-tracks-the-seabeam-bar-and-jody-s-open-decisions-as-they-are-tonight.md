Suggestion: The platform STATE tracks the SeaBeam bar and Jody's open decisions as they are tonight
Type: Documentation
Summary: The bar decision says the daily report tracks progress against A, B and C, but the platform STATE.md (last updated 2026-11-11) has no line for the bar. Its 'Blocked on (Jody)' lists 4 needs-decision issues against 13 open tonight, among them cloud#34 (llmproxy's Anthropic workload identity) and fleet-seatrials#46. It says tacplus v2.6.6 (v2.6.11 is out), and with ROADMAP.md's Now it still says reports run 'up to each repository's cap', which looper#107 removed on 2026-11-08
Why: Jody reads the record daily to decide, and since 2026-11-12 the bar is the platform's top priority after throughput: 'The daily report tracks progress against A, B and C'. Nothing in STATE says where A (12 Release: issues, then the two pins), B (fleet-seatrials#4) and C (fleet-dev#3) stand. Its list of what waits on Jody holds 3 of the 13 issues that do (tacplus#145, #1, cluster#18) and misses the other 10: cloud#34, fleet-seatrials#46 and #48, looper#142 (syft's GO-2026-6905), opa#15 and #16, and tacplus#569-#572 (tonight's gosec decisions). Its Loopers line describes caps and 'the interim runner VMs', where the jobs now run on tacplus-ci's shared and build pools behind cloud's runner manager with no caps (the recap's runner_pools). This is the same kind of drift that the accepted 2026-11-10 suggestion fixed, two days on

## Evidence
- https://github.com/tacplus-dev/roadmap/blob/0f1387e/STATE.md ('Last updated: 2026-11-11'; Version 'v2.6.6'; Loopers 'up to each repository's cap ... interim runner VMs'; Blocked on (Jody): tacplus#145, tacplus#1, cluster#18, llmproxy#56) and https://github.com/tacplus-dev/roadmap/blob/0f1387e/ROADMAP.md#L19 ('up to each repository's cap')
- https://github.com/tacplus-dev/roadmap/blob/0f1387e/decisions/seabeam-can-start-2026-11-12.md ('The daily report tracks progress against A, B and C')
- https://github.com/tacplus-dev/looper/issues/107 (closed 2026-11-08: 'No per-repository caps: start a report for every ready issue'); https://github.com/tacplus-dev/tacplus/releases/tag/v2.6.11 (2026-11-13 00:17Z)
- https://github.com/search?q=org%3Atacplus-dev+is%3Aissue+is%3Aopen+label%3Aneeds-decision (13 open on 2026-11-13: tacplus#572, #571, #570, #569, opa#16, opa#15, looper#142, fleet-seatrials#48, #46, cloud#34, cluster#18, tacplus#145, tacplus#1)

## Cost
An hour, by an agent at Jody's request (ROADMAP.md is Jody's; STATE.md is overwritten by agents). STATE gets a 'SeaBeam bar' line (A: each Release: issue's state and the pins; B; C), the Blocked on list from the needs-decision label, the current tacplus line, and the Loopers line as it runs (no caps; tacplus-ci pools; the in-cluster runner next). ROADMAP's Now drops 'up to each repository's cap'

## Touches
decisions/seabeam-can-start-2026-11-12.md (the daily report tracks A, B, C), decisions/throughput-first-2026-11-11.md, decisions/repo-roadmap-files-2026-10-27.md (STATE current, overwritten), looper#107 (no caps; Jody 2026-11-07: loopers are never blocked), suggestions/2026-11-10-the-platform-roadmap-s-critical-path-and-state-as-they-stand.md (accepted; the same file two days later)

## Reuse considered
The accepted 2026-11-11 suggestion (looper#132) has each repository's nightly scan check a STATE.md's Blocked on against issue states, but it covers 'Blocked on' lines that name closed issues, not a list missing open needs-decision ones, and the roadmap's own STATE is written by agents at Jody's request rather than by a repository's scan. The recap (suggestions/recap/) carries the night's pipeline numbers but not the bar's progress. A line in the roadmap scan's own prompt to refresh the bar's state would make it daily; this suggestion is the first refresh

## Status
open
Rank: 4
Filed: 2026-11-13 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17999306134)
