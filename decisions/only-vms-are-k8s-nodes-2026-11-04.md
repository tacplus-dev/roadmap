# The only VMs are Kubernetes nodes (Jody, 2026-11-04)

Jody, 2026-11-04: "I realize Kubernetes has nodes, and those nodes are running on VMs, but those are the only VMs that
we're going to have. So I want to make sure that everything you're talking about understands that those VMs will be
nodes within Kubernetes only."

Supersedes, where they speak of VMs outside the cluster:
- cluster-node-pools-2026-11-03.md ("we only use vms for builds"): builds are pods on the spot node pool;
- builds-in-containers-2026-11-03.md ("the VM is vanilla", COS VMs): the vanilla host is the GKE node (COS);
- looper-job-per-issue-2026-11-03.md ("containers share VMs", "one always-on VM"): pods share nodes; the always-on
  capacity is the normal pool's 3 nodes.

## Decided
- **No standalone VMs** in any design, stack or plan: no runner VMs, no runner manager VM, no builder VMs, no MIGs.
  The VMs are GKE nodes, and only in two kinds of pool: the normal pool (3 small on-demand Arm nodes: the platform,
  the loopers, the GitLab runner) and the spot pools (dynamic from 0, labelled and tainted: builds).
- **CI runs in the cluster:** the GitLab Kubernetes executor. Every job (kicker, looper reports and managers, scans,
  builds) is a pod with resource requests and limits; the scheduler packs pods onto nodes. Capacity is the node pools'
  size, which stays manual (scale-up on Jody's yes, with numbers).
- **Builds are pods on spot nodes,** rootless BuildKit in the pod, caches in the ci project's bucket and registry.
- **Language:** "capacity" means node-pool nodes and pod requests, never "a VM per job".

## Consequences
- The jody1-328023 runner VMs, `gitlab-runner-mgr` and their MIGs are interim; they go when the in-cluster runner
  takes over. tacplus-dev/cloud#7's runner-VM questions (manager-to-worker SSH, the manager's spot/OS) and #9 (its
  Ubuntu image) become "removed with the runner VMs".
- Pending work that adds or tunes runner VMs (e.g. packing jobs per VM) is replaced by the in-cluster runner with pod
  requests.
