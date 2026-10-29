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
candidate. flock is an **alternative to Stratum**, not a replacement, and will evolve and merge with kernel in some form. Hazelcast comes out of kernel only; Stratum keeps it (Jody, 2026-10-28).

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

## Revision (Jody, 2026-10-28): kernel and flock, one state model

- **Two services, kept separate.** **Kernel always stays within its cluster. flock spans clusters.** This replaces the earlier "flock
  will evolve and merge with kernel": they don't merge. They share one state model and differ only in where the state lives.
- **The annotations reuse kernel's state model.** There is no `flock.*` namespace. A flock table is a state root:
  `option (spec.service_state) = {label: ["flock"], export: true}`.
  - Boundaries are `repeated` entry fields with `(spec.persistence)` and `(spec.assert_max_size)`, as in kernel.
  - Each entry is keyed by `(spec.store_key)`, which flock creates.
  - Readers use `(spec.state_imports)` and the same generated state clients and Watch.
  - New field options:
    - `(spec.merge)`: `IMMUTABLE`, `MIN`, `MAX`, `LAST_WRITE`, `SET_ADD_WINS`, `SET_REMOVE_WINS`.
    - `(spec.unique)` and `(spec.identity)`, for detecting duplicates.
  - The deletion mark is the existing record `(spec.deleted_flag)`, extended with `DeleteReason` and `merged_into`.
  - The write clock is `(spec.store_epoch)`, which becomes a hybrid clock.
  - The generator refuses a `flock`-labelled root if any field in it has no `(spec.merge)` rule.
- **Moving a state root from kernel to flock** is therefore a label change plus merge rules. Services that import it keep the same
  generated clients.

The `(flock.table)` / `(flock.merge)` spellings earlier in this document are superseded by the `spec.*` forms above.

## Naming idea for later (Jody, 2026-10-29)
**iKernel and eKernel** may be a better naming scheme later: iKernel for the cluster-local kernel (today's `kernel`, which always stays
within its cluster) and eKernel for the one that spans clusters (today's `flock`). That reflects that they share one state model and
differ only in reach. Nothing is renamed now; the names stay `kernel` and `flock` until Jody decides.

## Kernel across clusters (Jody, 2026-10-29)
- **Kernels in one trust domain mint across clusters.** Each kernel's **private** signing keys never leave it; only the **public**
  verifying keys are replicated to every kernel in the trust domain, so any kernel's token verifies fleet-wide. So kernel *does* host a
  fleet-scoped root: its public verifying keys, an add-only set with expiry and no identity merges.
- **Kernel-to-kernel trust uses Kubernetes service-account tokens plus an OPA policy (Jody).**
  - A kernel presents a short-lived **projected service-account token** (audience: the peer kernel).
  - The peer verifies it against the sending **cluster's OIDC issuer** (GKE publishes discovery and JWKS), with no call to the other
    cluster's API server.
  - A generated **OPA policy** decides: the issuer is a member cluster listed in the fleet's proto config; `sub` is
    `system:serviceaccount:tac-<deployment>:kernel`; the audience matches; the token is fresh.
  - Only then are public keys exchanged.
  - There's no new CA tier. Within a cluster, kmaker (TokenReview plus the operator CA) still issues kernel certificates.
- **Hub's proposal (open):** replication is an annotation-driven **generated capability** (`scope: FLEET` on a state root), not a
  service. Kernel uses it only for the verifying-key set. The people table, with its merges, duplicates and quarantine, lives in the
  people service with the same capability, which keeps the complex merge code out of the token authority.

## Decided (Jody, 2026-10-29): the identity graph stays in kernel, as a fleet-scoped root
- Replication is an annotation-driven **generated capability** (`scope: FLEET` on a `(spec.service_state)` root), not a service. There
  is **no flock service and no people service**.
- **Kernel hosts the people table** (XIDs and personal links) as a fleet-scoped, annotation-replicated root, beside its verifying-key
  set. The company tenants' mappings (kernel-owned, per-tenant identifiers) live in kernel too.
- **Why:**
  - The 10-25 reason for moving the graph out (kernel was cluster-local) no longer holds, now that kernel hosts fleet-scoped state.
  - Every mint reads the graph, so it's local and in-process.
  - It removes the circular dependency in which kernel needs a people service that needs a kernel token.
- **The cost, and the condition:** the merge, duplicate and quarantine logic runs inside the token authority. So **none of it is
  hand-written**. It's generated by tacplus, with per-schema property tests and the multi-copy simulation (no resurrection, no cycles).
- The **people tenant still exists**, as the tenant that Scotty and personal services run in, not as an identity service.
- To be put to a **panel** (Jody: "we might ask the panel to weigh in"). Panel questions to include: hosting the identity graph in kernel versus a separate service; and retiring kmaker (kernel#29).

- Supersedes: the identity graph moving to a people service (2026-10-25; kernel#2, people#1) and "a separate flock service".
