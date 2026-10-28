# The people tenant spans the fleet: a small convergent store for the identity graph

- **Status:** design (huddle in progress with Jody, 2026-10-28). Changes nothing yet; people#1 / kernel#2 / operator#2 keep their
  current plan until this is decided.
- **Owner:** Jody
- **Opened:** 2026-10-28

## What Jody said (2026-10-28)
- The `people` tenant should **span clusters**: **fleet-wide**. **The fleet is the trust domain** (deployments are a later topic).
- No Stratum for it: overkill. A **purpose-built small store** for the XID table. Reads are hot (every mint), writes are rare.
- **No locks: smart convergence.** Each cluster writes locally; replicas converge.
- **Deletes are marks, not removals.** A deletion is a mark; once a **quorum** of replicas has seen it, a cleanup thread deletes it
  everywhere.

## Sketch so far (hub)
- One replica per cluster; reads always local. Writes local, then gossiped. No consensus on the write path.
- Merge rule per field, so any two replicas that saw the same writes agree regardless of order:
  - **New person:** random XID, no conflict.
  - **Bind `(idp, subject) → XID`:** the one real conflict (the same new sign-in in two clusters at once). The **older XID wins**;
    the other folds into it as a merge (tombstone + redirect, like MergePeople).
  - **Merges:** redirects point only to the survivor; the oldest XID represents the group.
  - **Memberships, devices:** add/remove sets; a removal beats a concurrent add of the same entry.
  - **Generation ("sign out everywhere"):** only goes up; max wins.
  - **Deletion / age-out:** a mark that always wins and is never undone, refused for minting the moment a replica sees it.
    Physical deletion only after a quorum has seen the mark.
- The people tenant then needs no Stratum and no redo log, like `restore`.
- Language: Rust (the default).

## Open questions
1. ~~The convergence window~~ **Answered (Jody, 2026-10-28):** a short window is fine: plain gossip, revocations and deletion marks
   included. For something **really important**, use a **two-person / two-key rule** (a second approver must confirm, e.g. the existing
   device-approval path) rather than synchronous confirmation.
2. Gossip transport and membership: how a cluster's replica joins and leaves; anti-entropy (e.g. Merkle digests) for catch-up.
3. ~~Quorum for cleanup~~ **Answered (Jody, 2026-10-28): a majority of known replicas.** A replica returning after an outage never serves until it has caught up (anti-entropy). Marks win, so a late replica can't revive a deleted person.
4. What exactly the table holds: person (XID, created, generation, mark), links, memberships, devices. Anything else (e.g. last
   sign-in for age-out)?
5. ~~How kernel reads it~~ **Answered (Jody, 2026-10-28): A.** One people-service instance per cluster, in the `people` tenant's namespace, holding that cluster's replica and taking part in gossip. Kernel reads it over a local call with a short cache (as kernel#2 plans), so no identity storage goes back into kernel.

## Direction: a convergent-state primitive in tacplus (Jody, 2026-10-28)
The merge rules are per field (max wins, removal beats add, older wins, a mark is never undone), so they can be **declared as proto
annotations**. tacplus would then generate the convergent replicated store, the merge code and the gossip for any message that
declares them. Uses beyond `people`: other small fleet-wide state, and **eventually replacing Hazelcast in kernel** (platform direction
2026-10-23: nothing new in kernel may depend on Hazelcast).

**Sequencing (Jody, 2026-10-28): generic first.** Design the tacplus annotations and generator first; the people store is their first
user and is generated from the start (no hand-written store to rewrite later). **No hurry**: not on stratum's critical path.
**Transparent:** the switch to convergent state happens behind the generated interfaces (kernel's `IdentityGraph`, the generated state
clients), so services and kernel code that use them don't change. The same holds when it later replaces Hazelcast-backed state.

Still open for the design: gossip transport and replica membership (join/leave, anti-entropy digests); exactly what the people
table holds.
