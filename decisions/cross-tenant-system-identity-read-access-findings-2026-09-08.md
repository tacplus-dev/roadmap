# Cross-tenant read for `/system`-identity services — findings and design critique

Research pass only. No code changed in tacplus, schematic, or docmanager. Every claim below cites
the file and line it was read from; paths are absolute unless the repo root is obvious from context.

Repos: `S` = `/Users/jody/workspace/gitlab/seabeam/schematic`,
`T` = `S/tacplus`, `D` = `/Users/jody/workspace/gitlab/seabeam/sonar/docmanager`,
`L` = `/Users/jody/workspace/loopers`.

---

## 1. Summary / verdict

**The grant the problem doc asks tacplus to invent already exists, and it is `(spec.record).s2s_read`.**
It is declared (on the record), paired (`(spec.record_read)` on the reader, build fails without both),
read-only by construction (the S2S read plane has no write RPC), per-service-per-record-type, and it is
already emitted into `registry.json` as `auth_services` on every `query.RecordMap` RPC for that record.
What is *not* true is the problem doc's characterisation of the cross-tenant behaviour as an
"accidental side effect of an undocumented design note." It is a stated doctrine of the generator:
the Java `(spec.store_namespace)` backstop is **C2S ONLY on purpose**, and the generator source says
why in so many words — on the S2S half "which rows a PEER may see is the auth_services decision"
(`T/pkg/internal/java/types/recordnamespace_gen.go:36-42`). **Every** `s2s_read` grant in the fleet is a
cross-tenant read grant today, not just docmanageragent's.

So the actual gaps are narrower than proposed, and they are these:

1. **Doctrine is in the wrong place.** The "S2S reads are tenant-unscoped" rule lives in a Go generator
   comment and one app proto comment, not on the `s2s_read` field in `spec.proto` where every other
   record-plane rule is written down. That is the "tribal knowledge" part of the complaint, and it is
   real — but it is a documentation gap on an existing grant, not a missing grant.
2. **The registry does not *say* the read is unscoped.** The `RecordMap` RPC entries carry
   `subject_namespace_source: "token.namespace"` (`T/pkg/internal/common/registry/registry.go:1157-1173`)
   which is actively misleading for a surface on which the token's namespace is never consulted. An
   auditor cannot grep `registry.json` for "every cross-tenant read surface in this deployment."
3. **There is no declared, generated way to make an S2S read *narrower* than fleet-wide.** The problem
   doc frames the ask as "grant cross-tenant read"; the honest ask is "let a record require that every
   S2S read of it name one tenant, so a peer with cross-tenant *reach* still performs tenant-*scoped*
   reads, and the tenant is visible per call." That is the only shape here that generates a real check
   rather than decoration.
4. **The DocManagerAgent-specific blocker is not a platform grant at all.** It is (a) the hand-written
   `caller.namespace()` pin in `D/services/docmanageragent/src/service.rs:72-101`, and (b) the
   `(spec.namespace)` marker on `ListDocsRequest.namespace` (`D/proto/services/docmanageragent.proto`,
   `message ListDocsRequest`), which the generated MCP dispatch stamps to the *service's* namespace
   (`/system`) before the handler ever runs (`D/services/docmanageragent/target/generated-sources/model/s2sm/internal/grpc/servers/services/docmanageragent/doc_manager_agent.rs:494-503`).
   Removing the pin alone does nothing for `list_docs`; the generated stamp still yields `/system`.

**Recommendation** (detail in §4): do not add a new "cross-tenant" opt-in on top of a plane that is
already cross-tenant — tacplus's own rule refuses annotations nothing reads
(`T/pkg/internal/common/parser/record_read_validate.go:26-33`, the "decoration" rule). Instead:
(i) write the doctrine onto `s2s_read` in `spec.proto`; (ii) emit an explicit tenant-scope marker on
S2S read surfaces in `registry.json` plus a per-record grant summary; (iii) add one **additive,
per-record** option — `s2s_read_scope: S2S_READ_SCOPE_TENANT_PINNED` — whose generated effect is that
the store's `RecordMap` wrapper refuses a query on that record whose predicate does not pin exactly one
namespace, and the generated Rust reader's `get`/`list` take the namespace as a required parameter.
Default stays today's behaviour. Nothing in this touches identity, tokens, `Delegate`, or the
authorizer, and it cannot be widened into a bypass because it lives on a surface with no write.

