## Status line first

The design marks several points "(Jody)". Those are his most recent statements on those points, so they are where the review starts. Where I push back on one of them below, I say so, because pushback on his own calls is invited: *"I like pushback."* Everything not marked (Jody) is treated as open.

One framing problem comes before the details. **flock quietly reopens something the 2026-10-21 panel called sound.** That panel listed "Stratum-first for invariant-bearing writes" and "one key to one XID with a conflicting login refused" under "do not reopen". flock replaces prevention with repair: there are no locks, a duplicate is allowed to exist, and it is merged later. Jody's newer statements (*no locks of any kind*, *Stratum is more than this needs*) carry more weight than the reviewers' consensus. But the design has to say outright that it supersedes those two items and why. Otherwise the next reader finds two documents that contradict each other. *Derived from "attribution is load-bearing" and "newer is closer to the truth", not said by Jody.*

## Pushback

### 1. "Majority of known copies" breaks when "known" means "currently healthy" (pushback on a Jody point)

Physical removal waits until a majority of known copies have seen the mark. Under Option A, membership comes from kernel fleet health. During a partition, the minority side's health view shrinks to itself, so it is a majority of one. It then removes data the other side has never marked.

The majority rule has a second, separate problem. Suppose a copy missed the mark and the mark is later removed. That copy gossips the live entry back and resurrects it, and nothing remains to defeat it. This is the classic tombstone-GC failure.

- **Recommendation:** removal waits until **every declared copy** has acknowledged the mark, not a majority. A copy that has been absent longer than the GC horizon is **wiped and rebuilt from its peers. It is never merged.** That bounds the failure by construction.
- **Derived from:** *"Ethernet should have had a TTL. Because it should have been fail safe."* Not said by Jody.
- **What would change this:** a proof that a copy which missed a removed mark cannot re-introduce the entry. I don't see one in the draft.

### 2. "Serves nothing until caught up" relies on a signal that may never arrive

A copy cannot tell whether it is behind or partitioned. Absence of gossip carries no information. Either way, the rule as written has a problem:

- It leaves a restarted cluster dark for the whole partition, which contradicts "a cluster must keep serving reads through a partition".
- Or it uses "no news" to mean "caught up".

Derived from the MTU story: *"the absence of a message is not information, so a design that treats it as such is wrong."* Not said by Jody.

"Caught up" needs a definition: digests match a majority of the **declared** copies. And the trade-off against availability goes to him (see Conflicts).

### 3. Hand-written code deciding "these two are duplicates" should not exist

The design keeps the survivor choice in generated code, which is right. But it still lets hand-written code decide *that* two identities merge. A merge moves authority from one identity to another, so it is AAA.

- **Quoted:** *"AAA is a job that should not be left to user code."* One buggy or compromised service calling `merge_entries` is an account takeover.
- The design already has a generated path: adding a link that collides on `(flock.unique)` triggers the merge. A user who says "these are both me" proves the second link, the link is added, and the generated check merges.
- **Recommendation:** drop the hand-written entry point. The only trigger for a merge is a unique collision. Derived from the AAA rule; one step.

### 4. Every service following merges by hand is the substrate's job

"Services learn about merges by watching the table" means every service holding an XID writes its own redirect-following code. The three-part test applies directly:

- every tenant needs it;
- it must be done identically to be correct;
- getting it wrong in one tenant mints for a dead identity.

So the substrate owns it. **Recommendation:** add a typed reference annotation (say `(flock.ref) = Person`) in place of `bytes xid` on `Session`. Generated clients then resolve redirects themselves. This also satisfies *"generated surfaces are typed"*, since a bare `bytes xid` is an untyped reference. Derived from the three-part test (confirmed, agent-proposed), not said by Jody.

### 5. "Transparent to services" is transparent in shape, not in behaviour (pushback on a Jody point)

The interfaces stay the same, but the consistency behind them changes. Callers that did a read-check-write against Stratum or Hazelcast, which is exactly the uniqueness CAS the 10-21 panel identified, would compile and silently lose their invariant.

- **Quoted:** *"abstractions must complete[ly] hide the layer below from the layer above."* A contract that changes underneath a caller who relied on it is the leak, even when the method signatures don't change.
- **Recommendation:** make the consistency contract part of the generated type. Moving a table onto flock is then a rebuild, and anything that relied on linearizable writes fails to compile. *"Fixing an issue is done once in the generator and picked up during rebuild."*
- This stays consistent with his "transparent" for services that never relied on it.
- **What would change my mind:** an inventory showing that no current caller of `IdentityGraph` or the state clients does read-modify-write.

