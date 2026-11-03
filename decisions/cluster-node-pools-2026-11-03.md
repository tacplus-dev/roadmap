# Cluster node pools: 3 normal nodes; everything else dynamic and spot; builds only on spot (Jody, 2026-11-03)

Jody: "the loopers should just run in our k8s cluster" ... "Then we only use vms for builds and builds are always spot"
... "we can actually do the builds in the cluster to but use spot nodes as part of a dynamic pool. I think our cluster
should have a few normal nodes and then dynamic spot nodes" ... "I dont know how our cluster stays up if its all spot
nodes??" ... "I think we have 3 non-spot small arm nodes. then the other nodes are dynamic and spot. they are also labeled
and we dont run builds on our non-spot. but loopers do run on non-spot". Budget: "I just need to keep spend to 400 or
less a month".

## Decided
- **Normal pool:** 3 non-spot small Arm nodes (c4a-standard-2). They run the platform (kernel, gateway, stratum and the
  rest) **and the loopers** (agent and control jobs).
- **Dynamic pools:** spot, autoscaling from 0, **labelled and tainted** (`spot`), so only pods that tolerate it land
  there.
- **Builds run only on spot nodes, never on the normal pool,** as build pods (rootless BuildKit, the shared caches).
- **No runner VMs** once loopers and builds run in the cluster: the GitLab Kubernetes runner. Until then, the runner VM
  pools are spot (templates `runner-arm64-t2a8-v7-spot`, `runner-arm64-t2a16-v8-spot`).
- **Looper isolation on the shared normal nodes:** their own namespace, NetworkPolicy (no traffic to other namespaces;
  outbound only to the forges and the model API), no service-account token, no cloud identity, non-root, no container
  runtime. The shared-kernel risk is accepted for now; the sandboxed runtime is on PRODUCTION.md.
- **Cost:** the 3 normal nodes are the fixed part (about $195/month); spot only while working; under $400 in total.

## Supersedes
- cloud#5's VM pools (a shared always-on VM, builder VMs): replaced by cluster pools.
- The earlier "agents on a separate node pool" in the hub's proposal: Jody puts loopers on the normal nodes.

## Done by hand, 2026-11-03 (the cloud pipeline adopts it later)
- spot-pool: 3 x c4a-standard-2 spot, to become the dynamic pool;
- default-pool deleted, by the hub's mistake of making the cluster all-spot;
- normal-pool: 3 x c4a-standard-2 on-demand, recreated, limited by the IN_USE_ADDRESSES quota of 8 until the old pool's
  IPs free.