---

## 2. What was verified vs. what the problem doc got wrong or oversimplified

### 2a. Verified as stated

| Claim in problem doc | Verified at |
|---|---|
| S2S `namespace` claim derived unconditionally from the SPIFFE subject | `S/platform/services/kernel/server/src/main/java/.../cluster/IdentityServiceImpl.java:1647-1662` — the S2S branch builds an "identity-only token (no profiles — authorization is receiver-side)" and sets `namespace = SpiffeIds.namespaceFromSpiffe(subject)` at 1661-1662. No branch, no config. |
| `/system` hardcoded into every service subject | `S/.../cluster/SpiffeIds.java:31-33` passes `""` as location; `T/pkg/templates/rust/chassis/spiffe/src/spiffe.rs.tmpl:288-304` pushes `SYSTEM_SEGMENT` (line 23) unconditionally. The three template copies (`spiffe/`, `servicemanager/`, `grpcserver/`) differ only in their header comment (diffed). |
| A non-`/system` subject is unparseable as a service | `spiffe.rs.tmpl:467-474` (`classify` requires `segs[0] == SYSTEM_SEGMENT` for `SubjectKind::Service`), `:491-501` (`classify_system` sets namespace to `/system[/location]`). Grammar at `:192-245` admits only `/system…` or `/tenants…`, so the two are disjoint subtrees; only `/` is above both (`SpiffeIds.java:44-50`). Consequence not in the problem doc: the administrative level-or-below rule (`target_namespace_field`) **also** cannot help a `/system` caller reach `/tenants/x`, because `/system` is a sibling, not an ancestor. |
| `Delegate.authorized_services` is an audience allow-list with no namespace dimension | `S/platform/proto/services/kernel.proto:689-713`; enforced at `IdentityServiceImpl.java:1614-1638` (matches `targetAud` against the list or `SERVICE_NAME_ANY`). `custom_claims` cannot smuggle `namespace`: `:1681-1685` skips `STANDARD_JWT_CLAIMS`. |
| The generated Rust reader applies no namespace filter | `D/services/docmanageragent/target/generated-sources/model/s2sm/record_readers.rs:99-134` — `Create → List → Delete` with only the caller-supplied filter. |
| The only thing scoping the agent to `/system` is hand-written | `D/services/docmanageragent/src/service.rs:72-76, 89-92` (`get_doc`), mirroring `D/services/docmanager/src/service.rs:314-345`. **Plus** the generated `(spec.namespace)` stamp on `list_docs` — see 2b(4). |

### 2b. Wrong or oversimplified

1. **"Accidental / undocumented / by side effect."** The original investigation (`L/docmanager/workers/namespace-grant-investigation-result.md` §3) read only the *client-side* reader and the *Rust* namespace generator (`T/pkg/internal/rust/types/namespace_gen.go`), which is the generator for the *reading* service. The enforcement point for the store is the **Java** generator (stratum is Java: `S/platform/services/stratum/server/src/main/java/...`). There, the namespace backstop, the predicate pin, and the watch guard are all gated on `isC2S`:
   - `T/pkg/internal/java/types/servicewrapper_gen.go:1515, 1525-1526` (page filter), `:1650-1661` (predicate pin), `:1708, 1721-1722` (unary filter), `:2191` (watch guard).
   - The doctrine, verbatim, at `T/pkg/internal/java/types/recordnamespace_gen.go:36-42`: *"C2S ONLY … On the s2s half of the same plane the caller is a peer service authorized by identity through auth_services, and narrowing its reads to its own service namespace would not be a tenant boundary — it would break every peer aggregation the record query plane exists to serve. Which rows a PEER may see is the auth_services decision, made before the handler runs."*
   - The S2S half of the record plane is `query.RecordMap` (`auth_model: "s2s"`), the C2S half is `query.RecordQuery` (`auth_model: "c2s"`) — both visible in `D/services/docmanager/target/generated-sources/model/registry.json` under `vservices/query`.

   So it is a design decision with a written rationale, applied uniformly. What is missing is that the rationale is not on the `s2s_read` field doc (`T/pkg/proto/sys/spec.proto:782-838` never says "unscoped by tenant") and not in the registry.