### 6. The mark contradicts the redirect

One part of the draft says a copy *refuses* a marked entry: no new tokens. Another says a read of B *follows* the redirect to A. For a `MERGED` mark those disagree.

- **Recommendation:** `MERGED` resolves to the survivor and mints with `sub = A`. `USER` and `EXPIRED` refuse.
- Say so per reason, in the generated code.

### 7. Concurrent writes to the losing entry are silently discarded

"A mark always wins." So a device added to B on cluster 2 while cluster 1 merges B into A is lost without any error. That is a hidden failure.

- **Quoted:** *"the market works because of failure not in despite of."* A failure nobody sees cannot be corrected.
- **Recommendation:** when a newer version of a merged loser arrives, fold it into the survivor with the existing `merge(survivor, loser')`. That is already a commutative fold. Then apply the mark.

### 8. One flock instance per cluster is a single point of failure for every token mint in that cluster

- **Quoted:** *"I don't like active-passive… the best is active^N."*
- The protocol has no reason to care whether a copy is a cluster or a replica. **Recommendation:** N copies per cluster, all peers, each counted as a declared copy.
- Derived, one step.

### 9. Merges need a fall-forward path

A merge marks the loser permanently, and after the majority rule the loser's data is gone. The 10-21 doppelganger noted that deleting XIDs with no restore path meets this repo's definition of irreversible.

- **Quoted:** customers must *"never end up in a situation with no ability to fall back or forward."*
- **Recommendation:**
  - Keep the loser's full data, not just the stub, for the XID age-out window from `decisions/tenant-deletion-2026-10-27.md`.
  - Add a generated **split** that recreates a new entry from it. That is fixing forward, not undoing the mark.

### 10. Tests: what would have failed?

The kitchensink property tests check the algebraic laws of each rule. They exercise nothing at the system level.

- **Missing:**
  - a partition test that creates a duplicate on both sides and checks that both sides converge on the same survivor;
  - a resurrection test for item 1;
  - a clock-skew test;
  - a test of a write racing a merge, for item 7;
  - an order-independence property test for `merge_entries` itself, since the design claims it.
- "Once per release" is also too rare. **Quoted:** *"testing must be done at all stages of development"*. These run on every tacplus build, as a gate.

### Smaller points

- **`devices` has no merge annotation**, yet "a field with no rule is a generator error". As drafted, the map gets a default rule nobody wrote down. **Quoted:** *"Explicit annotation over inference."* Annotate it, and define how a revoked device is removed from a map.
- **`SET_REMOVE_WINS` re-add semantics are unstated.** If "remove wins" is permanent, a user who unlinks Google can never link it again. It needs per-add tags (OR-set semantics), and the design should say so.
- **Set elements with non-identity fields** (for example a `proved_at` on Membership) need a merge rule per field. Say that the field-by-field rule applies inside set elements too.
- **`created_ms` must come from flock's HLC, never from the caller.** The survivor is chosen by created time, so a caller-supplied time lets the attacker choose the survivor.
- **Two-person rule (Jody):** which writes count as "really important" has to be an annotation. Otherwise every service decides it differently.

## Open questions, answered

**1. Membership and transport.** Split them. Membership comes from the fleet's **declared** set of copies, as config. It does not come from health, which is state. Transport is Option A's generated cross-cluster S2S clients.

- Membership is intent, so it is declared. Liveness is state, so it is observed. Using liveness as membership is what breaks item 1.
- Transport must be generated, attested S2S: **quoted**, *"reachability implies authorization was and is a logical fallacy"*. A bare peer list of addresses would be gating on topology.
- Building the tacplus cross-cluster client feature once also serves redo#1. Under the parity rule it lands in Rust, Go and Java.
- One extra flag: if fleet health is Hazelcast-backed, flock now depends on Hazelcast, and on itself once kernel state moves onto flock. That is another reason not to use health as membership.
- **Confidence: 70%.**

**2. Merges as an attack path.** The shared-`(flock.unique)` trigger defeats the 10-21 attack (pre-create cheap XIDs and wait), because the attacker would have to prove the victim's IdP account. It opens a different one: **a merge is a union of authority.**

