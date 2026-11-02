# nsproxy becomes deploymentproxy; chaos acts through it (Jody, 2026-11-02)

Jody, 2026-11-02: "in regard to the chaos agent. the design should follow the same as the nsproxy. in fact it might make
sense to extend nsproxy to cover this. we should also rename nsproxy. maybe deploymentproxy. So the changes happen via CRDs
that the deploymentproxy picks up and handles. It would only make changes to the requesting deployment never the other
deployments"

The hub raised two questions, and Jody answered "yes to both, write it up":

1. Node-level faults run in a separate privileged node agent, reachable only through deploymentproxy, not inside the proxy.
2. The chaos controller records the proxy's result in the event trace, so the hop through the Kubernetes API is the one
   step that is not gRPC.

## Decided

- **nsproxy is renamed deploymentproxy.** It is the one component per cluster that holds cluster privilege on a
  deployment's behalf. The rename happens before the first deploy, while nothing depends on the old names:
  - the repository `tacplus-dev/nsproxy` becomes `tacplus-dev/deploymentproxy`;
  - the API group `nsproxy.tacplus.dev` becomes `deploymentproxy.tacplus.dev`;
  - the labels `nsproxy.tacplus.dev/*` become `deploymentproxy.tacplus.dev/*`;
  - the image, the module path and the docs follow.

  A breaking change on `0.x` is allowed, as a new minor.
- **One front door, custom resources only.** A deployment asks for changes only by creating custom resources in its own
  namespace (`tac-<deployment>`): `TenantNamespace` as today, and the fault kinds chaos needs (named by the survey in
  tacplus-dev/chaos#1, for example `PodFault`, `NetworkFault`). deploymentproxy has no network listener, as nsproxy has none.
  RBAC decides who may create which kind, per deployment namespace, applied by the pipeline.
- **Only the requesting deployment.** The deployment is read from the namespace's `tacplus.dev/deployment` label, never
  from its name, exactly as in nsproxy. A request may target only that deployment's own namespaces: its deployment
  namespace and its tenant namespaces. Any other target is refused, and so is anything on the exclusion list (letsencrypt,
  secret material, `kube-system` and other system namespaces, other deployments). A refusal is a status condition with a
  reason, as `TenantNamespace` already does.
- **Privilege is split; the project is one.**
  - **The deploymentproxy controller** (cluster RBAC, no node privilege) handles namespaces and **pod faults** (kill,
    failure) through the Kubernetes API.
  - **The node agent** (a privileged DaemonSet, one per node, shipped by the same project) does what needs root on the node:
    network delay and loss, clock skew, stress. It has a closed, typed API with no exec and no generic runtime access. It
    takes instructions **only from the deploymentproxy controller**, never from a deployment. It re-checks the target
    itself: the pod is on its node and belongs to the requesting deployment.
  - The two never merge into one binary: cluster-wide namespace admin plus root on every node is a larger blast radius than
    either part needs.
- **Event trace.** The chaos controller in each deployment stays a tacplus service, and its schedule, inject, revert and
  refuse calls are tacplus gRPC, traced as chaos#1 requires. It turns each fault into a custom resource. deploymentproxy acts
  on it and writes the outcome to the resource's status (applied, reverted, refused with a reason). The controller records
  that outcome in the trace under the fault's tag. The controller-to-proxy hop through the Kubernetes API is the one step
  that is not gRPC. It is accepted, and recorded here as that deviation.
- **Auto-revert and bounds** stay as chaos#1 has them. Every fault has a duration; deploymentproxy reverts at expiry, and
  when the resource is deleted.

## What changes

- tacplus-dev/chaos#1: the "wedge" is the deploymentproxy node agent. The chaos repository keeps the survey and the
  controller. The controller gets no node privilege and no cluster RBAC beyond creating fault resources in its own
  deployment namespace.
- tacplus-dev/registry#6: `chaos-wedge` is not a chaos service any more. Register `chaos-controller` only.
- tacplus-dev/operator#3 and #18, `deployments-per-cluster-2026-10-26.md` item 4, and the fleet manifests: "nsproxy"
  reads "deploymentproxy". The earlier decision stands as amended by this one.

## Addendum 2026-11-02: how the node agent knows the controller

Jody chose (a) on deploymentproxy#10 ("I thought we already selected A"): no listener on the node agent. The controller writes a cluster-scoped per-node task resource. Only deploymentproxy's service account may write it (RBAC plus a ValidatingAdmissionPolicy), and each agent watches only its own node's tasks. Authentication, TLS and audit are the API server's.
