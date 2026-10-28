# Stratum: per tenant, in memory, no eventbus; the redo service is durability — DECIDED 2026-10-25

Huddle Jody + the kernel identity session, evening of 2026-10-25. Started from the identity work:
a `people` tenant need not live on the cluster that is minting, so the identity graph cannot be
kernel-local state, and "once Stratum is down this will all make more sense". Builds on
`platform-direction-2026-10-23.md` (Stratum is identity's durable, atomic record; Hazelcast stays in
Stratum) and the 10-21 identity huddle's note that the graph needs a Stratum above the clusters.
Amends `identity-session-credential-2026-10-25.md` where it says who holds the graph (below).

## Decided

1. **Stratum runs per tenant**, `people` included. Placement is config: within one cluster, or across
   clusters, with Stratum's black-box (Hazelcast member) traffic allowed between clusters when enabled.
   Because each tenant has its own Stratum, a tenant can move between clusters.
2. **Nothing is special in code.** `people` is treated as special by config only (for example, it is
   the tenant expected to span regions, and to read from local backups); every tenant runs the same
   Stratum.
3. **Records are written and read through Stratum, which has no dependency on eventbus.** The bus-to-databases fan-out
   (eventbus → stratum, timedb, graphdb) is a leftover; timedb and graphdb are already gone. Writes go
   to Stratum through its own API, which tacplus generates, and the answer is final (durable on the
   redo service, item 6). eventbus stays, for
   messaging between services; it is no one's database.
4. **Atomicity is per entry first.** Atomic entry operations (insert if absent, replace if unchanged)
   cover most invariants: the XID is a random string, so minting one is "insert if absent, retry on
   collision" — no map lock, and on first insert the entry does not exist yet. Uniqueness on another
   attribute is made per-entry by keying a row by that attribute (a login's link row is keyed by
   `(idp_id, subject)`). Entry locks, taken in a fixed order, cover operations on a few known entries
   (a merge: two people and their links). Map locks exist and are rare.
5. **Stratum holds no durable state.** It is in memory only: no writer/search roles, no volumes, no redo
   log, no checkpoint, no disk index behind idle eviction. Its in-memory replication (members spread
   across availability zones, backups zone-aware) is for availability and speed, not durability. With
   no disk to reload from, a tenant's Stratum keeps its whole dataset resident (sized and priced so).
6. **The redo service is durability, and the log is the database.** A service that is NOT per tenant
   receives every tenant's writes tagged by tenant id. **A write is acked once the redo instance local
   to the cluster has it on disk — one ack.** (This replaces the earlier "two nodes ack": the one ack
   is the durable one.) The local redo's volume is a regional disk, replicated synchronously across two
   zones by the cloud provider, so that one ack survives a zone loss.
7. **Across regions is asynchronous.** The local redo ships to the redo service outside the region
   behind the ack; a region loss (very rare, clusters span zones) loses only that lag.
8. **Restore is a normal start.** A tenant's Stratum starting anywhere — after a full restart, on
   another cluster, after a region loss — replays its tenant from the redo service. The redo service
   therefore compacts (a snapshot per tenant plus the tail behind it), the job Stratum's daily
   checkpoint does today. Restore is to the latest state; a point-in-time restore is done by editing
   the redo log by hand, as today. Rollouts never take all of a tenant's members down at once.
9. **Watch is first-class on every record type.** It is the change feed that replaces the bus.
10. **The footprint is tunable per tenant** (members, memory), and pricing builds it in. With durability
    in the redo service, a member is a unit of availability, not durability, so **a small tenant may run
    a single Stratum member** and still lose nothing acked at Local or Remote (item 11). What it trades
    is availability: a restart or a rollout is downtime until the replay finishes, and a zone loss means
    rescheduling and replaying elsewhere. Durability is chosen per record type, availability per tenant
    by member count; the two price independently.
11. **Durability is declared per record type in the proto**, as part of `(spec.record)`:
    - **Memory** — acked in memory on the Stratum members; survives a member loss, not a full restart
      (it comes back empty). For working sets, caches, anything rebuildable.
    - **Local** (the default) — acked when the local redo has it on disk; survives a full restart and a
      zone loss.
    - **Remote** — acked when the local redo and the cross-region redo have it; survives a region loss.
      For identity's XID and link rows, tenant names, IdP records: rare writes that must never be lost.
    A single write may ask for a higher level than its type declares, never a lower one. The redo log is
    a prefix, so waiting for Remote on one write makes every earlier Local write remote too. An operation
    spanning record types takes the highest level involved. Watch and reads see a write only after its
    declared ack. This replaces Stratum's per-record durability floor (`min_durable_copies`).