- **Example:** an org IdP admin resets an employee's password, proves the employee's org account and links it to the admin's own XID. The collision merges the admin's personal links with the employee's personal XID. The admin now holds the employee's Scotty.
- **Recommendation:**
  - **(a)** Every link add needs fresh proof of possession in the same request, from a session already authenticated as that entry. A locally visible collision is **refused**: the 10-21 rule stays as the gate.
  - **(b)** `(flock.unique)` is allowed only on links at an anchoring assurance level. Org-IdP and address-shaped links never trigger a merge. A collision on them marks the link conflicted and surfaces it.
  - **(c)** A collision revealed by gossip auto-merges only when one entry's links are a subset of the other's, so no link gains authority it did not already have. Otherwise both entries survive, sign-in through the shared link is refused, and the user merges explicitly by proving every link.
  - Under (c) the survivor choice ("older") stops mattering for security and only serves determinism. Keep it.
- **Derived from** "reachability implies authorization" and the 10-21 "never an org-IdP proof alone"; two steps. **Confidence: 55%.** This touches identity, so it should be re-huddled rather than acted on.

**3. Redirect stub lifetime.** At least the XID retention horizon from the 2026-10-27 tenant-deletion decision, and probably for good: as a hash-only pseudonymous record, per the 10-21 point that audit rows and backups name XIDs forever. The stub is tiny and losing it lets a stale copy resurrect B.

- Removal, if it ever happens, follows the same all-copies-acknowledged and rebuild-if-absent rule as marks (item 1), not a majority.
- **Confidence: 65%.**

**4. `IMMUTABLE` conflicts.** Neither option as posed.

- A local write that changes an `IMMUTABLE` field is **refused at write time**. **Quoted:** *"Validate and reject; never mutate the input"*, applied as a gate.
- At merge, two differing values can only come from a bug, because flock creates keys on one copy. The merge must still converge, so it keeps the value with the earliest HLC on every copy and raises a conflict record.
- "Keep both" cannot be expressed in a single field, and it breaks the merge laws.
- **Derived**; one step from "a difference is a bug and is surfaced" (already in the design) plus convergence. **Confidence: 75%.**

**5. What the people table holds.** Add these:

- `last_sign_in` as `MAX` (an HLC, not caller milliseconds), for the age-out.
- An **assurance level on every `Link`**, which (b) in Q2 needs.
- On memberships: `proved_at` and the IdP's `auth_time` (MAX), per the 10-21 revocation finding.
- The loser's retained data and the `split` source (item 9).

Nothing else until a consumer needs it: *"No features beyond what was asked."* **Confidence: 60%.**

**6. Clock bounds.** Yes.

- A copy picks its own timestamp, so an unbounded HLC is a control gated on a signal the controlled party chooses. **Quoted:** *"the control must gate on something the party cannot choose."*
- Don't just drop far-ahead writes at the receiver, because receivers' clocks differ and they would diverge. Instead:
  - **(a)** A copy that sees peers' HLCs differ from its own by more than ε **stops accepting writes itself**. It still serves reads. **Quoted:** *"it's okay for apps to fail but much less okay for them to take down another app."* The bad copy fails, not its siblings.
  - **(b)** A receiver holds a write more than ε ahead (applying it once its clock catches up) and raises an alert. Without the hold, one bad write drags every copy's HLC into the future permanently.
- **Confidence: 70%.**

## Conflicts escalated, not ranked

- **Revocation during a partition:** should a cluster cut off from the rest keep minting for an identity that was signed out or revoked elsewhere (availability through partitions, Jody), or stop minting after T without contact (the failure bounded by construction, Ethernet-TTL)?
- **Restart during a partition:** should a copy that restarts during a partition serve reads that might be stale, or stay dark until it can prove it has caught up?

## Sound as drafted

- flock-generated random keys.
- Generated survivor choice that does not depend on call order.
- HLC with copy-id tiebreak.
- No hand-written per-field merge.
- `(flock.identity)` as the shared basis for set semantics and duplicate detection.
- Two-argument `merge` with fold semantics.
- The B → A redirect concept.
- Moving Hazelcast-backed kernel state later rather than now, consistent with 2026-10-23.

**Not covered:** rollout. flock's gossip, index and GC are hand-written, so they get canaried per cluster, with the current store staying authoritative until each cluster's reads match. Derived from the canary rule, which exempts generated code but not this.