2. **"Invisible in `registry.json`."** Partly wrong. The read grant *is* in the registry: every `RecordMap` RPC for `DocFileMetadata` carries `"auth_services": ["docmanager", "docmanageragent"], "auth_model": "s2s"` (registry.json, `vservices/query` → `RecordMap` → `CreateDocFileMetadataQuery`, and `List…Page`, `Delete…Query`). What is invisible is the *scope*: the entry says `"subject_namespace_source": "token.namespace"` with no target source, which `registry.go:1157-1173` emits for every authenticated RPC regardless of whether the wrapper consults the claim. On this surface it does not.

3. **"Any other service … would make the same mistake."** Already happened in the other direction, and it is load-bearing: `docmanager` (a **user-facing** service) reads `DocFileMetadata` through the same unscoped `s2s_read` and enforces the tenant boundary for its C2S callers **by hand** (`D/services/docmanager/src/service.rs:314-345`). That hand-written pin is the only thing preventing a `/sprite` user from reading a `/dr-pepper` document via `GetDocFileMetadata`. The generated C2S backstop exists for exactly this (`spec.proto:1961-2006`), but it only fires on *repeated* record fields in the response (`namespace_gen.go:53-78`, `servicewrapper_gen.go:1727-1735`); `GetDocFileMetadataResponse.metadata` is a single field. **Not verified** whether the single-field case has a generated backstop — flagging as a side finding to check, not a conclusion.

4. **"Drop the self-imposed filter and the agent reads cross-tenant."** Incomplete for `list_docs`. `ListDocsRequest.namespace` carries `(spec.namespace)` (`D/proto/services/docmanageragent.proto`, `message ListDocsRequest`), and the generated dispatch binds it from the Caller's claim and refuses the tool argument (`doc_manager_agent.rs:480-503`). With the agent's `self_caller` (`L/docmanager/decisions/watch-rpc-is-record-topic-event-watch-2026-09-07.md:146-168`) the Caller's namespace is `/system`, so the handler receives `req.namespace == "/system"` and gets zero rows. Removing the marker would make `namespace` an LLM-writable tool argument, which the same proto's own comment forbids. The tenant has to come from somewhere else (§6, item 2).

5. **"`s2s_read` can already read across every tenant's rows"** — true, and it is worth being explicit that this is also true of `s2s_subscribe`: the agent's topic callback already receives every tenant's `DocFileMetadata` rows with their `namespace` field (decision doc above, lines 73-77; the Java watch guard is C2S-gated at `servicewrapper_gen.go:2191`). Cross-tenant *observation* is already granted, exercised, and declared in the registry (`/docmanager.DocFileMetadataBusStateService/SubscribeToStream`, `auth_services: ["stratum","docmanageragent"]`).

6. **The "arbitrary row on `get(id)`" bug noted in `docmanager/src/service.rs:325-336` is stale.** The generated filter now carries `file_id` (`D/services/docmanageragent/target/generated-sources/model/types/vservices/query/query.rs:409-418`) because `s2s_read` folds the key in (`T/pkg/internal/proto/record_map.go:236-245`, `:606-619`). The handler's `metadata.file_id == req.file_id` re-check is still cheap and correct; the comment calling it TEMPORARY is out of date. Relevant here because once namespace is no longer pinned by the caller, a keyed `get` must actually be keyed.

---

## 3. Precedent — "a declared, generated grant naming what a service may do across a boundary"

Searched tacplus and the platform for every mechanism that names a service on a boundary. In
increasing order of relevance:

