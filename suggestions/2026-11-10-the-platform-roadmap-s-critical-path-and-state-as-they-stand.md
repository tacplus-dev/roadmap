Suggestion: The platform ROADMAP's critical path and STATE as they stand
Type: Documentation
Summary: The platform ROADMAP's Next 1 still says tacplus#163 'needs Jody's decision on D1–D6' and lists #170 and stratum#2 ahead of redo#1, all three closed on 2026-10-28. It puts redo in the restore tenant, which Jody replaced on 2026-11-07 (redo is a /system service), and says releases wait on looper#12 (closed 2026-10-29). STATE.md, last updated 2026-10-27, says v2.4.1 (tacplus is at v2.6.5) and lists the same closed blockers
Why: These two files are Jody's one-page view of the platform, and they now point at the wrong step: the Stratum path's next open item is redo#1, the service whose shape Jody decided on 2026-11-07. The 'Blocked on (Jody)' list names two issues that closed two weeks ago, so a real blocker would not stand out. The 2026-11-04 suggestion (accepted, done in looper's ROADMAP) covered only the loopers' line. Nothing has touched the platform's Now, Next or STATE since. Also serves Operability: loopers and scans read these files as the platform's direction.

## Evidence
- https://github.com/tacplus-dev/roadmap/blob/main/ROADMAP.md (Next 1: 'tacplus-dev/tacplus#163 ... Needs Jody's decision on D1–D6'; 'tacplus-dev/redo#1: the redo service, in the reserved restore tenant (00000002)'; Next 2: 'All wait on the org secret tacplus-dev/looper#12'; Now: 'tacplus 2.4.x is the current line')
- https://github.com/tacplus-dev/tacplus/issues/163 and https://github.com/tacplus-dev/tacplus/issues/170 and https://github.com/tacplus-dev/stratum/issues/2 (closed 2026-10-28); https://github.com/tacplus-dev/looper/issues/12 (closed 2026-10-29)
- https://github.com/tacplus-dev/redo/issues/1 (Jody, 2026-11-07, MEMBER: 'Decided: redo is a /system service, not tenant-owned and not in the restore tenant ... Drop pinned_tenant (restore)')
- https://github.com/tacplus-dev/roadmap/blob/main/STATE.md ('Last updated: 2026-10-27'; 'tacplus v2.4.1'; 'Blocked on (Jody): tacplus-dev/tacplus#163 ... tacplus-dev/looper#12'); https://github.com/tacplus-dev/tacplus/releases (v2.6.5, 2026-11-09)

## Cost
An hour, by an agent at Jody's request (ROADMAP.md is changed only that way): Next 1 starts at redo#1 as a /system service, then operator#2, people#1/kernel#2, letsencrypt#2, fleet-seatrials#3. Next 2 drops looper#12 and names what services' releases wait on now (gateway#6, the pins). Now gives 2.6 as the current line and the loopers as one job per issue. STATE.md is rewritten to today, with its blockers checked against open issues.

## Touches
decisions/repo-roadmap-files-2026-10-27.md (STATE overwritten to stay current), decisions/stratum-per-tenant-2026-10-25.md, decisions/products-layer-2026-11-08.md and redo#1's 2026-11-07 decision, suggestions/2026-11-04-bring-the-looper-s-roadmap-up-to-the-redesign.md (accepted; the looper half)

## Reuse considered
The 2026-11-04 suggestion's fix (an agent's pass at Jody's request) is the way, applied here to the platform's own files. Considered: having the nightly scan flag closed issues named as blockers in any ROADMAP.md or STATE.md, a cheap check, but it would be a looper-scan change of its own and is not proposed here.

## Status
accepted (Jody, 2026-11-10): tacplus-dev/roadmap#3
Rank: 4
Filed: 2026-11-10 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17872762621)
