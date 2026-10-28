# flock: fleet-wide convergent state, declared in proto

- **Status:** design draft; panel reviewed 2026-10-28 (decisions/flock-panel-2026-10-28/SYNTHESIS.md). Nothing is decided beyond the points marked "Jody". Nothing is built.
- **Owner:** Jody
- **Background:** `items/people-fleet-wide-store.md` (the huddle log this is distilled from).
  `decisions/tenant-deletion-2026-10-27.md` (XIDs kept and aged out after about a year).
  `decisions/platform-direction-2026-10-23.md` (nothing new in kernel depends on Hazelcast).

## Problem

The `people` tenant (XIDs, their links to IdP accounts, memberships, devices) has to span the **fleet**, and the fleet is the trust
domain. Reads are hot: every token mint reads it. Writes are rare: sign-ins, new links, membership changes. A cluster must keep
serving reads and taking writes through a partition from the other clusters. Stratum (per-tenant, redo-logged, transactional) is
more than this needs.

## Shape

**flock** is a service, named the way kernel is. It runs one instance per cluster and holds that cluster's copy of every flock table.
Copies gossip writes to each other and converge with **no locks of any kind** (Jody). Writes are never coordinated across clusters.

Tables are declared in proto. **tacplus generates** the merge code, the duplicate check, the store and the clients from the
annotations. The interfaces services already use (generated state clients, kernel's `IdentityGraph`) stay the same, so moving state
onto flock is transparent to services (Jody). The people table is the first user. Kernel's Hazelcast-backed state is a later
candidate.

Language: Rust (the default).

## Annotations (draft)

```proto
message Link {
  string idp      = 1 [(flock.identity) = true];   // idp + subject identify the element
  string subject  = 2 [(flock.identity) = true];
  int64  added_ms = 3 [(flock.merge) = MIN];
}

message Person {
  option (flock.table) = {};                       // flock creates the key
  int64  created_ms = 1 [(flock.merge) = MIN];
  uint64 generation = 2 [(flock.merge) = MAX];     // "sign out everywhere"
  string display    = 3 [(flock.merge) = LAST_WRITE];
  repeated Link links = 4 [(flock.merge) = SET_REMOVE_WINS, (flock.unique) = true];
  repeated Membership memberships = 5 [(flock.merge) = SET_REMOVE_WINS];
  map<string, Device> devices = 6;                 // map of messages: merged per entry, field by field
}

message Session {                                  // a table that never merges field by field
  option (flock.table) = { merge: LAST_WRITE };    // the newest whole entry wins
  bytes  xid        = 1;                           // no per-field rules: the entry is replaced as a whole
  string device_id  = 2;
  int64  expires_ms = 3;
}
```

### Keys
**flock owns the key (Jody).** It creates every entry's key at random (128 bits, like an XID). A caller never chooses one, so two
creates can never collide.

### Field merge rules
`IMMUTABLE` (a difference is a bug and is surfaced), `MIN`, `MAX`, `LAST_WRITE`, `SET_ADD_WINS`, `SET_REMOVE_WINS`. A nested message
is merged field by field, and a map of messages is merged per entry. A field with no rule is a generator error. There is no
hand-written merge.

### Last write wins, on a clock (Jody)
Some data shouldn't merge: the newest write should simply replace it. `LAST_WRITE` does this, either on one field or on a whole table
(`flock.table.merge = LAST_WRITE`). "Newest" comes from a **hybrid logical clock**: wall-clock time, a counter, and the writing
copy's id to break ties. A write always gets a later time than anything that copy has already seen, so clock skew can reorder only
writes that really were concurrent, never a write and one made after it. Deletion marks still win over any write.

### Generated helpers
- `merge(a, b)`: two versions of the **same** entry, from different copies. **Two arguments only.** Every rule must be commutative,
  associative and idempotent, so merging more than two is a fold. tacplus's kitchensink property-tests those three properties for
  every rule in every language, once per release.
- `merge_entries(a, b) -> (survivor, loser's mark)`: two **different** entries that are the same person. **The generated helper picks
  the survivor (Jody):** the older entry, by created time and then key, whatever order it is called in. Hand-written code may decide
  *that* two entries are duplicates, but never *which one survives*. Two clusters that find the same duplicate at once therefore
  converge, rather than marking each other merged.

### Duplicates: detected by generated code (Jody)
A duplicate made on both sides of a partition is seen by no caller. It only appears when gossip brings the two entries together, so
flock detects it itself:
- **`(flock.identity)`** on a message's fields says which fields make two elements the same element. Sets of messages need this
  anyway: `SET_REMOVE_WINS` has to know which add a removal cancels.
- **`(flock.unique)`** on a set field says no two live entries may share an element. flock indexes the field and checks it on every
  local write and on every entry gossip brings in. On a hit it calls `merge_entries`.

The sign-in service (oauth today) creates an XID, or adds a link to an existing one. If adding a link makes a duplicate, the same
check merges the two entries. Services learn about merges by watching the table.

### Deletion marks
flock adds a mark to every table:

```proto
enum DeleteReason { DELETE_REASON_UNSPECIFIED = 0; MERGED = 1; USER = 2; EXPIRED = 3; }  // Jody
message Mark { DeleteReason reason = 1; bytes merged_into = 2; Hlc at = 3; }
```

- A mark always wins a merge and is never undone. A copy refuses a marked entry (no new tokens) as soon as it sees the mark.
- Physical removal happens only after a **majority of known copies** have seen the mark (Jody).
- A copy coming back from an outage serves nothing until it has caught up.

### Links: a merged entry's redirect outlives it (Jody)
A user can be signed in on one cluster under XID B while, on another, B is merged into A. Tokens, sessions and devices go on naming B.
So a `MERGED` entry keeps a small redirect stub (B → A) after its data is removed, for as long as anything may name B. A read of B
follows the redirect, and a chain of merges collapses to the last survivor.

### Convergence window
Plain gossip is enough, revocations and marks included. For anything really important, use a **two-person rule** (a second approver,
for example the device-approval path), not synchronous confirmation (Jody).

## Open questions for the panel
1. **Copy membership and transport.** Option A: each flock finds its peers in the kernel's fleet health state, gossips over generated
   S2S clients, and catches up with Merkle digests. That needs a tacplus feature: generated clients that reach a service in another
   cluster of the same fleet. redo#1 needs the same feature. Option B: a configured peer list.
2. **Merges as an attack path.** `merge_entries` keeps the older entry. The 2026-10-21 identity-graph panel found that "keep oldest"
   lets an attacker take over an account (pre-create cheap XIDs and wait). Here a merge happens only when two entries share a
   `(flock.unique)` element, such as the same IdP account. Is that enough? What must be proven before a link is added?
3. **How long redirect stubs live**, and whether they also need a majority to have seen them before removal.
4. **`IMMUTABLE` conflicts**: refuse the newer write, or surface the conflict and keep both?
5. **What the people table holds**, beyond person, links, memberships and devices: for example, last sign-in for the age-out.
6. **Clock bounds**: should a write whose time is far ahead of the receiving copy's clock be refused, so a copy with a bad clock
   cannot win every `LAST_WRITE`?
