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

## Open

- **Round-table** (issue-flow addendum, 2026-10-31: security-related mechanisms). The hub recommended one. Jody's
  "approved" didn't say whether it's waived, so the hub asked.
