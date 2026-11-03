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

## The network layer: Jody's two options (2026-11-03)
Jody: "What I want to do is either 1) maintain our own implementation for the network traffic RPC 2) use our blackbox gRPC
implementation for this and if 2 possibly hijack the session in code as opposed to using a local loopback"

1. **Our own implementation of Ratis's RPC layer, typed.** Ratis plugs in its transport (its gRPC and Netty transports are
   both plugins). Ours sends each Raft message (vote, append, install-snapshot) as a typed message in our protos.
   - For: every consensus message is traced and validated.
   - Against: more work, and we follow Ratis's plugin interface across upgrades.
2. **The black-box gRPC member port** (tacplus#259): Ratis's own bytes inside our authenticated bidi stream.
   - For: little work. Authentication, rotation and the close at expiry come with #259.
   - Against: the trace sees opaque bytes.
   - **In code, not loopback:** attach the member-port stream directly to Ratis's transport inside the process. Then there
     is no extra listening port in the pod, no double hop, and nothing outside the process to configure.

**The hub checks before the huddle:**
- how cleanly Ratis's transport accepts an in-process stream (a custom Netty channel, or a gRPC in-process channel);
- what its RPC plugin interface requires.

**The hub's lean, to test:** (2) in code first; (1) if consensus should be traced message by message.

## The same question for Hazelcast (Jody, 2026-11-03)
Jody: "sense we are using the hazelcast opensource product I am curious how difficult it is for us to hijack the network
flow and use our own system just like we are discussing here"

- **Today:**
  - #212's member port wraps Hazelcast **by loopback**: Hazelcast binds to 127.0.0.1 in the pod, and the member port relays to
    it (`cross-test/memberport/testdata/HazelcastMemberPortTest.java`).
  - #259 moves the member port onto gRPC, but keeps the relay unless we change it.
- **In code:**
  - Hazelcast builds member networking through its internal `NodeContext`, which creates the node's `Server`.
  - Hazelcast's own test suite runs clusters with no sockets through it (its mock network), so a full replacement is proven
    possible in open source.
  - Ours would implement `Server` over our gRPC member stream, with members found through the public `DiscoveryStrategy`
    plugin interface, backed by kernel's fleet health state.
- **Risk:** `NodeContext` and `Server` are **internal** (`com.hazelcast.internal.*`) and can change between versions. Pin the
  version, keep a contract test of the adapter, and treat every Hazelcast upgrade as a reviewed change.
- **Question for the huddle:** one in-code member transport over our gRPC stream, with an adapter per engine (Hazelcast
  through `NodeContext`, Ratis through its RPC plugin)? It would replace #212's loopback relay too. The hub checks the exact
  `NodeContext`/`Server` surface on the pinned Hazelcast version first.

## Facts checked before the huddle (hub, 2026-11-03)
- **Kernel already runs Apache Ratis.** `server/.../fencedlock/RatisFencedLock.java`: one embedded Raft server per kernel
  instance, `FencedLockStateMachine`, a fixed group id (`kernel.FencedLock`), Raft log in a storage dir, automatic
  snapshots. Ratis is in `server/pom.xml` (ratis-server, ratis-client, ratis-grpc, ratis-metrics-default).
- **The API is already ours:** `service FencedLock` in kernel.proto, S2S, `Acquire` (non-blocking, idempotent per
  `holder_id`) and `Release` (by token). **The fencing token is already "the Raft log index of the granting Acquire",
  strictly increasing.** Every other service reaches locks only through kernel's FencedLock surface.
- **The transport is Ratis's stock gRPC** (`SupportedRpcType.GRPC`, `GrpcConfigKeys.Server.setPort`). There's no TLS or
  token configuration in RatisFencedLock: **peer traffic is unauthenticated gRPC on its own port.** That's against the
  platform's rules (server TLS plus kernel tokens) and the real problem this huddle solves.
- Hazelcast: still to check, the exact `NodeContext`/`Server` surface on the pinned version.
