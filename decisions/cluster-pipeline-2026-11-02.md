# The cluster pipeline is the only holder of cluster-admin (Jody, 2026-11-02)

Jody approved the design in tacplus-dev/roadmap#1 on 2026-11-02 ("approved"). It closes the "top pipeline" left open in
`deployments-per-cluster-2026-10-26.md` ("Open"). The pipeline is now on the critical path: a fleet has no rights on the
shared cluster until something creates its namespace and grants its account.

## Decided

- **tacplus-dev/cluster** owns everything cluster-scoped on `tacplus`. The code is on GitHub (open source); CI runs in the
  GitLab project of the same path, the fleet-seatrials pattern. That corrects the issue's "a GitLab project".
  - deploymentproxy: its namespace, CRDs, RBAC and Deployment, pinned to a release line.
  - `deployments.yaml`, one entry per deployment: the namespace `tac-<deployment>` labelled `tacplus.dev/deployment`,
    deploymentproxy's per-deployment Role, and the fleet deploy account's and `operator`'s namespace-scoped roles.
  - Cluster-scoped objects a fleet's chart needs, and only from here.
- **One identity:** the Google service account `cluster-admin-pipeline`, keyless, through Workload Identity Federation for
  this project's protected `main` only. Nothing else holds cluster-admin: no fleet pipeline, and not the hub.
- **Jody signs off on each apply.** `plan` runs on every change. `apply` is a protected manual job only Jody can play. A
  nightly drift check runs `plan` and files an issue; it never applies.
- **Deletes are explicit.** Removing an entry deletes nothing. Deleting a deployment's namespaces, and with them its tenant
  data, is a separate, named manual job that only Jody runs.
- **Never:** fleet workloads, letsencrypt, tenant data.
- **Alternatives rejected:**
  - cluster-admin per fleet pipeline (it breaks deploymentproxy's rule);
  - the hub's account (deploys run only from pipelines).

  A reconciler (Config Sync or Argo CD) stays a possible later step.

## Settled 2026-11-02

Jody: "I dont think we need a round-table for the cluster pipeline. what you said is what we agree upon. one pipeline
manages the cluster another manages each deployement in a 1 to 1 relationship"

- **No round-table** (waived by Jody).
- **Two kinds of pipeline:**
  - **One cluster pipeline** (tacplus-dev/cluster) manages the cluster and everything cluster-scoped.
  - **One pipeline per deployment, 1:1**, manages that deployment only, inside its own namespaces, with its own
    namespace-only account. For example, tacplus-dev/fleet-seatrials manages `seatrials`.
  - A deployment pipeline never touches the cluster or another deployment. The cluster pipeline never deploys a
    deployment's workloads.
