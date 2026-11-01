# Best practice: black-box traffic stays inside a cluster; typed gRPC across clusters (Jody, 2026-10-29)

- **Inside one cluster, black-box (opaque) traffic is acceptable.** That's traffic tacplus can't see into, for example Hazelcast's
  member protocol in Stratum. It always goes through the generated **member port** (tacplus-dev/tacplus#212): server TLS plus framed,
  in-band-refreshed kernel tokens, checked in both directions, failing closed. No mTLS, and no plaintext.
- **Across clusters we very much prefer typed gRPC:** contracts declared in proto, generated clients and servers, with tenant routing,
  policy and audit applied as on every other call. That covers the replication capability (`scope: FLEET`), kernel-to-kernel calls
  (Kubernetes service-account tokens plus OPA), and redo's second-site link (`redo.RedoSite`, tacplus#204).
- **When a design wants black-box traffic across clusters,** treat it as a reason to define a typed contract instead. An exception
  needs Jody's decision, recorded here.
- Related: `tenant-routing-2026-10-26.md` (member traffic uses tokens, never mTLS), `items/flock-design-2026-10-28.md` (kernel without
  Hazelcast).

## Amended 2026-10-31: the cross-cluster limit is lifted (Jody)

Jody, 2026-10-31, in a huddle on Stratum across regions: "We are going to lift the black box traffic limitation rule because
delivering it over our gRPC implementation reduces much of the concern."

- Black-box member traffic **may cross clusters and regions** when it is carried by the generated **member port over gRPC**
  (tacplus-dev/tacplus#259): server TLS, kernel tokens for `member:<svc>` checked at every stream open, make-before-break rotation,
  same service and same tid only, fail fast. It is still never raw and never plaintext.
- What remains is **operational, not a security rule**: a stretched cluster's membership heartbeats and partition rebalancing over
  WAN latency. Stratum's design must state its timeouts and partition behaviour for cross-region membership.
- Context: Stratum across regions. Most volume is converged data (lock-free merge). Atomic data gets transactions, so the lock
  cost is amortized over many mutations, plus map and entity locks on random unique keys (no hot keys).
