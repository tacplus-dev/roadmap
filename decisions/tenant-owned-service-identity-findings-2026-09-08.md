# Tenant-owned service identity (`spiffe://<prefix>/<tenant>/services/<svc>`) — findings

Research only; no files changed. Paths: `P` = `/Users/jody/workspace/gitlab/seabeam/schematic/platform`,
`T` = `/Users/jody/workspace/gitlab/seabeam/schematic/tacplus`,
`D` = `/Users/jody/workspace/gitlab/seabeam/sonar/docmanager`,
`SC` = `/Users/jody/workspace/gitlab/seabeam/sonar/backend`.
Prior research (`cross-tenant-system-identity-read-access-FINDINGS.md`) is taken as established and not re-derived.

**Correction to the brief:** docmanager is NOT in scotty. `SC/services/` holds `scotty` and `scottyagent` only;
docmanager/docmanageragent live in their own repo `D` (`D/services/docmanager`, `D/services/docmanageragent`,
`D/bootstrap/operator.yaml:16-18` declares them as `app_services`). §6 covers `D`; scotty is cited where it
documents the same gap.

---

## Verdict

**There is no code path anywhere in the three repos that handles a non-`/system` service identity.** The
tenant-owned shape exists exactly once, as a table row in `P/todo/MULTI_TENANCY_PLAN.md:21`. Nothing parses,
produces, or tests it. Today every layer would reject it: the SPIFFE grammar (three languages), kernel mint,
`TokenWrapper.servicePath()`, both generated authorizers, and kernel's state-plane authenticator. So the
question "would the kernel reject it, treat it as `/system`, or derive `/<tenant>`" has one answer: **reject**
(UNAUTHENTICATED / IllegalArgumentException), never silently `/system` — details in §1.

Is there a clear path? **Yes for identity + record-plane scoping; no for "kernel confines all state."**

| Piece | Status today | New work | Size |
|---|---|---|---|
| SPIFFE grammar admits `/tenants/<t>/services/<svc>` | Rejected in Rust/Java/Go templates + vectors | tacplus template + cross-test change, byte-identical across langs | Medium (1–2 d) |
| Kernel mints a tenant-namespaced service token | `/system` hardcoded via `SpiffeIds.build` (`""` location) | `Delegate` gains a tenant/namespace field; mint builds URI from it; `findDelegateByName` keyed by full URI | Small (≈1 d incl. tests) |
| Operator: one Deployment + SA + delegate per tenant | Operator is service-keyed, tenant-blind | tenant list in `operator.yaml` (recommended), loop in `enabledServices` + `EnsureServiceDelegates` | Medium (≈2 d) |
| Store-side pin for tenant-namespaced peers (record plane) | Pin renderers exist, gated `isC2S` at generation time | tacplus Java generator: runtime branch on token namespace in the S2S `RecordMap` wrapper; regenerate stratum | Small–Medium (≈1 d); one test asserts the opposite today |
| Kernel `service_state` confined per tenant | **No namespace dimension at all** — maps named per package, authz by service path | New design in kernel's own codegen (map keying or per-tenant map names, Watch scope, policy) | Large; no seam exists |
| Eventbus S2S subscribe scoped per tenant | Topics fleet-wide; watch guard C2S-only | New filter in eventbus `SubscribeHandler` or a generated one | Medium |
| One namespace vocabulary (`/tenants/acme` vs `/acme` vs `/local/dev/acme`) | Three vocabularies coexist | Prerequisite for any exact-match pin to ever match | Cross-cutting; decision first |
| Gateway routes tenant users to the tenant's instance | Not verified; routing is by service name | Unknown, flagged | Unverified |
| Docmanager hand-written pins | Present in 5 handlers | Delete; `(spec.namespace)` on `ListDocsRequest` becomes correct as-is | Small |

The two things most likely to bite are the **namespace vocabulary mismatch** (§5.1) and the **kernel state
plane having no tenant dimension** (§2.3). The design's sentence "the kernel … confines all state reads/writes
to that tenant" is true of the record plane (stratum) after a generator change, and false of kernel-hosted
`service_state` without new kernel work.