| Mechanism | Where | Shape | Cross-tenant? | Enforced where |
|---|---|---|---|---|
| `Delegate.authorized_services` | `kernel.proto:702-706`; **derived from `registry.json`** by the operator (`S/platform/services/operator/pkg/bootstrap/registry.go:56-154`, `example.go:504-598`) | who may mint `aud=X` | n/a (audience, not data) | kernel at mint (`IdentityServiceImpl.java:1614-1638`) |
| `S2SCredPair.authorized_services` | `T/pkg/proto/sys/credpair.proto:30-37` | same, for cred-pair bootstrap | n/a | kernel |
| `CredPair.namespaces: ["*"]` | `S/.../TenantAdminServiceImpl.java:545-612` | tenant allow-list with wildcard | yes — **the only namespace allow-list in the platform** | kernel; but it mints `sub_type: user` (investigation §2), and is the root-admin path |
| `(spec.rpc).auth_services` / `MethodS2SAnnotation.auth_services` | `spec.proto:1352-1356`, `:1234-1245` | per-RPC service allow-list | S2S authorizer never consults namespace at all (`T/pkg/templates/rust/chassis/grpcserver/src/authorizer.rs.tmpl:384-412`; `T/pkg/templates/java/chassis/grpcserver/Authorizer.java.tmpl:449-490`: "raw sub/namespace are never matched directly") | generated wrapper / authorizer |
| `(spec.boundary_callers)` | `spec.proto:1710-1713`, `:2121-2123` ("Cross-namespace call gating on queue/topic/command boundaries is declared with `(spec.boundary_callers)`") → `SecurityPolicy.BoundaryGrants` (`T/pkg/internal/proto/libs_protos.go:486-560`) | per-state-boundary service allow-list, explicitly "cross-namespace" | yes | kernel-side state plane |
| `target_namespace_field` | `spec.proto:1358-1369`; `authorizer.rs.tmpl:216-249` | level-or-below administrative reach | only downward from an ancestor; `/system` is not an ancestor of `/tenants/…` | generated wrapper |
| `PrincipalScope` / `data_grants` in `service_entitlements` | `spec.proto:602-628`; `T/pkg/internal/rust/types/ownerdefault_gen.go:100-115`, `egress_sanitize_gen.go:155-160` | OPA-stamped claim widening a **user's** authority (SELF_ONLY / SUBTREE / ANY) | yes, for users | generated wrapper reads claim bag |
| `(spec.record).s2s` / `s2s_subscribe` / **`s2s_read`** / `c2s_read` | `spec.proto:686-882` | per-record, per-service allow-lists for publish / consume / **read stored rows** / disclose | **yes on all S2S halves** (`recordnamespace_gen.go:36-42`) | stratum's generated `RecordMap` wrapper via `readAuthLiteral` (`record_map.go:606-619`) |
| `(spec.record_read)` + `ValidateRecordReadAnnotations` | `spec.proto:140-178`, `:2357-2386`; `record_read_validate.go:1-60` | the USE half; build fails without the GRANT half and vice versa | — | generation time |
| `C2SScope` (`C2S_SCOPE_NAMESPACE` "declared but NOT IMPLEMENTED") | `spec.proto:2388-2426`; `record_read_validate.go:339-350` | a *scope* enum on a read grant, with a reserved-but-refused value | — | generation time |

Conclusions from the precedent search:

- **The idiom for "who may do X to record type T" is the `(spec.record)` grant block, and it is already the mechanism in use.** Extending `Delegate` would be the wrong layer twice over: `Delegate` names *audiences*, and IDENTITY.md invariant #3 (`S/platform/services/kernel/IDENTITY.md:174`) says "Destination services enforce access policy, not kernel." S2S tokens are deliberately identity-only (`IdentityServiceImpl.java:1648`). Putting a namespace-reach claim on an S2S token would be the first authorization payload on a service token in the platform.
- **The idiom for "how wide is a read" is a scope enum next to the grant** — `C2SScope` on `RecordReadC2S`, `PrincipalScope` on the claim bag. `C2S_SCOPE_NAMESPACE`'s "declared but not implemented, so it cannot be invented twice" stance (`record_read_validate.go:339-350`) is the exact pattern to copy for any new value that is not built yet.
- **Tenant allow-lists are runtime/deployment state, not proto.** The only one that exists (`CredPair.namespaces`) lives in `TenantConfig` in kernel state; tenants (`/tenants/drpepper`) do not exist when a proto is authored, and protos are location-agnostic by design (`SpiffeIds.java:14-16`). A tenant list cannot go in an annotation without breaking that.
- **The audit trail already carries the caller's identity claims per RPC**: `AccountingRecord.identity` is "the claims from the verified token" (`T/pkg/proto/sys/accounting.proto:82-85`), so `sub`/`namespace` of the reading service are recorded wherever accounting is emitted. What is *not* recorded today is *which tenant's rows* an S2S read returned — the predicate is opaque to the audit layer. Not verified whether `RecordMap` RPCs are in stratum's accounting map.

