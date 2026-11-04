Suggestion: The deterministic security checks in the pipelines that apply infrastructure
Type: Security
Summary: Run looper#49's floor (OSV-Scanner, govulncheck -mode=binary on pinned tools, gitleaks, Trivy on the CI image, an enforced security-exceptions.yaml) as a job in cloud's, cluster's and fleet-seatrials' own GitLab pipelines, which tacplus-dev/build's checks never reach
Why: cloud, cluster and fleet-seatrials hold the platform's highest privilege: cloud-admin-pipeline (folders, projects, IAM), cluster-admin on the cluster, and seatrials' live deploy secrets. Outside production each applies by itself on main (decisions/deterministic-deployments-2026-11-02.md). They run their own GitLab pipelines, not tacplus-dev/build's, so looper#49's deterministic checks will never run on them. Today only the nightly agent scan sees their supply chain, and it keeps finding vulnerabilities: 23 known ones in cloud's gitlab provider (cloud#3), 31 reachable ones in seatrials' operator pin (fleet-seatrials#11), 13 vulnerable pins (fleet-seatrials#13), gitleaks' xz (fleet-seatrials#25). cloud's accepted exceptions expire on 2026-12-03 and nothing enforces the date (cloud STATE: 'No job here runs the scan yet'). With the pipelines about to go live (cloud#1, cluster#1, fleet-seatrials#17), the floor should be there before the first apply.

## Evidence
- https://github.com/tacplus-dev/cloud/blob/main/STATE.md (Known vulnerabilities, #3: accepted in security-exceptions.yaml until 2026-12-03; 'No job here runs the scan yet')
- https://github.com/tacplus-dev/cloud/blob/main/PROPOSALS.md ('Deterministic security checks in this pipeline', nightly scan 2026-11-03: 'cloud runs its own GitLab pipeline, not tacplus-dev/build's, so looper#49's checks never reach it'; 'Enforce the exceptions' expiry')
- https://github.com/tacplus-dev/cluster/blob/main/ci/gitlab-project.gitlab-ci.yml (stages render, validate, plan, apply, drift: no vulnerability or secrets scan) and https://github.com/tacplus-dev/cloud/blob/main/ci/gitlab-project.gitlab-ci.yml (lint, plan and apply per stack, drift: none either)
- https://github.com/tacplus-dev/fleet-seatrials/issues/11 (operator pin 3d0e4b7: 31 reachable vulnerabilities), https://github.com/tacplus-dev/fleet-seatrials/issues/13 (13 vulnerable exact pins), https://github.com/tacplus-dev/fleet-seatrials/issues/25 (pipeline gitleaks GO-2025-3922): all found by agent scans, none by a deterministic check
- https://github.com/tacplus-dev/looper/blob/main/STATE.md (#49: the checks have run only by hand, looper-scan local; they will run in tacplus-dev/build's child pipeline, for repositories built there)

## Cost
Hours per repository: a security job on every push and nightly in each ci/gitlab-project.gitlab-ci.yml, running looper-scan (or the same pinned tools through ci/tools.sh) on the tree, on the pinned binaries (govulncheck -mode=binary of OpenTofu, the providers, kubectl, GitLab Runner, gitleaks), and Trivy on the CI image. It fails a secret, and a HIGH/CRITICAL finding with a fix, unless an unexpired security-exceptions.yaml entry accepts it. The JSON report goes out as an artifact. Each repository's first run fails until its findings are fixed or accepted (cloud's are already accepted).

## Touches
looper#49 (the deterministic floor: SCA and SAST, fail on HIGH/CRITICAL, expiring exceptions), decisions/deterministic-deployments-2026-11-02.md (apply on main by itself), decisions/cloud-pipeline-2026-11-02.md, decisions/cluster-pipeline-2026-11-02.md, fleet-seatrials#17, ROADMAP Later 'Automatic security remediation' (it needs deterministic findings to act on)

## Reuse considered
looper-scan (in the looper image) is the policy and the tool set already written and tested, and looper-scan local runs it on a checkout. Each pipeline can call it as one job, or include a shared job file, rather than re-implement it. cloud's PROPOSALS entry is this for cloud alone; this takes it to all three apply pipelines with one shared job. fleet-seatrials' checks.sh already runs gitleaks, which covers its secrets but not its dependencies. Moving these repositories into tacplus-dev/build would cover them too, but their plan and apply jobs need their own protected identities, which is why they have pipelines of their own.

## Status
open
Rank: 2
Filed: 2026-11-04 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356)