---

## 1. Token/identity path

**Where `/system` is stamped — a hardcoded constant, twice.**
- Kernel: `SpiffeIds.build(td, name)` → `Spiffe.serviceSpiffeId(td, "", name)` (`P/services/kernel/server/src/main/java/.../cluster/SpiffeIds.java:31-33`); the Java template appends `SYSTEM_SEGMENT` unconditionally (`T/pkg/templates/java/chassis/spiffe/Spiffe.java.tmpl:300`). Rust twin: `service_spiffe_id` pushes `SYSTEM_SEGMENT` (`T/pkg/templates/rust/chassis/spiffe/src/spiffe.rs.tmpl:288-304`).
- The K8s path: `mintFromKubernetesToken` verifies the projected SA token, maps `(k8s namespace, SA)` → `Delegate` via `findDelegateByKubernetesServiceAccount` (`IdentityServiceImpl.java:465-466, 513-521`), then `spiffeId = SpiffeIds.build(trustDomain, delegate.getName())` (`:473-474`). The subject is derived from the **delegate name only**; neither the SA name, the k8s namespace, nor anything in the token contributes a tenant. `Delegate` has no namespace field (`P/proto/services/kernel.proto:689-713`; `KubernetesServiceAccountMatcher` at `:667-675` is explicitly "the K8s namespace, not an authorization namespace").
- Namespace claim: `newToken.setNamespace(SpiffeIds.namespaceFromSpiffe(subject))` for service mints (`IdentityServiceImpl.java:1383-1386` refresh path, `:1661-1662` S2S exchange path). Users get OPA's namespace or root (`:1247-1252`).

