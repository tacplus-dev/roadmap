# Identity graph: XID, mappings, TID, per-tenant tokens

- **Status:** held
- **Owner:** Jody (design), schematic looper (build, when released)
- **Opened:** 2026-10-22
- **Held on:** the 2026-12-01 launch. Sign-in works today through the OPA namespace stopgap, and the
  schematic looper is actively investing in that shape (email-domain claim matchers, post-auth
  domain-to-tenant mapping). Building this mid-stream halts that work and forces a full-fleet roll
  that invalidates every in-flight user token.
- **Gets more expensive while held:** the user `sub` in records owned by SPIFFE id. Today there are
  zero real user rows, so the change is a regeneration; after customers exist it is a data
  migration. Everything else stays flat.
- **Records:**
  - huddle (nothing decided): `schematic/roadmap/identity-graph-huddle-2026-10-21.md` (items 1-24)
  - panel + doppelganger review: `schematic/roadmap/identity-graph-panel-2026-10-21/SYNTHESIS.md`
  - code survey and estimate: `schematic/roadmap/identity-graph-estimate-2026-10-22.md`
  - prior decisions it departs from: D29 (`schematic/looper/schematic/decisions/d29-idp-uncoupled-from-tenants-design-2026-10-02.md`),
    `schematic/roadmap/tenant-owned-services-decisions-2026-09-08.md`,
    `sonar/backend/decisions/login-and-identity-model-2026-09-08.md`
  - fleet shape it assumes: `schematic/roadmap/fleets-and-canary-huddle-2026-10-22.md`

## Movement while held (2026-10-25)

The tacplus side of (a) already landed: tacplus v2.2.0 carries the `/users/<xid>` SPIFFE root for
user principals with `/people` retired (#131, PR #136, by Jody's other session). The kernel and
schematic side (mappings, TID, membership record, per-tenant exchange generalized) is still held.

## What

Users move from `/tenants/people/<pid>` to `/users/<xid>`, an immutable random id we mint. Login
identities are mappings keyed `(idp_id, immutable subject)`, one XID per key; personal mappings
(Google, Apple, Microsoft) mint the XID and come first, so the person's agent survives leaving an
employer; org mappings are memberships with a thin record (proved_at, max proof age, revoked) and
no onboarding step, the org IdP is the proof; device keys are mappings with an assurance level;
tenants get an immutable TID with names as mappings; IdPs get an immutable id; one session token
plus per-tenant tokens minted by exchange; XID merge with fresh proof of every mapping; the kernel
owns the graph with Stratum as the durable atomic record.

## Why held

In Jody's words (2026-10-22): "we wont be doing this project yet but we want to put it on the
roadmap and then reference why we are holding." The reason is the launch: the stopgap works, the
looper is building on it, and the cost of waiting is flat until real users exist.

## Minimal path (from the estimate)

About 3-4 weeks of looper work plus one coordinated fleet roll: treat `PersonalPrincipal` (already
an opaque kernel-minted id) as the XID row, move its root to `/users`, add mappings keyed by IdP id
and subject, make sign-in resolve a mapping instead of deriving an id, add one membership upsert
RPC, and use the existing per-tenant exchange. Deferred inside the project: TID as a stored
attribute with the name still in the path, Stratum as the durable record (the per-cluster identity
snapshot already holds the invariant for one cluster per domain), merge, device attestation.