---

## 4. Concrete proposal

Three parts, in the order they should land. Part C is the only new mechanism; A and B are what makes
the existing grant "declared, scoped, auditable" as the problem doc asks.

### A. Put the doctrine on the grant (spec.proto only, no generator change)

On `RecordAnnotation.s2s_read` (`T/pkg/proto/sys/spec.proto:782-838`) add the sentence that today lives
only in `recordnamespace_gen.go:36-42`:

> A service named here reads EVERY tenant's rows of this type. The S2S half of the record query plane
> applies no namespace narrowing — the caller is a peer authorized by identity, and the tenant boundary
> is enforced by the peer's own C2S surface, never by this one. Naming a service here IS a cross-tenant
> read grant; if the peer's need is tenant-scoped, see `s2s_read_scope`.

And on `(spec.record_read)` (`spec.proto:140-178`), the mirror sentence for the reader. This is the
whole of "tribal knowledge sitting in a comment" fixed at its source.

### B. Make the scope visible in `registry.json` (registry emission change only)

1. On every S2S read-plane RPC surface (the `RecordMap` entries), replace the misleading
   `subject_namespace_source: "token.namespace"` with an explicit
   `"tenant_scope": "unscoped"` (or, with Part C, `"tenant_pinned"`). Emitted from
   `namespaceAuthMetadata` (`registry.go:1157-1173`) or its caller at `:1085-1092` — the RPC already knows
   it is a record-plane read (the `readAuth` path in `record_map.go:54-69`).
2. Add a per-record grant summary at the registry top level alongside `entitlement_grants`, e.g.
   `"record_grants": [{ "record": "docmanager.DocFileMetadata", "publish": ["docmanager"], "subscribe": ["stratum","docmanageragent"], "read": ["docmanager","docmanageragent"], "read_scope": "unscoped", "disclose": [] }]`.
   Today an auditor must find the `vservices/query` node and read three RPC entries per record to
   reconstruct this. The operator's registry reader ignores unknown keys (`bootstrap/registry.go:21-30`
   models only the fields it needs), so this is additive for consumers.

### C. One additive per-record option: `s2s_read_scope`

**Shape** — a sibling field on `RecordAnnotation`, not a change to `MethodS2SAnnotation` (which is
shared by `s2s`, `s2s_subscribe` and `(spec.method_s2s)`, where a scope would be meaningless):

```proto
// S2SReadScope names how a PEER's stored-row read of this record type is
// scoped. It is orthogonal to s2s_read.auth_services, which names WHO may
// read; this names WHAT SHAPE a read must take.
enum S2SReadScope {
  // Unspecified = FLEET. Absence must keep meaning today's behaviour, or every
  // record in the fleet changes meaning on upgrade (same rule as Residency).
  S2S_READ_SCOPE_UNSPECIFIED = 0;
  // FLEET: the peer may read rows of every tenant in one predicate. Today's
  // behaviour; what argus needs for AccountingRecord.
  S2S_READ_SCOPE_FLEET = 1;
  // TENANT_PINNED: every read of this type through the S2S plane MUST name
  // exactly one tenant in its predicate. The peer keeps cross-tenant REACH
  // (it may name any tenant) but never performs a cross-tenant SCAN, and the
  // tenant it named is on the wire for every call.
  S2S_READ_SCOPE_TENANT_PINNED = 2;
}

message RecordAnnotation {
  ...
  MethodS2SAnnotation s2s_read = 10;
  MethodS2SAnnotation c2s_read = 11;
  // s2s_read_scope: see S2SReadScope. Only meaningful when s2s_read is
  // declared; declaring it without s2s_read fails generation (decoration rule).
  S2SReadScope s2s_read_scope = 12;
}
```

**What it generates** (where the check lives — answers open question 4 concretely):

