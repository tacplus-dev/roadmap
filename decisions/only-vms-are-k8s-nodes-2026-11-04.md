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

## Addendum: CI jobs are GitLab executor pods (Jody, 2026-11-04)

Jody: "if GitLab supports something, then we should probably be utilizing the GitLab way of doing things rather than
inventing our own." GitLab's Kubernetes executor runs each CI job as a pod (not a Kubernetes Job), so:
- every CI job, builds included, is a pod the GitLab Kubernetes executor creates: builds on the spot pool, control work
  (kicker, reports, managers) on the normal pool;
- spot preemption is handled the GitLab way: `retry:` (`runner_system_failure`, `stuck_or_timeout_failure`) in the
  pipeline files; time limits by `timeout:`; placement and requests by the runner's settings and per-job overrides;
- no build submitter and no Kubernetes Job manifests of our own.

## Addendum: a new cluster in seabeam.net, then the old one goes (Jody, 2026-11-04)

Jody: "I dont think we need the cluster running on thescottsweb.com org anymore. AFAIK we are not using it. We should
bring up the new cluster in seabeam.net and then transistion to it."

- The `tacplus` cluster in jody1-328023 runs only deploymentproxy and an empty `tac-seatrials` namespace (tacplus.dev
  is not live), so nothing migrates but the pipelines' targets.
- The new cluster is tacplus-dev/cloud's `dev` stack, in the folder `tacplus-dev` of seabeam.net, with cloud#7's pools
  (normal 3 x c4a-standard-2, spot build pool 0 to 3). tacplus-dev/cluster and the fleets retarget to it; the
  in-cluster runner (cluster#6) lands there.
- Then the old cluster is deleted (Jody: a deletion), and the rest of jody1-328023 (runner VMs, DNS zones, the
  looper's KMS key) follows as each moves.

## Addendum, 2026-11-07: where loopers and builds run (Jody)

"Every single looper should be running on one of the 3 nodes that is always in existence and they should all run at the same time. Any starvation they have is temporary just as if they were running on my local mac. The builders run on spot instances. And we dont care too much about how long they take. The loopers just move on to something else. If a build failure occurs that is picked up again by a looper. ... loopers are never blocked. We start as many as we need."

- **Agents (kicker, plan, reports, managers):** pods on the always-on normal pool (3 × c4a-standard-2), all at once, each with a small request (they mostly wait on the model's API). There are no per-repository caps that keep a ready issue waiting.
- **Builds and gates:** build jobs on the spot build pool, asynchronous. A report hands its gates to a build job and moves on, and a red build is a looper's next work, never a block.
- **Today's gap, as tacplus-dev/looper#84 closes it:** reports run gates inline in 4-CPU/14 GB slots on runner VMs, and caps.yaml caps each repository at 3 (tacplus 6).
