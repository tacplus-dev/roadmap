# Fenced lock: consensus behind kernel's FencedLock without Hazelcast (huddle agenda)

- **Status:** agenda for a deep-dive huddle on 2026-11-03 (Jody). Nothing is decided beyond the points marked Jody.
- **Owner:** Jody

## Jody so far (2026-11-03)
- "we might need to create our own fenced lock implementation based on org.apache.ratis but not using org.apache.ratis. the
  reason is for us to use our grpc pattern for establishing the cluster between the nodes."
- "I really dont intend to rewrite kernel in rust any time soon. I am actually fine with continuing to use apache, etc. so the
  server will stay in java. the client needs to be for our languages and we want to hide everything behind our proto
  schema. I believe we already do much of this anyways where we define the fenced lock in proto"

So the server side is **Java, in kernel**. Apache libraries are acceptable. Clients are generated for every language, and
everything sits behind the proto schema.

## Facts to start from (the hub checks these before the huddle)
- `FencedLock` is already declared in `kernel.proto` (around line 535: `FencedLock.key` is "opaque to kernel; callers
  partition the namespace themselves"). Services reach it through generated clients.
- Hazelcast comes out of kernel (`decisions/platform-direction-2026-10-23.md`; flock revision, 2026-10-28). Stratum keeps it.
  So kernel's locks need a new backing.
- Black-box member traffic moves to typed gRPC (`decisions/black-box-traffic-2026-10-29.md`, tacplus#259).
- Kernel stays within its cluster (flock revision, 2026-10-28), so a consensus group is per cluster.
- Stratum's transaction shape (one request takes the fenced lock, applies, commits; check again under lock) consumes
  fencing tokens (`decisions/fleets-and-canary-huddle-2026-10-22.md`, `items/stratum-transactions.md`).

## Questions for the huddle
1. **Ratis as a library**, with its RPC layer replaced by our generated gRPC member transport (Ratis has a pluggable RPC
   layer; the hub verifies how pluggable). Or **our own Raft modelled on Ratis**, in Java?
2. **The fencing token:** Raft term plus log index, monotonic. Is it returned by `FencedLock` today, and does every protected
   write check it?
3. **Cluster formation and auth:** members find each other through kernel's fleet health state. They authenticate with
   server TLS plus kernel tokens (`member:<svc>`), with no mTLS, and close at token expiry (#260).
4. **Storage:** our redo/diskstore for the Raft log and snapshots, or Ratis's own?
5. **Verification:**
   - a TLA+ model of the fencing and lease additions;
   - deterministic simulation tests;
   - Jepsen-style linearizability checks under our chaos faults.
6. **The round-table** this needs, under the 2026-10-31 issue-flow addendum: it's a security- and correctness-critical
   mechanism.