1. **Store side, Java, stratum's generated `RecordMap` wrapper** — the S2S twin of the C2S predicate
   pin. Today `javaPredicateNamespacePin` (`recordnamespace_gen.go:260-284`) is emitted only under
   `isC2S` (`servicewrapper_gen.go:1650-1661`) and *stamps* the caller's namespace. For a
   `TENANT_PINNED` record, emit on the **S2S** `Create<T>Query` (and `Update<T>Query`) a *require*
   variant: the predicate's `(spec.store_namespace)` criterion must be present and non-empty, else
   `INVALID_ARGUMENT` ("<Record> is tenant-pinned; a peer read must name the tenant it is reading").
   No stamping — a `/system` caller has no tenant to stamp. Same seam, same helper family, one new
   renderer.
2. **Reader side, Rust, `record_reader_gen.go`** — for a pinned record, `get` and `list` take the
   namespace as a required first parameter and build it into the filter:
   `get(namespace: &str, file_id: &str)`, `list(namespace: &str, filter, page_size, token)`. The
   type system makes "forgot to pin" uncompilable, which is the same "cannot be generated wrong"
   posture `ValidateRecordReadAnnotations` already takes for the grant pair.
3. **Generation-time validation** in `ValidateRecordReadAnnotations`
   (`record_read_validate.go`): `s2s_read_scope` without `s2s_read` is refused; `TENANT_PINNED`
   on a record whose `(spec.store_namespace)` field is not a `(spec.query_field)` EQ criterion is
   refused with the fix in the message (exactly `validateC2SNamespaceCriterion`'s stance,
   `:353-355`) — a pin the store does not evaluate is not a pin.
4. **Registry**: `read_scope: "tenant_pinned"` on the record grant and `tenant_scope: "tenant_pinned"`
   on the RPC surfaces (Part B).
5. **Optionally, accounting**: since the tenant is now a required criterion, the store's wrapper can
   put it in the `RpcCompleted` accounting event, giving per-call "service X read tenant Y's rows of
   type T." Not required for v1.

**What it explicitly does NOT do:**

- It does not touch `verify_token`, `Caller`, the authorizer, `Delegate`, kernel, or the SPIFFE
  grammar. No token carries anything new. The `/system` identity stays exactly what it is.
- It grants nothing. It can only *narrow* an existing `s2s_read`. A record that does not declare it
  behaves byte-for-byte as today.
- It is read-only by construction: it lives on `s2s_read`, and the S2S read plane has no row-mutating
  RPC (`record_map.go:40-42`, `assertNoRowMutatingRPCs`). It cannot become a write or delete
  entitlement, so the "no delete entitlement, ever" ruling
  (`~/workspace/gitlab/seabeam/sonar/backend/scotty-design-record.md:332-338`;
  `D/decisions/service-agent-rpc-split-and-identity-2026-09-07.md:78`) is untouched.
- It is not a tenant allow-list. It says "name one tenant," not "only these tenants" (see §6 for why
  that is a Jody call, and why I recommend against it in v1).
- It is not a blanket "ignore namespace" flag. There is no such flag to add: the S2S plane already
  ignores it, and the proposal is the inverse.

**What it breaks or complicates for existing services:** nothing unless a record opts in. For
`DocFileMetadata` opting in: `docmanager`'s and `docmanageragent`'s generated reader signatures
change (namespace parameter), and both hand-written pins become redundant — `docmanager`'s pin
(`service.rs:314-345`) can stay as a belt-and-braces re-check the same way the `file_id` re-check
does, or be removed. `argus`/`AccountingRecord` must NOT opt in (fleet-wide aggregation is its
purpose). If the default were flipped instead (open question 3), every `s2s_read` consumer's
generated API changes and argus breaks — do not flip.

### What DocManagerAgent does with this (app-side, not tacplus)

1. `docmanager.proto` `DocFileMetadata`: add `s2s_read_scope: S2S_READ_SCOPE_TENANT_PINNED`.
2. `docmanageragent`: remove the `caller.namespace()` comparison in `get_doc`; pass the tenant into
   the generated reader from the **event row's `namespace`** (already delivered to
   `on_stream_docmanager_doc_file_metadata_bus_stream`, stamped by docmanager's generated wrapper from
   the ingesting user's token — a trusted value, never LLM-supplied).
