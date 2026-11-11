Suggestion: A land whose only conflict is STATE.md lands its code and sends back only its STATE.md commit
Type: Performance
Summary: fleet-seatrials' manager sent 4 re-lands back to new report runs on STATE.md conflicts, 3 of them (#44, #46, #47) on STATE.md alone: each report edits STATE.md, and the first merge conflicts the rest. looper#105 unions PROPOSALS.md and HISTORY.md but leaves STATE.md sending branches back. Have reports put their STATE.md change in a last commit of its own, and on a STATE.md-only conflict have the manager land the rest and send back just that commit, as a docs-only re-land
Why: STATE.md is the one planning file every report edits (repo-looper rule 6: 'update STATE.md when your change changes what it states'), and it is not append-only, so #105's union merge cannot take it. With lands queued for hours (fleet-seatrials report to merge 816.3 min, 79.61 h waited, 3 nights over), sibling lands of one repository conflict there almost surely. Tonight fleet-seatrials#44, #46 and #47 went back on STATE.md alone and #42 on README.md and STATE.md, redone: #42, #44, #46. Each send-back is a new report job that queues for a VM (#46 725.4 min, #44 496.9 min tonight), reruns its gates, and lands again behind the others, to resolve a few lines of prose. The same happened to tacplus#322 (STATE.md alone, 2026-11-10) and stratum#32 (2026-11-08), and with PROPOSALS.md beside it to tacplus#300, #313 and #359 and tenantecho#10. Once #105 lands, STATE.md is the largest cause of send-backs left. It is a cause of work serialized that could land in parallel: disjoint code, one shared status file. Also serves Cost (report runs and VM slots).

## Evidence
- https://github.com/tacplus-dev/fleet-seatrials/issues/44 (manager job https://gitlab.com/tacplus-dev/looper/-/jobs/17924501757, 2026-11-11T09:26Z: 'sent looper/fleet-seatrials-44 back to its report: rebasing it on main conflicts in STATE.md'; #46 the same at 09:30Z, job https://gitlab.com/tacplus-dev/looper/-/jobs/17928384358; #47 at 09:25Z; #42 'README.md, STATE.md' on 2026-11-10, job https://gitlab.com/tacplus-dev/looper/-/jobs/17892430187)
- https://gitlab.com/tacplus-dev/looper/-/jobs/17892430187 (recap 2026-11-11: manager/fleet-seatrials/sent-back, 4 branches sent back on a conflict, resolved 0, redone #42, #44, #46; manager/fleet-seatrials/merge-time, report to merge p90 816.3 min, lands waited 79.61 h, 3 nights; capacity/vm: fleet-seatrials#46 725.4 min and #44 496.9 min for a VM)
- https://github.com/tacplus-dev/looper/issues/105 (Expected: 'it resolves those two files [PROPOSALS.md, HISTORY.md] with git's built-in union merge ... Every other file's conflict still sends the branch back, as today')
- https://github.com/tacplus-dev/tacplus/issues/322 (manager, 2026-11-10T04:33Z: sent back on 'STATE.md' alone; also https://github.com/tacplus-dev/stratum/issues/32, 2026-11-08, 'STATE.md', and https://github.com/tacplus-dev/looper/issues/112, 2026-11-09, 'README.md, STATE.md')

## Cost
A day in repo-looper. (1) The report's prompt (rule 6) and its checkpoint put the STATE.md change in a last commit of its own, touching only STATE.md. (2) In rebase_on_main, when every conflicting file is STATE.md (after #105's unions), the manager drops that commit, so main's STATE.md is kept, and lands the rest on its gates as today. It comments the dropped diff on the issue and leaves the issue approved for a docs-only re-land of STATE.md on main's text (minutes once the gate maps of looper#48 and tacplus#266 land). The issue closes after that. A conflict in any other file still sends the whole branch back. Tests: two lands that change different code and both STATE.md both merge, and the second's STATE.md comes back as its own land.

## Touches
decisions/repo-roadmap-files-2026-10-27.md (STATE.md overwritten to stay current, by loopers and agents), decisions/merge-train-2026-11-04.md (batches of disjoint lands), decisions/looper-job-per-issue-2026-11-03.md (the manager merges, never by hand), looper#105 (the append-only logs' union), looper#86 (the merge train: the same conflicts inside a batch), looper#48 (docs-only lands in minutes), Jody on looper#57: 'much of this can be done in parallel'

## Reuse considered
Considered: (1) union-merging STATE.md like #105: it is overwritten, not appended, so a union duplicates or contradicts its lines. (2) Keeping main's side and dropping the report's text silently: STATE.md would stop saying what the merge did. (3) The manager resolving by a model: the manager holds the merge identity and runs no model or repository code (looper#90). (4) Regenerating STATE.md from HISTORY.md: STATE.md is prose about gaps and blockers, not derivable. Dropping a commit that only touches STATE.md is a fixed rule like regenerating a mapped file, and the follow-up reuses the issue's own resume path (looper#20). Not in an open issue.

## Advice
Recap: 2026-11-11 manager/fleet-seatrials/sent-back
Cause: every report edits STATE.md (repo-looper rule 6) and the manager resolves only regenerated files and, after looper#105, PROPOSALS.md and HISTORY.md, so sibling lands that change disjoint code conflict on STATE.md and each goes back to a full new report run
Change: repo-looper: the report's STATE.md change as its own last commit (prompt rule 6); rebase_on_main: a STATE.md-only conflict drops that commit, lands the rest, comments the diff and re-queues STATE.md as a docs-only land
Gain: tonight 3 report re-runs in fleet-seatrials (#44, #46, #47; #42 also conflicted in README.md and still goes back) and their VM waits (725.4 and 496.9 min for #46 and #44), with their code landing on the first green verdict instead. Since 2026-11-07, 6 more whose only conflicts beside #105's logs were STATE.md: tacplus#300, #313, #322, #359, stratum#32, tenantecho#10

## Status
open
Rank: 3
Filed: 2026-11-11 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17928642546)
