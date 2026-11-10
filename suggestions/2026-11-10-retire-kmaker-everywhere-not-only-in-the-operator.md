Suggestion: Retire kmaker everywhere, not only in the operator
Type: Cost
Summary: Jody retired kmaker on 2026-10-29 and operator#16 stopped rendering it on 2026-11-06 ('nothing calls it'). Yet kmaker#14 (pin the build) and #2 (release) are still approved priority:high, so the loopers keep working on a service no one calls. Its STATE.md and ROADMAP.md treat it as live, fleet-seatrials' fleet.lock still pins it, and the platform ROADMAP lists it in seatrials. Close its issues, unpin it from the fleet, and archive the repository
Why: Since the retirement the loopers have closed kmaker#12 (trace_id on MintKernelCert, 2026-11-05) and #13 (floor 2.6, 2026-11-06), and opened #14 on 2026-11-06 with 7 comments since: model runs, gates and VM slots on code that nothing calls, while the shared pool's reports waited 421.8 h for a VM tonight. #14 and #2 also keep kmaker in the build pipeline's pin list and its security floor. An archived repository drops out of the kicker's search (gh repo list --no-archived) and this scan's. Also serves Documentation: kmaker's STATE ('deployed nowhere yet ... Blocked on: the release (#2)') and ROADMAP (floor 2.4, 'the operator renders kmaker's config') contradict the operator's STATE ('kmaker is retired (#16): no kmaker workload, config or RBAC').

## Evidence
- https://github.com/tacplus-dev/operator/issues/16 (closed 2026-11-06: 'Retire kmaker (Jody, 2026-10-29). tacplus-dev/kernel#29 found nothing calls Kmaker.MintKernelCert'; its objects and the extra signing CA private key removed)
- https://github.com/tacplus-dev/kmaker/issues/14 (open, approved, priority:high, opened 2026-11-06: pin the build) and https://github.com/tacplus-dev/kmaker/issues/2 (open, approved, priority:high: release); https://github.com/tacplus-dev/kmaker/issues/12 and https://github.com/tacplus-dev/kmaker/issues/13 closed 2026-11-05 and 2026-11-06, after the retirement
- https://github.com/tacplus-dev/kmaker/blob/main/STATE.md ('deployed nowhere yet ... Blocked on: the release (#2)'); https://github.com/tacplus-dev/operator/blob/main/STATE.md ('kmaker is retired (#16)')
- https://github.com/tacplus-dev/fleet-seatrials/blob/main/fleet.lock (services: kmaker, id 49, commit f1ab8f33); https://github.com/tacplus-dev/roadmap/blob/main/ROADMAP.md (Now: seatrials runs '... letsencrypt, kmaker, llmproxy ...')

## Cost
An hour of the hub's time at Jody's word: close kmaker#2 and #14 as 'retired' (not done), drop kmaker from fleet-seatrials' fleet.lock and its uni-config in a fleet change (the operator no longer renders it, so nothing deployed changes), note it in kmaker's STATE.md and README, and archive tacplus-dev/kmaker. Saves the model runs and VM slots of every future kmaker issue and its nightly scan.

## Touches
operator#16 and kernel#29 (the retirement), decisions/deterministic-deployments-2026-11-02.md (fleet.lock as the deployed truth), the pin-every-repository suggestion (accepted, 2026-11-06), decisions/secrets-by-pipeline-2026-11-05.md (one key fewer: the kmaker signing CA is already gone)

## Reuse considered
Archiving is GitHub's own, and the kicker, the plan and this scan already list repositories with --no-archived, so nothing in the looper changes. Considered: removing the approved label only, but the nightly scan would keep proposing work and the repository would stay in the pin list. SeaBeam's retired sources (ports.yaml, looper#5) were retired the same way, never ported.

## Status
open
Rank: 5
Filed: 2026-11-10 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17872762621)