3. `ListDocsRequest.namespace`: the `(spec.namespace)` marker is the wrong binding for a virtual owner
   (it binds to the *service's* namespace). It must not become a tool argument either. The tenant
   has to be bound to the MCP session / `run_loop` invocation. How, is a design call — §6 item 2.

---

## 5. Answers to the five open questions

**Q1. Extend `Delegate`/`authorized_services` with a namespace field, or live near `s2s_read`?**
Near `s2s_read`, and the code gives a hard answer, not a preference:
- `Delegate` is minted-side and names audiences (`kernel.proto:702-706`); S2S tokens are
  "identity-only … authorization is receiver-side" (`IdentityServiceImpl.java:1648`); IDENTITY.md
  invariant #3. A namespace grant on the delegate would need a new S2S token claim, a kernel change,
  an operator YAML change, and a generated receiver-side check — every layer the investigation's §2
  lists for option 3, for something the record plane can express locally.
- The grant is already per-record-per-service on `s2s_read` (`record_map.go:54-69`, `:606-619`) and
  already validated as a pair (`record_read_validate.go`). The only thing missing there is scope.
- **Answered from code.**

**Q2. Per-RPC, per-record-type, or per-field? Tenant allow-list vs. all?**
- Per-record-type. That is the granularity of every other record-plane grant (`s2s`, `s2s_subscribe`,
  `s2s_read`, `c2s_read`, `min_group_size`, `mutability`), the reader is generated per record
  (`record_readers.rs` is one struct per record), and the enforcement point (stratum's per-record
  `RecordMap` RPCs) is per record. Per-RPC would mean per `Create/List/Delete<T>Query`, which are
  one lifecycle, not three decisions. Per-field is the sanitizer's job (`egress_sanitize_gen.go`),
  a different axis. **Answered from code.**
- Allow-list: **not in proto** — tenants are deployment state (`TenantConfig`, `kernel.proto`),
  protos are location-agnostic (`SpiffeIds.java:14-16`), and the only allow-list precedent
  (`CredPair.namespaces`, `TenantAdminServiceImpl.java:545-612`) is a user-type credential in
  kernel state. Whether the platform *needs* a service-level tenant allow-list is a product call
  (§6 item 3); the technical answer is that if it is ever wanted it belongs on the delegate as a
  claim the store's wrapper checks, which is the big change, and nothing today requires it.

**Q3. Keep unscoped-by-default, or flip to scoped-unless-declared?**
Keep the default; flipping is not "safer, but bigger" — it is non-functional. "Scoped to the
caller's namespace" on the S2S half means scoped to `/system`, where no tenant data lives, so every
`s2s_read` consumer in the fleet would read nothing (`recordnamespace_gen.go:36-42` says exactly
this). The only meaningful narrowing is "scoped to a tenant the read names," which is Part C, and
that must be opt-in for the same reason every other record option is additive
(`spec.proto:888-890`, `:721-723`: "absence must keep meaning today's behaviour"). **Answered from
code.** Whether `DocFileMetadata` specifically should opt in is Jody's (§6 item 1).

**Q4. Where does the generated check live?**
Not in `verify_token`/`Caller` (identity, not policy — `Caller::namespace()` just returns the claim,
`servicemanager/src/auth.rs:121`) and not in the authorizer (both authorizers match
`service_path` only and state that namespace is never matched: `authorizer.rs.tmpl:384-388`,
`Authorizer.java.tmpl:449-454`). It lives in the **generated service wrapper of the store** —
the same seam as `javaPredicateNamespacePin` (`recordnamespace_gen.go:260-284`, emitted from
`servicewrapper_gen.go:1650-1661`) — plus the generated Rust reader signature
(`T/pkg/internal/rust/types/record_reader_gen.go`) so the client cannot omit it. **Answered from
code.**

**Q5. Precedent to reuse instead of building parallel?**
Yes: `s2s_read` *is* the grant; `C2SScope` (`spec.proto:2388-2426`) is the shape for a scope enum
beside a read grant, including the "declared but NOT IMPLEMENTED" stance for any value not built
yet (`record_read_validate.go:339-350`); `javaPredicateNamespacePin` is the enforcement renderer to
twin; `ValidateRecordReadAnnotations` is where the validation goes. The one thing that would be
genuinely new is the registry's `record_grants` summary (Part B.2) — and even that mirrors
`entitlement_grants`. **Answered from code.**

---

## 6. Genuine judgment calls for Jody (not technical answers)

1. **Is "tenant-pinned" the right posture for `DocFileMetadata`, or is fleet-wide read acceptable
   for the agent?** The investigation's option 1 ("accept `/system`, stop scoping") is what the
   code already permits, with the boundary enforced by deployment topology. The bootstrap trusts
   two realms (`drpepper`, `sprite` — `L/docmanager/decisions/verify-script-tenancy-model-2026-09-02.md`),
   so one fleet does serve multiple tenants, and the design record says the agent is "singular
   company wide" (§9), which reads as *per company*, i.e. per tenant. Part C makes each agent
   action tenant-scoped without denying it reach; that is my recommendation, but "the agent may
   list every tenant's inbox in one query" is a legitimate product position too.

2. **How the tenant binds to an agent MCP session.** `(spec.namespace)` binds a request field to
   the *caller's* claim, which for `self_caller` is `/system`. A virtual owner needs "the tenant
   this session is about," which must not be an LLM argument (`docmanageragent.proto`,
   `ListDocs` comment; `doc_manager_agent.rs:480-485`). Options: (a) app-level — `run_loop` is
   constructed with the event row's namespace and the handler closes over it (no tacplus change,
   but the tenant boundary is again hand-written in the agent); (b) a tacplus concept — an
   `RPC_PRINCIPAL_MCP` self-orchestration session carrying a declared session-bound namespace the
   generated dispatch stamps instead of the caller's claim. (b) is a new generator feature with no
   second consumer today; by the platform's own "no consumer, refuse rather than guess" rule
   (`record_read_validate.go:345-350`) I would not build it yet, but it is the only way to get the
   agent's tenant boundary out of hand-written code, which is what goal.md line 83 asks for.

