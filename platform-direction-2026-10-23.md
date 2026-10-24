# Platform direction (Jody, 2026-10-23)

Jody, in a huddle with Claude about where identity's durable record lives:

- **The platform moves to open source.** What is in `seabeam/schematic` today moves to github.com/tacplus-dev.
  *"We are moving the stuff that is currently in schematic to open source tacplus-dev (github)."*
- **Plug and play fleets.** A fleet is defined by its fleet definition; much of it comes from tacplus-dev. Each
  service's proto config moves to the repo that owns the service. When a fleet is built, a public and a private
  registry are consulted to find every dependency, and kernel is then built for that fleet.
- **Stratum goes public** (tacplus-dev). It is the database: high-performance SQL search, dynamic result sets,
  dynamic instances spun up on demand. **Hazelcast stays in Stratum.**
- **Kernel is just a state object.** *"It uses Hazelcast but thats mostly because Kernel and Stratum were once the
  same thing."* Hazelcast will eventually be removed from kernel. (Consistent with "Hazelcast stays; replacement
  withdrawn as a plan", 2026-10-22: Hazelcast stays in Stratum.)

## Consequence for the identity graph (identity-graph-huddle-2026-10-21)
- **Stratum is identity's durable, atomic record** (huddle item 11 stands). Kernel holds working state and mints
  tokens; nothing new in kernel may depend on Hazelcast (no Hazelcast CP for identity invariants).
- Stratum needs, in this order, for identity: (1) insert-if-absent with a unique index (one XID per login key, one
  TID per name); (2) a revision per record / compare-and-set (bind, revoke); (3) a change feed so kernels invalidate
  instead of waiting for a miss; (4) multi-record atomicity for merge (last; merge is deferred anyway).
- Until those exist, kernel's IdentitySnapshot keeps identity durable inside the one identity fleet, behind an
  interface shaped like those Stratum operations so the switch is a backend swap.
- Kernel→Stratum dependency is an ordering rule, not a cycle: kernel mints its own token; kernel boots, then Stratum,
  then identity writes are available (fail closed for mint/bind/merge/revoke while Stratum is unreachable; cached
  identities keep signing in — huddle item 21).
- Deferred (Jody 2026-10-23): device attestation.
