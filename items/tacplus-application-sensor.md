# Application sensor: generated wrappers emit the decision record

- **Status:** exploring
- **Owner:** Jody
- **Opened:** 2026-10-26
- **Held on:** (not held; exploring; not in the 2026-12-01 launch scope)
- **Gets more expensive while held:** every fleet service built before the envelope gains the new
  fields ships without them; the wire change is additive, so the cost is a rebuild, not a migration
- **Records:** `tacplus/pkg/proto/sys/accounting.proto` (Event envelope, RpcCompleted, RpcError,
  TokenRejected, StateUpdated, SessionSummary), `tacplus/pkg/proto/sys/trace.proto`,
  `tacplus/docs/API.md` "Tracing And Registry" (durable AccountingRecord topic + spool),
  `tacplus/cross-test/accounting-identity/README.md` (verified-identity gate, F10),
  `tacplus/pkg/proto/sys/spec.proto` (`redact`, `data_entitlement`, `identity_spiffe_resource`,
  RpcInfo governed classification), `tacplus/roadmap/LANGUAGE_PARITY.md`

## What

Make the generated service wrapper the security sensor for the application layer: every call
produces, without any handler code, a record of who (verified principal), what was decided
(allow/deny and the rule), on which objects (ids declared in the proto), with what shape (sizes,
latency, caller service) and by which code (release digest). The record is a typed accounting event
on the existing bus and spool; the application team cannot omit it, disable it, or log a secret by
accident because the field classes come from proto options, not from code.

Argument (2026-10-26 conversation on DataBahn/Cylake/SIEM): the firewall sees reachability, not
authorization; the endpoint only helps when the endpoint is yours; the attacks that AI makes cheap
are valid requests from valid principals (logic, object reference, authorization gaps, sequence).
They are visible only where the application decides what a request means. Nobody instruments that
layer because applications are bespoke. tacplus already generates the interceptor on every call, so
the sensor is a property of the generator, not a thing a team remembers to add. Distribution of the
sensor is the problem the industry has not solved; codegen solves it by construction.

## What already exists (do not rebuild)

- `accounting.Event` envelope: timestamp, `source_id`, `request_id`, `identity` map from the
  verified token (`sub`/`aud`/`iss`/`azp`), level. F10 holds Java to the Go record: identity is
  the verified claims, never the header the caller wrote.
- `RpcCompleted` (method, duration), `RpcError` (internal failures only), `TokenRejected`
  (authn failures with reason), `StateUpdated` (entity type/id/action, writes via state services
  only), `SessionSummary` (per-token aggregation bounded by token expiry).
- Durable `accountingrecord.AccountingRecord` topic with in-process spool (4096) and optional
  disk spool (64 MiB, at-least-once, dedupe on `accounting_id`); never on the RPC path.
- Trace v2 (`trace_id`, per-hop TraceEvent, gateway external/internal kinds).
- Field classification already in spec: `(spec.redact)`, message-level `data_entitlement`,
  `identity_spiffe_resource`; RpcInfo carries governed verb/noun/access_kind for C2S.

## Gaps (the item)

1. **Authorization decision is not a typed event.** Allow is implied by `RpcCompleted`; an
   entitlement/authz deny has no event of its own (`TokenRejected` is authn; `RpcError` is
   reserved for our bugs). Add `AuthorizationDecided` [INFO on allow, WARN on deny]: method,
   outcome, which flag/entitlement/service pattern matched, reason on deny. The deny with a reason
   is the one verdict the firewall used to give and no longer can. *(Verify first: how does an
   entitlement deny surface today in Go/Java/Rust wrappers; it may be a bare status code.)*
2. **Objects touched are recorded for writes only.** `StateUpdated` covers state-service
   mutations; a read of record 4471 for tenant X is invisible, and that is the mass-read / object
   reference signal. Add a field option `(spec.resource_ref) = {kind: "..."}` (generalising
   `identity_spiffe_resource`) so the wrapper records the declared id fields of the request and
   response (ids only, never the record) as `objects: repeated {kind, id}` on the envelope.
   `redact`/`data_entitlement` already say what must never be logged; reuse, do not add a second
   sensitivity option.
3. **Shape.** Request/response byte sizes, stream message counts, and the caller service
   (`azp` is on the envelope already; make it a first-class field) on `RpcCompleted`.
4. **Which code decided.** Release digest / version of the emitting binary on the envelope (ties
   to the signed-release idea; the record says what code made the decision).
5. **The durable record drops the envelope.** `AccountingRecord` flattens to `principal_sub`,
   method, code, duration, namespace. Carry the new fields through (`azp`, outcome, objects,
   sizes, digest) or the sensor exists only on the transient bus.
6. **Noise is the next problem, not this item.** At scale this stream needs the same
   sample-then-generate-rules loop as any telemetry, except the events are already structured,
   so rules are trivial next to firewall syslog. Out of scope here; note that `SessionSummary` is
   the existing aggregation seam.

## Why exploring / why not now

Launch scope is frozen (2026-12-01). The wire change is additive and the parity rule applies
(Go/Java/Rust for every chassis capability; Python/TypeScript consume), so the item is three
wrappers plus the proto options plus the durable record, gated by cross-tests like F10. It is also
the thread through Aporeto (identity, not addresses), BlueRock (runtime) and tacplus (generated
AAA); a decision on whether this is a product (sensor only, any SIEM consumes) or a fleet feature
belongs with Jody before anyone writes a generator line.

## Minimal path

Proto first, generators second:

1. Add `AuthorizationDecided` and the `objects`/`caller`/`sizes`/`release_digest` fields to
   `accounting.proto`; add `(spec.resource_ref)` to `spec.proto`; carry the fields into
   `AccountingRecord`. No generator change yet; this is the schema review.
2. A cross-test in the F10 style: entitled allow, entitled deny, a read naming an object, a
   forged bearer (no record), a `redact` field never appearing in any record. Three languages.
3. Wrappers emit. Python/TypeScript remain consumers.

Size: proto + record a day; three wrappers and the cross-test a week of looper time once the
schema is agreed.
