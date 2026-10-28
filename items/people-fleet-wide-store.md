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
3. Quorum for cleanup: a majority of known replicas, or all live ones?
4. What exactly the table holds: person (XID, created, generation, mark), links, memberships, devices. Anything else (e.g. last
   sign-in for age-out)?
5. How kernel reads it (embedded replica library vs a local people-service instance per cluster).