3. **Does the platform want a per-service tenant allow-list at all?** Nothing today needs one; the
   deployment (which tenants a fleet serves) is the boundary, and OPA decides namespace only for
   users (`IdentityServiceImpl.java:1247-1252`). Building it means the first authorization payload
   on an S2S token. I recommend explicitly deciding "no, not in v1" and writing that down next to
   `s2s_read`, so it is a decision rather than an absence.

4. **Whether `s2s_read` should be renamed or its wildcard tightened once the doctrine is written on
   it.** `["*"]` on `s2s_read` means "any authenticated peer may read every tenant's rows of this
   type" (`spec.proto:821-823`, `:859-861`). That is sayable, and `c2s_read` already refuses the
   wildcard for the disclosure case. Whether the fleet-wide-read wildcard should also be refused
   (forcing every cross-tenant reader to be named) is a policy call; the code is indifferent.

5. **Whether `subject_namespace_source: "token.namespace"` on S2S read surfaces is a bug to fix
   in place or a field to deprecate.** It is emitted for every authenticated RPC
   (`registry.go:1157-1173`) and is correct for C2S. Replacing it on S2S read surfaces (Part B.1)
   changes a registry field's meaning; any downstream consumer of that field should be checked
   first. I found none outside the registry itself, but did not audit every consumer.

---

## Appendix — side findings outside this task's scope (mention, do not act)

- `D/services/docmanager/src/service.rs:325-336`: the "TEMPORARY backstop" comment about `.get(id)`
  returning an arbitrary row is stale; `DocFileMetadataFilter` now carries `file_id`
  (`query.rs:409-418`). The re-check itself is still worth keeping.
- Unverified: whether the generated Rust C2S namespace backstop covers a *single* record-typed
  response field (`GetDocFileMetadataResponse.metadata`). `namespaceListFields`
  (`namespace_gen.go:53-78`) only walks repeated fields. If uncovered, docmanager's hand-written
  pin is the only tenant boundary on that RPC.
- `record_reader_gen.go:253` and `record_readers.rs:51-53` say the key becomes an EQ criterion
  "because the record declares `s2s_read`"; `docmanager.proto:248-256`'s comment predates that and
  still describes the pre-fold behaviour.
