Suggestion: A week before a security exception expires, the nightly scan files the decision it needs
Type: Operability
Summary: Dated exceptions keep eleven pipelines green and expire in waves nobody tracks: cluster's GO-2026-6859 in kubectl and gke-gcloud-auth-plugin on 2026-11-25 (no release of either with the fix), cloud's pins and stratum's shaded Jackson on 2026-12-03, fleet-seatrials' and cluster's on 12-04 ('every pipeline fails from then'), cluster's and cloud's on 12-05 and 12-06, looper's syft on 12-13, kernel's eight Jackson CVEs on 12-14. An expired entry turns the pipeline red the morning after; today a PROPOSALS line asks for 'a new dated decision'. Have each repository's nightly scan file a needs-decision bug seven days ahead, with the upstream's state, and the recap list the week's expiries
Why: The floor was built with expiring exceptions (looper#49) so a decision is revisited, not forgotten, but nothing revisits one before the gate goes red, and a red main stops every land, plan and apply of that repository (cloud#59). The first cliff is in ten days, in the pipeline that holds cluster-admin, and its fix has no upstream release, so the only good outcome is a decision made in time. The scan already reads STATE's Blocked on against the issues' states (looper#132) and looper-scan already parses the exceptions file with its dates. Also serves Security: a decision taken on the upstream's state, under a green pipeline, instead of a renewal typed under a red one.

## Evidence
- https://github.com/tacplus-dev/cluster/blob/main/STATE.md (Known vulnerabilities: 'GO-2026-6859 (net/http, Trivy's CVE-2026-97054) accepted until 2026-11-25 in kubectl v1.34.13 and gke-gcloud-auth-plugin 0.5.21 only, no release of either with the fix'; '10 accepted ... until 2026-12-04'; '14 accepted until 2026-12-05'; gh's 'until 2026-12-06') and https://github.com/tacplus-dev/cluster/blob/main/PROPOSALS.md ('GO-2026-6859 in kubectl and gke-gcloud-auth-plugin after 2026-11-25', 2026-11-13, #21: 'or a new dated decision')
- https://github.com/tacplus-dev/cloud/blob/main/STATE.md (pins 'accepted in security-exceptions.yaml until 2026-12-03'; gitleaks' xz 'until 2026-12-04'; brotli and GO-2026-6865 'until 2026-12-06') and https://github.com/tacplus-dev/fleet-seatrials/blob/main/STATE.md ('The security exceptions expire on 2026-12-04, and every pipeline fails from then'; Keycloak's four HIGH 'until 2026-12-06')
- https://github.com/tacplus-dev/kernel/blob/main/security-exceptions.yaml (nine entries, expires 2026-12-14: 'On expiry the gate goes red and the decision is revisited'), https://github.com/tacplus-dev/stratum/blob/main/STATE.md ('accepted as unreachable until 2026-12-03, waiting on a Hazelcast release'), https://github.com/tacplus-dev/looper/blob/main/STATE.md (syft's GO-2026-6905 'until 2026-12-13: from 2026-12-14 looper's builds fail on it')
- https://github.com/tacplus-dev/looper/blob/main/README.md (Security checks, Policy: 'an exception that has expired or is malformed' fails the run; 'expires: YYYY-MM-DD; from the day after, the build fails until it is renewed or fixed') and https://github.com/tacplus-dev/cloud/issues/59 (a red security job on main: 'every plan and apply is skipped')

## Cost
Hours in tacplus-dev/looper: a scan item that reads security-exceptions.yaml (and .trivyignore.yaml where a gate reads one) for entries expiring within seven days and files one bug per cause (approved; needs-decision when the fix is upstream with no release, Blocked-by the tracking issue, kernel#59's way), plus a recap line of the week's expiries across repositories. No pipeline change, no money.

## Touches
looper#49 (expiring exceptions), looper#132 (the scan checks STATE against the issues), looper#129 (one issue per cause), looper#57 (the recap), decisions/issue-flow-2026-10-27.md, kernel#59 and stratum#28 (decisions with expiries), cluster#21

## Reuse considered
looper-scan's exception parser (read_exceptions) and the scan's STATE-check item are the pieces; the recap's items (looper#57) are the place for the week's expiries. The repositories' own PROPOSALS lines ('Track and bump the pins', cloud#3; 'after 2026-11-25', cluster#21; 'Bump Hazelcast when it re-shades a fixed jackson', stratum#28) are what this automates. Considered: longer expiries (they defer the same cliff) and a calendar kept by hand (what fails today).

## Status
open
Rank: 5
Filed: 2026-11-15 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18042630451)
