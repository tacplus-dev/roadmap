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
