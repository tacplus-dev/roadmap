Suggestion: Plan conflicts by file, not by directory
Type: Performance
Summary: The parent's plan treats a 'touches' directory as conflicting with everything under it; have the plan name files, and hold two issues back only for a shared file, contract or named semantic conflict
Why: Conflict holds were the night's largest wait: 155.63 issue-hours in tacplus, plus 7.19 in operator and 4.74 in looper. Several of tacplus's overlaps are only a shared directory ('both touch cross-test', 'both touch cross-test/accounting-identity', 'both touch cross-test/current-generation'). Issues that change different files of cross-test could run side by side: the lander already rebases and re-runs the gates, so a true overlap still fails safe. This is exactly Jody's 'much of this can be done in parallel' (looper#57).

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17683550072 (recap 2026-11-03, held/tacplus/conflict: 38 holds of 17 issues, up to 6.01 h, 155.63 h in all; overlaps include 'both touch cross-test' (#264), 'both touch cross-test/accounting-identity' (#273), 'both touch cross-test/current-generation' (#196), each held 7.47 h by conflict)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17666176518 (recap 2026-11-03: tacplus slots capacity 5, peak 5, mean use 0.5 while 17 issues were held by conflict)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17681695801 (recap 2026-11-03, held/operator/conflict: 10 holds, 7.19 h; held/looper/conflict: 6 holds, 4.74 h, https://gitlab.com/tacplus-dev/looper/-/jobs/17685557203)
- https://github.com/tacplus-dev/looper/blob/main/README.md (the parent's plan: 'Two issues that touch the same path (or one under the other) ... never run at the same time')

## Cost
A day or two in repo-looper: the plan prompt asks for files, with a directory allowed only for generated output it can't name file by file. The conflict rule compares files, not prefixes. A directory 'touches' over N files counts as unknown and runs alone, as today. Tests in tests/repo-looper.test.sh: two issues in one directory with disjoint files run together; the same file is held.

## Touches
looper#32 (parallel children), looper#39 (rank between conflicts), looper#57 (the recap), looper#48 (gate scaling: same spirit, smaller units), the rule that broken work never lands (the lander's rebase and gates stay the safety net)

## Reuse considered
The plan already writes 'touches', 'contract' and 'conflict' lines, and the README's 'The plan's areas, kept' proposal (#32) would record each child's 'git diff --name-only' against its predicted areas: the data to tune this. The proposal 'Release a conflicting issue once the land-pending one has landed' (#39) is complementary: it shortens real conflicts, this removes false ones. Neither is an open issue.

## Advice
Recap: 2026-11-03 held/tacplus/conflict
Cause: the plan's conflict rule counts a path and anything under it as the same area, so issues touching different files of cross-test/ or a generated tree are serialized
Change: repo-looper's plan (prompt and conflict check): file-level 'touches', directory prefixes only for generated outputs; a coarse directory claim treated as 'unknown areas' (runs alone) rather than blocking every issue under it
Gain: about 15 issue-hours a night from the directory-only overlaps seen tonight (#264 and #273, 7.47 h each), more as #196's and the generated-output overlaps are split; the ceiling is the 167.56 h of conflict holds in tacplus, operator and looper

## Status
open
Rank: 3
Filed: 2026-11-03 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17692416595)
