Suggestion: cluster's plan and drift run as a read-only identity; cluster-admin only in the apply Jody plays
Type: Security
Summary: Every pipeline on cluster's main, so every looper land the manager merges and every nightly drift, runs as cluster-admin-pipeline, the one identity the decision makes cluster-admin on dev, though plan and drift only read and dry-run. The gates that admitted the change ran in tacplus-dev/build with no credential, so a model-written change to ci/plan.sh, ci/lib.sh or ci/drift.sh holds cluster-admin the moment it is on main. Give plan and drift a second keyless identity (cluster view plus the server-side dry-run RBAC its diff needs), bound per environment, and leave cluster-admin to the manual apply and delete-deployment alone
Why: The decision of 2026-11-02 lists it Open and cluster's PROPOSALS asked for it the same day (#1); no issue or ROADMAP line holds it 16 days later, while what it guards against has arrived: since looper#84 (2026-11-16) cluster's lands are model-written and merged by the manager on green gates (6 lands tonight, cluster#28 to #32 merged), and each push to main runs plan as cluster-admin. The gates check what a change renders, not what its scripts do with a credential, and looper#40 treats the text a model reads as untrusted. Also serves Operability: with a read-only identity a merge request pipeline could show the plan before the merge, as the same PROPOSALS entry says

## Evidence
- https://github.com/tacplus-dev/cluster/blob/main/PROPOSALS.md ('A read-only identity for plan and drift', 2026-11-02, #1: 'today they run as cluster-admin-pipeline, so every pipeline on main holds cluster-admin without anyone playing a job, though they only read and dry-run ... For the security round-table the decision (cluster-pipeline-2026-11-02.md, Open) still lists'; no issue, not in cluster's ROADMAP.md on 2026-11-18)
- https://github.com/tacplus-dev/roadmap/blob/main/decisions/cluster-pipeline-2026-11-02.md ('One identity: the Google service account cluster-admin-pipeline, keyless, through Workload Identity Federation for this project's protected main only'; 'plan runs on every change'; 'A nightly drift check runs plan')
- https://github.com/tacplus-dev/cluster/blob/main/README.md (the jobs table: plan on 'main (push, web, API)', drift on the nightly schedule, apply and the deletion job manual under the protected environment cluster, Jody only; What the hub does, step 4: 'a ClusterRoleBinding of cluster-admin to the User cluster-admin-pipeline@tacplus-dev.iam.gserviceaccount.com')
- recap 2026-11-18, managers.cluster: 6 lands, 5 merged by the manager on green gates (cluster#28, #29, #30, #31, #32; e.g. https://gitlab.com/tacplus-dev/looper/-/jobs/18100762525), each a push to cluster's main whose pipeline ran plan as cluster-admin; the lands' gates ran in tacplus-dev/build (security-cluster of 3024b9c, nightly)
- https://github.com/tacplus-dev/roadmap/blob/main/PRODUCTION.md ('No standing human or hub write access' and 'A security review of the production layout': the same identity question, deferred to go-live while dev takes model-written lands now)

## Cost
Days: tacplus-dev/cloud makes a second keyless service account (say cluster-reader-pipeline) with roles/container.clusterViewer and a WIF binding of cluster's protected main by environment (attribute.environment, as the test IdP's); tacplus-dev/cluster binds it to a ClusterRole of view plus the dry-run verbs kubectl diff --server-side needs, and ci/lib.sh takes the identity by job: plan and drift read-only, apply and delete-deployment cluster-admin under the protected environment cluster. The hub applies the cloud side; Jody plays the first apply

## Touches
decisions/cluster-pipeline-2026-11-02.md (its Open list: a read-only identity for plan and drift), decisions/cloud-pipeline-2026-11-02.md (identities per pipeline, people read only), the accepted 2026-11-04 suggestion 'Only the manager can write main' (the manager merges model-written code on green gates), PRODUCTION.md 'No standing human or hub write access' and the security review before go-live, looper#40 (prompt provenance), cluster#13

## Reuse considered
fleet-seatrials' test IdP already binds a WIF principal per environment (attribute.environment) and cloud's gitlab stack already makes the pipeline identities, so the second identity is one more of the same; cluster's apply already runs under the protected environment cluster, which is where cluster-admin stays; kubectl diff --server-side needs only the dry-run verbs, which a ClusterRole can grant. Considered: leaving it to the go-live security review (PRODUCTION.md), but the exposure is on dev today, at every land

## Status
open
Rank: 1
Filed: 2026-11-18 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18137420443)
