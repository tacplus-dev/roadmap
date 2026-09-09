# Problem: how a `/system`-identity service reads tenant-scoped data, as a general tacplus capability

## Context — how this came up

DocManagerAgent is a "virtual owner" service (settled design: it acts under its own static
service identity, never on behalf of a requesting user — see
`~/workspace/gitlab/secondstarcavalier/scotty/backend/scotty-design-record.md` §9 and
`~/workspace/loopers/docmanager/decisions/service-agent-rpc-split-and-identity-2026-09-07.md`).
It's meant to autonomously read and organize a company's documents. Its own S2S identity,
like every service identity in the platform, is necessarily `/system` — but the documents it
needs to read are tenant-scoped (`/dr-pepper`, `/sprite`, etc.), not `/system`-scoped.

A dispatched investigation (`~/workspace/loopers/docmanager/workers/namespace-grant-investigation-result.md`,
284 lines, iteration 8015-8016) traced this through real source rather than guessing:

- A service token's `namespace` claim is derived **unconditionally** from its SPIFFE subject
  (`IdentityServiceImpl.java:1661-1662` in
  `platform/services/kernel/server/.../cluster/IdentityServiceImpl.java`), and every service
  subject is built as `spiffe://<td>/system/services/<name>` with `/system` hardcoded —
  see `SpiffeIds.build` in tacplus's `pkg/templates/rust/chassis/*/spiffe.rs.tmpl` templates.
  **There is no per-service override or parameterization of this today.**
- `Delegate.authorized_services` (the grant that lets a token be minted for a given service)
  has no namespace field at all — it's an allow-list of *who*, not *what namespace*.
- Separately, the investigation found `docmanager.proto`'s `s2s_read` record-reader plane is
  **already unscoped by namespace, by design** (`docmanager.proto:248-252`'s own comment says
  s2s record_read "does not auto-scope by namespace"). So DocManagerAgent's S2S grant, as
  generated, can *already* read across every tenant's rows — the `/system` restriction it's
  currently hitting is a **hand-written, self-imposed `caller.namespace()` comparison in its own
  `service.rs`**, not a platform limit.

## The actual problem, generalized

There are two overlapping issues, not one:

1. **Immediate**: DocManagerAgent's own code defensively filters by namespace even though the
   platform already grants it unscoped S2S read access — an easy, narrow fix (drop that
   self-imposed filter), but it works *by accident of an undocumented design note in one proto
   comment*, not because tacplus has a real, declared concept of "this service may read across
   tenants." Any other service built the same way (mirroring docmanager's pattern, which is
   explicitly called out as the reason docmanageragent added the filter in the first place) would
   make the same mistake in the opposite direction just as easily.
2. **General/durable**: tacplus has no first-class, generator-level, auditable way to say "service
   X, acting under its own `/system` identity, is authorized to read [these record types / these
   RPCs] across any tenant namespace." Today this capability exists only as an implicit
   side-effect of one proto's read-plane being unscoped — invisible in `registry.json`, not
   something a reviewer or a future generator user can discover, decide about, or restrict per
   service. It's tribal knowledge sitting in a comment, not policy.

## Proposed direction (not yet decided — this is what needs research/critique)

Make cross-tenant read access for trusted `/system`-identity services a **declared, generated,
auditable capability** in tacplus, not an implicit behavior:

- Something like a proto/service-level annotation — e.g. on the RPC or on the record type —
  that explicitly grants a named service cross-namespace read for specific RPCs/record types.
  Conceptually similar to the existing `Delegate`/`authorized_services` grant model, but adding
  the missing dimension (namespace scope) rather than leaving it absent.
- Must stay **read-only by default** and **scoped to specific record types/RPCs**, not a
  blanket "ignore namespace" flag — same shape as the already-settled "no delete entitlement,
  ever" rule for DocManagerAgent. A service shouldn't get more than it declares needing.