**What a tenant-shaped URI does today, layer by layer:**
- Parser (`Spiffe.java.tmpl:476-497`, `spiffe.rs.tmpl:467-484`): `SubjectKind.SERVICE` requires `segs[0] == "system"`. Both `spiffe://td/acme/services/docmanager` (plan's shape) and `spiffe://td/tenants/acme/services/docmanager` fall to the final `else` → `UNKNOWN`, namespace = whole path.
- `namespaceFromSpiffe` → `normalizeNamespace` (`Spiffe.java.tmpl:559-562`, `:189-222`): `/acme/...` hits the `default:` throw ("must start with system or tenants"); `/tenants/acme/services/...` throws at `:203-205` ("only workspaces may follow a tenant"). So the kernel's own `namespaceFromSpiffe(subject)` at `:1662` would throw IllegalArgumentException.
- `servicePathFromSpiffe` throws for non-SERVICE/WORKLOAD (`Spiffe.java.tmpl:550-556`; Rust `:537-543`). Consequences: `findDelegateByName` returns `null` (`IdentityServiceImpl.java:2027-2056`) → `isS2S=false` → the exchange path treats the caller as a **user** (`:1569`, `:1700+`); every receiver's authorizer fails `service_path()` with UNAUTHENTICATED for `sub_type=service` (`T/pkg/templates/rust/chassis/grpcserver/src/authorizer.rs.tmpl:389-393`; Java `Authorizer.java.tmpl:456-464`); kernel's state-plane `SecurityAuthenticator` throws UNAUTHENTICATED (`P/services/kernel/codegen/src/main/resources/templates/SecurityAuthenticator.java.template:401-409`).
- Nothing constructs it: the operator only builds the kernel's own URI (`P/services/operator/pkg/bootstrap/deploycrypto.go:377-385`, via `wrapperSpiffe.ServiceSpiffeID(td, "", "kernel")`); service pods carry no SPIFFE cert — the only URI SAN emitted is on kernel node certs (`deploycrypto.go:161-177, 471-476`). Service identity is the K8s SA token, not a cert.

**Vectors/tests:** `T/cross-test/fixtures/spiffe_vectors.json` has only `/system/.../services/...` service ids (`:9-14, 48-55, 73-74, 89-90`) and `/tenants/acme/workspaces/dev/{users,data}` for tenant subtrees (`:60-70`). `MintServiceTokenKubernetesTest.java:138` asserts `at.getSubject() == delegateSpiffe(DELEGATE_NAME)` (the `/system` shape). The Rust authorizer tests do mint `sub_type=service` tokens with `namespace=/tenants/acme` (`authorizer.rs.tmpl:832-842`) — which only shows the authorizer never looks at namespace (`:385-388`), not that a tenant service identity is supported.

**Operator/trust-domain wrinkle:** `operator.Config.spiffe_prefix` defaults to `local/dev` (`P/proto/services/operator.proto:270-277`) but is only consumed by kmaker (`deploy.go:541-544`); kernel's `spiffe_trust_domain` defaults to `local.dev` (`kernel.proto:221-229`) and `location_path` is separate (`:231-237`). The plan's `spiffe://local/dev/tenantA/...` examples don't match either constant.

**What changes for the design:** (a) grammar: admit `/tenants/<t>/services/<svc>[/workloads/...]` as `SERVICE` with namespace `/tenants/<t>` — in `classify`/`classify_system` and `validate_namespace_segments` in all three language templates (+ the three duplicate Rust copies under `spiffe/`, `servicemanager/`, `grpcserver/` noted in prior findings) and the vectors; (b) kernel: `Delegate` gains `tenant_namespace` (or similar), `SpiffeIds.build` takes it, `findDelegateByName` must key by full SPIFFE (name alone is now ambiguous across tenants — `:2027-2056`); (c) nothing in the tacplus auth interceptors changes: they match `service_path` (`docmanager`), which stays location-agnostic (`authorizer.rs.tmpl:384-388`), so `auth_services` grants keep working unchanged for every tenant instance.

---

## 2. State scoping by identity

### 2.1 Where the C2S pin lives (record plane, stratum)
All four seams are gated on the **generation-time** flag `isC2S` (= service has `service_c2s`, `T/pkg/internal/java/types/servicewrapper_gen.go:35, 764`):
- page filter `:1515, 1525-1530`; predicate pin `:1650-1661` (`javaPredicateNamespacePin`, `recordnamespace_gen.go:260-284`, reads `Auth.verifiedTokenFromContext().getNamespace()`); unary filter `:1708-1725`; watch guard `:2191-2202`.
- Doctrine: C2S ONLY on purpose (`recordnamespace_gen.go:36-42`). Pinned by test: `TestJavaNamespaceFilterIsC2SOnly` (`recordnamespace_render_test.go:226-231`) fails if the S2S wrapper carries any namespace filter.
- The S2S half is `query.RecordMap`; its audience is per-record `s2s_read.auth_services` (`T/pkg/internal/proto/record_map.go:54-69, 606-619`).

### 2.2 What must change for a tenant-owned service token to get the C2S-style pin
A **tacplus generator change only**, then regenerate stratum; no hand-written kernel/stratum code. In the S2S `RecordMap` wrapper, emit the same four renderers under a **runtime** condition instead of `isC2S`: resolve the caller's namespace claim; if it is not under `/system` (i.e. a tenant-namespaced peer), apply predicate pin + page/unary/watch filters; a `/system` peer (argus, stratum) keeps today's fleet-wide read. Concretely: change the four `if isC2S {` gates at `servicewrapper_gen.go:1515/1525/1650/1708/1721/2191` to `isC2S || recordPlaneS2S`, and have the rendered Java branch on `__nsTok.getNamespace().startsWith("/system")`. `javaPredicateNamespacePin` already does stamp-or-refuse on the query predicate (`recordnamespace_gen.go:266-283`), and `s2s_read` already folds the key into the filter (`record_map.go:236-245`), so a tenant instance's `get(file_id)` becomes `file_id EQ ∧ namespace EQ /tenants/acme` with no reader-side change — which is exactly "service code accesses state the same way; only the store knows." Update `TestJavaNamespaceFilterIsC2SOnly` to assert the `/system`-peer branch instead. Estimate ≈1 day. The Rust `(spec.namespace)` request stamp is already **not** C2S-gated (`T/pkg/internal/rust/types/servicewrapper_gen.go:1615-1622`, `namespace_gen.go:93-108`), so a tenant instance's own RPC requests bind to `/tenants/acme` with no change.

### 2.3 Kernel-hosted `service_state` — no seam exists
This is the part the design under-states. Kernel state (scotty declares `(spec.service_state) = {label: ["kernel"]}` at `SC/proto/services/scotty.proto:134` and `scottyagent.proto:255`; docmanager declares none) is:
- named per package with no tenant segment: `{parentDir}.{package}.{instance}.{Field}` (`P/services/kernel/codegen/src/main/java/.../BoundaryAnalyzer.java:133-135`);
- authorized by exact service-path match against `rw`/`ro`/`boundaryGrants` (`SecurityAuthenticator.java.template:205-238, 242-255`), derived from `service:<package>` + `state_import` importers (`T/pkg/internal/proto/libs_protos.go:480-560`, `state.go:236-244`);
- the only per-caller keying precedent is `instance_keyed` (`T/pkg/proto/sys/spec.proto:1731-1734`: key = `service_id@instance_id`).
Two per-tenant docmanager instances (same service path) would therefore read/write **one shared** state root, and `checkAccess` cannot tell them apart. Confining this per tenant means a new dimension in kernel's own codegen (namespace-qualified map names or key prefixing from `CALLER` namespace, Watch scope, persistence registration) — a kernel change, not a tacplus one, and not small. Either scope the design to the record plane (docmanager has no `service_state`, so this is viable for docmanager specifically) or plan the kernel work explicitly.

---

## 3. What exists vs. what is aspirational

- The shape `spiffe://${spiffe_prefix}/${tenant_name}/services/${service_name}` appears only at `P/todo/MULTI_TENANCY_PLAN.md:21`. A repo-wide grep for `tenant_name}/services`, `<tenant>/services`, `/tenants/<x>/services`, "tenant-owned service" across `P`, `T`, `SC` finds nothing else (code, tests, vectors, protos, rego).
- No code path handles a non-`/system` service identity (§1). No tests exercise one.
- The plan is partly stale relative to code: it calls for `ClusterAdmin` + `TenantAdmin` with cred-pair-signed envelopes and `spec.scope` (`MULTI_TENANCY_PLAN.md:120-215, 290-300`); the kernel today has a single C2S `TenantAdmin` gated by `target_namespace_field` (`kernel.proto:1242-1300`), tenant namespaces like `/acme` (`:1256`), and no `ClusterAdmin`. The plan's row 21 shape lacks the grammar's mandatory `/tenants/` segment (`spiffe.rs.tmpl:220-245`). Its component table says "Operator … Tenant-aware? No" and "The operator never talks to kernel about tenants" (`:34, 62`) — directly in tension with running one Deployment per tenant.
- The closest related implemented concept is `Token.namespace` on **user** tokens set by OPA (`IdentityServiceImpl.java:1247-1252`) and the C2S egress/predicate pins in §2.1; the scotty protos document the exact gap this design would close: an s2s publish is stamped with the *service's* namespace, not the user's (`SC/proto/services/scottyagent.proto:158-160, 395-411`; eventbus `PublishHandler.java:326-359`).

---

## 4. Operator side

**Today:** services are enumerated by name; each app service gets one `serviceSpec{Name, Replicas}` (`P/services/operator/pkg/helm/generator.go:780-795`), `withKernelServiceAuth` sets `ServiceAccountName = svc.Name` + a projected kernel-audience token (`:575-600`), one `ServiceAccount` per spec (`:1201-1210`), Deployments named `<name><i>` (`:1119-1145`, `ExpectedDeploymentNames :835-845`). Delegates are ensured per deployed service with `authorized_services` from the registry graph and a `(k8s ns, sa=name)` matcher (`pkg/bootstrap/example.go:510-600`, `services.go:86-95`). App services enter via `Topology.app_services` (`operator.proto:124-138`; `RegisterAppServices`/`AppServiceNames`, `services.go:40-62`).

**Minimal per-tenant change:** for each (tenant, tenant-owned service): a `serviceSpec` named e.g. `docmanager-acme` with `ServiceAccountName: docmanager-acme`, a kernel delegate `{name: docmanager, tenant_namespace: /tenants/acme, kubernetes_service_accounts: [{ns, docmanager-acme}]}` (delegate *name* must stay `docmanager` because it is the service path the authorizers and `authorized_services` match on — `authorizer.rs.tmpl:384-388`, `SecurityAuthenticator.java.template:242-255`; `normalize_segment` forbids `/` in a name so `acme/docmanager` is not an option). No per-tenant SPIFFE SAN is needed: services authenticate by SA token, not cert (§1). `EnsureServiceDelegates`' `existing[d.Name]` map (`example.go:529-532`) must be keyed by (name, tenant). Helm generation is the bigger edit: the image ref, config bundle (`buildSvc`, `deploy.go:190-200`) and bootstrap YAML are keyed by service name.

**Tenant source of truth — options:**
1. **`operator.yaml` list** (e.g. `topology.tenant_services: {docmanager: [acme, beta]}` or a top-level `tenants:` list). Fits every existing pattern: the operator is "the single source of truth for cluster configuration" (`P/todo/k8s_service_account_auth.md:60-62`); `app_services` is already exactly a config-declared, name-keyed service list (`operator.proto:124-138`); OPA's demo policy also hard-codes tenants (`P/mock/app/operator.yaml:24` `tenant_namespace := {"root":…, "drpepper":…, "sprite":…}`). **Recommended.** Cost: tenants become deploy-time, and the `operator.yaml` list must agree with kernel's `TenantAdmin` tree by convention.
2. **Kernel `TenantAdmin.ListTenants`** (`kernel.proto:1271-1277`). The operator has a kernel identity and already pushes OPA config through kernel RPC (`services.go:16-22`), so the plumbing precedent exists — but `ListTenants` is `RPC_PRINCIPAL_USER` + `AUTH_ENTITLED_USERS`, so an S2S operator token cannot call it today; and it inverts the plan's bootstrap order (tenants exist only after kernel is up, yet the operator renders the chart before). Would need a new S2S read RPC and a two-phase reconcile.
3. **A k8s CR/ConfigMap.** No precedent — the operator reads nothing from the cluster today except what it applies.

Size: ≈2 days for option 1 across `services.go`, `example.go`, `helm/generator.go`, `deploy.go`, plus `operator.proto`.

---

## 5. Risks / things that break

1. **Three namespace vocabularies, exact-match pins.** SPIFFE-derived service namespaces would be `/tenants/acme` (grammar, `spiffe.rs.tmpl:220-245`). User tokens carry OPA's value — `/local/dev/drpepper` in the mock (`P/mock/app/operator.yaml:24`, `IdentityServiceImpl.java:1240-1252`). `TenantAdmin` uses `/acme` (`kernel.proto:1256`, `TenantAdminServiceImpl.java:73-77` only checks a leading `/`). The pins are **plain string equality** (`recordnamespace_gen.go:20-25`; Rust `namespace_gen.go:14-18`), so rows written under a user's `/local/dev/acme` would never match a service instance reading as `/tenants/acme`, and a mismatch that reaches the egress backstop **aborts the JVM** (`recordnamespace_gen.go:26-33`). One vocabulary must be chosen (and OPA/TenantAdmin/grammar aligned) before any of this ships.
2. **Kernel `service_state` is shared across tenant instances** (§2.3) — silent cross-tenant state, not a rejection.
3. **Eventbus S2S subscribe is fleet-wide.** The watch guard is C2S-only (`servicewrapper_gen.go:2191`); `SubscribeHandler` filters nothing by namespace (only binds consumer groups to `callerSub`, `SubscribeHandler.java:142-147, 277-290`). Every tenant instance receives every tenant's `DocFileMetadata`/`AgentCursor` events. Consumer groups won't collide (distinct subs), but data exposure is unchanged. Conversely, publishes get the right stamp for free (`PublishHandler.java:338-359`), closing the scottyagent gap.
4. **`findDelegateByName` and grant/continuity paths key on delegate name** (`IdentityServiceImpl.java:2027-2056`; `MintDelegatedToken :809-841`; `RevokeGrant :2649-2666`; continuity cross-check `:485-490`). With N delegates named `docmanager` these must key on the full SPIFFE subject.
5. **Rate limiting** buckets by service path (`RateLimitInterceptor.java:198`) — all tenant instances share `docmanager`'s bucket. Minor.
6. **`target_namespace_field` level-or-below** (`authorizer.rs.tmpl:216-249`, `Authorizer.java.tmpl:385-410, 488, 507`): a `/tenants/acme` service can only reach targets under `/tenants/acme` — correct for this design, but it means a tenant instance can never call any `/system`-scoped administrative RPC that a `/system` instance could. Check docmanager's outbound calls for any such RPC (none found in `D/proto`).
7. **OPA rego** (`P/mock/app/operator.yaml:24`) only maps users → namespaces; it never sees service identities, so no rego rule assumes `/system`. No hand-written `/system` checks were found in eventbus, messaging, mailbox, gateway, stratum, or kernel main code (grep of `"/system` in non-generated sources hit only tests and the `SpiffeIds` comment). argus has no `src/` in the platform tree (deleted per scotty notes).
8. **Gateway routing** — unverified. C2S users of tenant acme must reach `docmanager-acme`; the REST/gRPC route is `{root}/{package}/{service}/{method}` by service name (`T/roadmap/DATA_ENTITLEMENT.md:19-20`). Tenant-aware routing from the user token's namespace is new work whose size I did not measure.
9. **Registry/`auth_services` are per service name**, so nothing distinguishes tenant instances at the authorization layer; any cross-tenant call between two tenant instances of the same service is allowed by `auth_services` and stopped only by the namespace pin. That is by design here, but it is the whole boundary.

---

## 6. Docmanager itself

- Hand-written namespace pins that this design makes redundant (the store's generated predicate pin would enforce the same equality): `get_doc_file_metadata` (`D/services/docmanager/src/service.rs:314-344`), `update_doc_file_metadata` (`:392-415`), `update_doc_file_location` (`:452-475`), `delete_doc_file_metadata` (`:502-520`), and agent `get_doc` (`D/services/docmanageragent/src/service.rs:72-101`). Under the design, each `.get(&req.file_id)` returns only the tenant's row or `None`, so the handlers reduce to `Some(m) if !m.deleted` / `not_found`. The "same not_found for both cases" reasoning (`:305-313`) stays true by construction.
- The **`TEMPORARY backstop` comment (`service.rs:325-336`) is stale** independent of this design: `s2s_read` folds `file_id` into the filter (prior findings §2b.6; `record_map.go:236-245`). The `file_id ==` re-check can go with it.
- `ListDocsRequest.namespace (spec.namespace)` (`D/proto/services/docmanageragent.proto:193`) and `ListDocFileMetadataRequest.namespace` (`docmanager.proto:456`): the Rust stamp binds them to the caller's claim (`namespace_gen.go:93-108`); with the agent's self-caller now carrying `/tenants/acme` they become correct with **no change** — the prior findings' "tenant must come from somewhere else" problem (§6 item 2 there) dissolves. Nothing to delete; the "not LLM-suppliable" property is preserved.
- `CreateIngestRequest.namespace (spec.namespace)` (`docmanager.proto:424`) is C2S-user-bound and unchanged.
- `DocFileMetadata.namespace` is already `(spec.store_namespace)` + `QUERY_OP_EQ` (`docmanager.proto:257-261`), which is the precondition `validateC2SNamespaceCriterion` demands for a pin to be real — so the record needs no proto change.
- Net: ≈100 lines deleted across two services, zero proto edits, and the tenant boundary moves from five hand-written comparisons to one generated predicate pin in stratum.
