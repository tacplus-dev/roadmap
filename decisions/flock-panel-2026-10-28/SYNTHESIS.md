# flock design: panel synthesis (2026-10-28)

Subject: `items/flock-design-2026-10-28.md` (design draft, nothing decided beyond the points marked Jody).

Reviewers:
- claude-opus-5.5 (claude CLI, no tools)
- gpt-6.1-sol (codex, high effort, read-only)
- copilot, its default model
- doppelganger: answering as Jody from `~/workspace/jody`, run as a Claude session given those files, because the service isn't
  reachable from here

The prompt is in `PROMPT.md`, and the raw outputs are beside this file.

## Where the reviewers converge (treat as real)

1. **Purging after "a majority has seen the mark" brings deleted entries back** (all four). A copy that missed the mark still holds
   the live entry. Once the others have purged, catch-up ships the entry back, and nothing is left to defeat it. "Known copies" is
   also a local view, so each side of a partition counts its own majority.
   **Fix:** never purge the mark itself, only the entry's data. Purge needs every member of an explicit, versioned membership list
   (not health state). A copy gone longer than a set limit is wiped and rebuilt from its peers, never merged back.
2. **Merging on a shared link is account takeover again** (all four), and here it can't be undone. An attacker with an old XID adds
   a phished link of the victim's; the local unique check merges the victim into the attacker. This also contradicts the 10-21 rule
   "a conflicting login is refused".
   **Fix:**
   - A conflict visible on a local write is **refused**.
   - Merge automatically only the genuine partition race: both entries were created from the same link within the window.
   - Every other overlap is **quarantined**: refuse to mint through that link and alert, then resolve with proof or a second
     person. The survivor is chosen by assurance or by the human, never by age.
3. **The survivor rule needs a birth stamp that can never change** (all four). `created_ms` is merged with MIN, so absorbing an older
   loser lowers it and the order flips between merges. That makes cycles (A→B and B→A, both dead) possible. The creating cluster's
   clock is skewable, and random keys can be ground for a low one.
   **Fix:** flock stamps an IMMUTABLE creation HLC. A redirect may point only strictly down that order, so cycles are impossible.
4. **Marks have no merge rule of their own** (all four). The same loser can get C→A on one copy and C→B on another, or USER on one
   and MERGED on the other. Digests then never agree, and anti-entropy loops forever.
   **Fix:** a deterministic order across reasons, and across `merged_into` (the end of the chain).
5. **Writes to the loser at the same moment as a merge are lost, so revoked access comes back** (claude, codex). An admin removes B
   from tenant T while another cluster folds B's memberships into A. The mark wins on B, so the removal dies with it, and A keeps T.
   **Fix:** absorption is a full CRDT join of B's state, removals included, into A, re-applied whenever new state for B arrives.
   Writes addressed to B go to A.
6. **Sets and maps need causal metadata** (claude, codex, copilot).
   - Identity fields alone can't tell an old add from a concurrent one or a deliberate re-add, so remove-wins either blocks relinking
     forever or means nothing.
   - The `devices` map has no remove at all, so a lost device's revocation is undone.
   **Fix:** per-element dots or version vectors, with their own tombstone cleanup.
7. **Moving a link looks like a duplicate** (claude, codex). Unlink from A, link to B: a copy can see B's add before A's remove and
   merge two different people. **Fix:** only treat a shared link as a duplicate once both entries' versions of it are causally
   settled, or quarantine instead of merging.
8. **One far-future timestamp poisons every hybrid clock** (claude, codex). Refusing a write at some copies but not others diverges.
   **Fix:** never move the clock beyond local time plus a bound; hold far-future writes until local time catches up; quarantine the
   sender beyond a hard limit. A client also carries the last clock time it saw, so the same user's writes on two clusters stay in
   order.
9. **A partition leaves revocation unbounded** (claude, codex). A cut-off cluster keeps minting for a revoked, merged or signed-out
   XID, and a two-person rule doesn't help with that. **Fix:** a freshness bound. A copy stops minting if it hasn't heard from a
   majority within T, or token lifetime ≤ T. This trades against "keep serving through a partition"; it's Jody's call.
10. **"Transparent" holds for the interface shape, not its behaviour** (claude, doppelganger). Callers that read, check, then write
    against Stratum or Hazelcast compile and silently lose the guarantee. **Fix:** the consistency contract is part of the generated
    type. Moving a table onto flock is a rebuild, and a caller relying on linearizable writes fails to compile.
11. **Per-rule property tests don't test what breaks** (claude, codex). The bugs are in the composition: marks, absorption, unique
    merges, chains, delivery order. **Fix:**
    - per-schema property tests over whole entries plus marks;
    - a deterministic multi-copy simulation (random delivery, partitions, purges) checking convergence, no resurrection and no
      cycles.
12. **flock reopens two 10-21 "do not reopen" items** (claude, doppelganger): Stratum-first for writes that carry invariants, and
    refusing a conflicting login. Jody's newer "no locks" may well win, but the design must say outright that it replaces them, and
    why.