- Should be **visible in `registry.json`** (or wherever generated auth config lands) so it's
  auditable — anyone reading the generated output can see which services hold cross-tenant
  reach and for what, rather than having to read hand-written service code or a proto comment
  to discover it.
- Generic, not DocManagerAgent-specific — per the platform's own governing principle
  ("nothing hand-written may scale with application types" — `schematic/goal.md`), this should
  be a capability any future service can opt into via annotation, generated the same way for
  everyone, not a one-off carve-out.

## Open questions for research

1. Is extending `Delegate`/`authorized_services` with a namespace-scope field the right shape,
   or does this want to live closer to the record/RPC definition itself (proto-level annotation
   near `s2s_read`, which is where the *existing* unscoped-by-design behavior already lives)?
2. Should the grant be expressed per-RPC, per-record-type, or per-field — and does the platform
   have (or need) a way to further restrict *which* tenants (allow-list) vs. *all* tenants?
3. How does this interact with the existing `s2s_read`-is-unscoped-by-design behavior — is that
   the right default to keep (and this annotation is about making it explicit + auditable +
   opt-out-able), or should the default flip to scoped-unless-declared (safer, but a bigger
   behavior change touching every existing `s2s_read` consumer)?
4. What does the generated code path actually look like — is this enforced in the same
   `verify_token`/`Caller` construction path, or somewhere else in the generated auth chassis?
5. Precedent check: does anything like this already exist elsewhere in tacplus/schematic under a
   different name that should be reused/extended instead of building something parallel?

## Pointers into the actual code (read these, don't guess)

- `~/workspace/gitlab/secondstarcavalier/schematic/platform/services/kernel/server/src/main/java/com/gitlab/secondstarcavalier/controlplane/schematic/services/kernel/cluster/IdentityServiceImpl.java`
  (~line 1661) — where a service token's namespace claim is derived from its SPIFFE subject.
- `~/workspace/gitlab/secondstarcavalier/schematic/tacplus/pkg/templates/rust/chassis/spiffe/src/spiffe.rs.tmpl`,
  `.../servicemanager/src/spiffe.rs.tmpl`, `.../grpcserver/src/spiffe.rs.tmpl` — `SpiffeIds.build`
  and where `/system` gets hardcoded into a service's SPIFFE subject.
- `~/workspace/gitlab/secondstarcavalier/docmanager/proto/services/docmanager.proto` (~line 248-252)
  — the `s2s_read` "does not auto-scope by namespace" design comment that's the current de facto
  (undeclared) mechanism.
- `~/workspace/loopers/docmanager/workers/namespace-grant-investigation-result.md` — the full
  284-line investigation this problem statement is drawn from; read it for the complete trace and
  reasoning, including the three narrow options considered (and rejected as insufficiently general)
  for DocManagerAgent alone.
- `~/workspace/loopers/docmanager/decisions/watch-rpc-is-record-topic-event-watch-2026-09-07.md`
  and `~/workspace/loopers/docmanager/decisions/service-agent-rpc-split-and-identity-2026-09-07.md`
  — the settled "virtual owner, static service identity" design ruling that makes this problem
  arise in the first place.
- tacplus's own `Delegate`/`authorized_services` grant model (wherever it's defined in
  `~/workspace/gitlab/secondstarcavalier/schematic/tacplus`) — the closest existing precedent for
  "a declared, generated grant naming what a service may do."

## What's wanted from this research pass

Critique the proposed direction above, don't just build it. In particular:
- Is this the right generalization, or is there a simpler/more idiomatic tacplus mechanism that
  already fits this shape?
- What does this break or complicate for existing services if adopted?
- Concrete proposal for the annotation/grant shape, with a sketch of what it'd generate.
- Anything in the "open questions" section above that has a clear right answer once you've read
  the actual code, vs. what's a genuine product/security call that should come back to Jody.

Do not write production code against the live tacplus repo — this is a research/design pass.