## Scope now (Jody)

**Now:** resilience at the active site, and the data backed up to another site. That is all.
- At the active site: per-tenant, in-memory Stratum spread across zones (items 1, 4, 5, 9, 10), writes
  acked on the local redo's regional disk (item 6), durability levels per record type (item 11).
- At another site: the redo service's copy (item 7), shipped asynchronously. It is the backup a tenant
  is restored from (item 8), by hand.

**Not now:** a full region failure. Nothing here has to make a region loss automatic. Everything below
is kept as the direction, not dropped.

## Kept for later: region failover

- Stratum never runs outside one cluster. Two clusters, in two regions or clouds: one active, the
  other's gateways forward to it (DNS steering to the active as the primary path, forwarding as the
  fallback).
- The passive cluster's local redo is the remote copy, so Remote-level writes survive a failover with no
  loss; the passive Stratum tails the redo so failover is a promotion, not a full replay.
- A tie-breaker (witness) at a third cloud provider grants the active a lease with a term. The passive
  promotes only after the lease expires plus a clock-skew margin and the witness grants a higher term.
  The active self-fences (stops acking) before its lease runs out; every redo write carries the term and
  the redo refuses an older one, so a cut-off old active cannot make a write durable after takeover.
- Kernel's state (signing keys, trust domain) and the public name's certificate fail over with it.
- Open then: warm or cold passive, manual failback (suggested), whole cluster or per-tenant active.
- This supersedes, for later, items 1 and 2's "across clusters" placement and the `people` region
  stretch: with an active/passive pair, no tenant's Stratum spans clusters.

## Consequence for identity

- **A people service, in the `people` tenant, owns the identity graph**: minting XIDs, the link index,
  Person rows and memberships, the first-login write, Link/MergePeople's moves, and the two mint-time
  reads (the link still exists, the membership still exists). Its storage is the `people` tenant's
  Stratum, using items 4 and 9. Kernel keeps what is stateless: signing the session credential and the
  access tokens, the proof rules, Merge and Unmerge, and the per-tenant exchange; it calls the people
  service through the tacplus-generated S2S client, never shared state.
- **Kernel reaches it across clusters through the gateway**, which carries gRPC and is tenant-aware
  (Jody). To check: routing a `/system` caller (kernel) to a tenant-owned service by the target tenant;
  one trust domain across clusters (shared or federated verifying keys); exposing those S2S RPCs at the
  edge to kernels only.
- **Deviation to correct:** steps 1 and 2 (kernel d8ba29d, 28ae372, then 8b41228) put XIDs, links and
  memberships in kernel's `PrincipalState`, which is Hazelcast-backed kernel state. That conflicts with
  10-23's "nothing new in kernel may depend on Hazelcast" and with this record. The credential, proofs
  and exchange are unaffected; the storage moves to the people service.
- The identity record's own surface (Mint, Merge, Unmerge, Link, Unlink, delete account, sign out
  everywhere) is unchanged; only where the graph lives moves.

## Open

- tacplus: generated write surface for records (put, insert if absent, replace if unchanged, delete,
  entry and map locks), and Watch restored on the record plane.
- Stratum: direct writes in place of bus ingest, acked on the local redo; the per-tenant instance
  (operator renders it as a tenant-owned service); removing the disk legs; zone-aware placement;
  full residency.
- Cross-cluster black box: a layer-4 path between clusters (not the gateway), member discovery across
  clusters, and which kernel issues the per-group member certificates when one tenant's members span
  clusters.
- Producers that publish records on the bus today move to direct writes (letsencrypt's certificates
  first; the gateway reads them back through Stratum).
- The redo service: a local instance per cluster (the ack) and where the cross-region copy lives;
  ordering per tenant (a sequence per tenant, so replay is exact); compaction; each tenant writing only
  under its own id (checked against the caller's `/tenants/<id>` identity); restore triggered by a
  Stratum start, and moving a tenant as a platform-operator action; per-tenant encryption keys.
- Ordering between the in-memory write and the redo ack (redo first, then apply, so nothing is visible
  that is not durable), and what a write does while the local redo is unreachable (refuse).
- The people service: its contract, and whether org memberships live in it or with each org tenant.

## Addendum 2026-10-28: the second site is redo to redo (Jody, redo#1)
The local redo ships each tenant's log to the redo at the second site over `redo.RedoSite`, a contract in redo's own proto, and
REMOTE is acked once that site has the write. The call between deployments uses generated cross-deployment S2S clients
(tacplus-dev/tacplus#204), never a hand-written mTLS client. A bucket as the second site (option B) was not chosen. Built in two
steps: the whole service, tested against an in-process second site, now; the deployed link once #204 lands.