## Raised by one reviewer, worth keeping
- **Doppelganger:**
  - Drop the hand-written merge entry point; a merge only ever comes from a unique collision ("AAA is a job that should not be left
    to user code").
  - A typed reference annotation (`(flock.ref) = Person`) in place of `bytes xid`, so generated clients follow redirects and no
    service writes that code by hand.
  - Say per reason what a marked entry does: MERGED resolves to the survivor; USER and EXPIRED refuse.
- **codex:**
  - `generation` merged with MAX doesn't give "sign out everywhere": two clusters can both go 5→6 for different sign-outs. The
    counter needs a per-copy part, or the clock.
  - A cross-entry merge must be written atomically on each copy.
- **claude:**
  - EXPIRED decided on one copy's stale view deletes active users. Expire only after every copy reports a last sign-in watermark
    past the cutoff.
  - A merge rule must never change, and a new field takes no writes until the whole fleet knows it. Merkle hashing must use a
    canonical encoding.
  - A rate breaker: if merges exceed N per window, stop and quarantine.
  - `generation` exposed only as `bump()`, never set.

## Answers the reviewers agree on
1. **Membership/transport:** A for discovery and transport, sharing one tacplus cross-cluster client feature with redo#1. The
   membership that purge counts is an explicit, versioned list changed deliberately, not health state.
2. **Merge attack:** sharing a unique element is not enough. See convergence point 2.
3. **Redirect stubs:** keep them forever. They're small, pseudonymous, and things naming the old XID have no end date.
4. **IMMUTABLE:** neither option. A set-once field written only by generated code; a conflict becomes a visible conflict set that
   fails closed.
5. **People table:** add last sign-in (day granularity, MAX), a per-link assurance level and org/personal class, and the immutable
   creation HLC. A USER mark removes personal data.
6. **Clock bounds:** hold, don't refuse (convergence point 8).

## What the reviewers say is sound
- Random keys owned by flock.
- Merge rules declared in proto, with a missing rule a generator error.
- Two-argument merge, with more than two as a fold.
- A hybrid clock rather than the wall clock.
- Deletion marks with a reason.
- Redirects that outlive the merged entry.
- Detecting partition duplicates in generated code.
- Keeping the survivor choice out of hand-written code.
- Local reads during a partition.

## Decisions for Jody
1. **A link conflict on a local write:** refuse it (the panel) or merge?
2. **Survivor choice:** age (the draft), or assurance with the human for everything except the partition race (the panel)?
3. **Purging:** keep marks forever and purge only data (the panel's preference), or purge with full membership plus wipe-and-rebuild?
4. **A freshness bound on minting during a partition** (stop after T without a majority), against serving throughout.
5. **The 10-21 items:** confirm that flock replaces "Stratum-first" and "refuse a conflicting login" for the people table.

## Jody's answers
1. **Refuse (Jody, 2026-10-28).** A link conflict seen on a local write is refused. flock merges automatically only in the genuine
   partition race: both entries were created from the same link within the convergence window. Any other overlap is quarantined:
   no minting through that link, an alert, and a person resolves it with proof.
2. **The older entry survives (Jody, 2026-10-28)**, ordered by an IMMUTABLE creation HLC that flock stamps once (the panel's fix for
   point 3). A person resolving a quarantined overlap decides *whether* to merge, with proof of both entries; the generated helper
   still decides *which one* survives. Read as applying to every merge, since refusing a local conflict (answer 1) already closes
   the takeover path that made "keep oldest" dangerous.
3. **Marks are purged, but only once every copy has seen them (Jody, 2026-10-28).** Not forever: a mark is removed once every copy
   in the fleet's membership list has acknowledged it. A majority is not enough, because a copy that missed it would ship the live
   entry back. A copy absent for more than **30 days** is wiped and rebuilt from its peers on return, never merged. Isolation is
   transient (Jody): no cluster is expected to be cut off for anywhere near that long. Redirect stubs for MERGED entries are dropped
   after the longest credential lifetime (tokens, sessions), not kept forever. This replaces the "majority of known copies" rule.
4. **Keep minting through a partition (Jody, 2026-10-28).** A cut-off cluster keeps serving sign-ins and minting tokens. There is no
   freshness bound that would stop it. A revocation or deletion made on the other side reaches it when the partition heals; until
   then, the exposure is the length of the partition plus one token lifetime. Isolation is transient, and short token lifetimes are
   the bound.
5. **flock is an alternative to Stratum, not a replacement (Jody, 2026-10-28).**
   - For the people table, flock is chosen in place of Stratum-first. The 10-21 "do not reopen" item is superseded there, and only
     there.
   - flock will evolve and merge with kernel in some form.
   - Hazelcast comes out of **kernel**, not Stratum: Stratum keeps Hazelcast (as ROADMAP.md already says).
   - Still open: should read-check-write callers fail to compile when their table moves to flock (the panel's point 10)?
