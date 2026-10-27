# Tenant-owned services — decisions (2026-09-08)

Findings: tenant-owned-service-identity-FINDINGS.md, kernel-state-path-translation-FINDINGS.md (this dir),
and the prior cross-tenant-system-identity-read-access-FINDINGS.md (session 57dd5337 scratchpad).

1. Shared vs tenant split: everything in schematic platform is shared (/system/services/<name>).
   Everything on scotty (and docmanager, its own repo) is tenant-owned, one instance per tenant,
   identity spiffe://<td>/tenants/<tenant>/services/<name>. Operator-managed.
2. Tenant is derived ONLY from the SPIFFE ID in the token. Never a request field, never config
   the service can set.
3. Kernel state is tenant-aware by identity: a tenant-owned caller asks for /data/table, kernel
   rewrites to /tenants/<t>/data/table on ingress and strips on egress (keys, Watch events,
   list results). Service code unchanged, never learns its tenant. Chroot semantics.
   Rule: prefix the map key only, never the payload.
4. DROPPED: S2S_READ_SCOPE_TENANT_PINNED / required-namespace parameter on Rust reader.
   No speculative mechanisms. Kept from prior research: doc the cross-tenant doctrine on the
   s2s_read proto field; make registry.json report real tenant_scope on S2S read surfaces.
5. Canonical namespace vocabulary: /tenants/<name>. Unify OPA user-token form
   (/local/dev/<name>) and TenantAdmin bare form (/<name>) to it.
6. Delegate name stays the service path (docmanager); tenant lives in a new delegate field.
   auth_services grants stay per service name.
7. Kernel topic/queue/leader boundaries: v1 rule, tenant-owned services may not use them
   (vestigial from when kernel was eventbus; eventbus is the real thing). Locks get the prefix.
8. Eventbus S2S subscribe gets the same runtime branch: /tenants/* subscriber filtered to its
   namespace, /system stays fleet-wide.
9. THREE service classes, not two (2026-09-08, after login/identity doc):
   a. Shared platform services: /system/services/<name>, own S2S reach, fleet-wide. (schematic)
   b. Tenant-owned instances: /tenants/<t>/services/<name>, one pod per tenant, act as
      themselves, chrooted by kernel. (docmanager, tenant scotty)
   c. Shared on-behalf-of services: /system/services/<name>, NO S2S reach of their own, act
      only through delegated user tokens (one token per tenant membership). (personal scotty)
      No blast-radius concern: can only do what its current users can do.
   Personal scotty is a SEPARATE deployed service from tenant scotty (Jody, leaning; my
   recommendation: make it a rule). Shared persona-runtime code as a library is fine; the
   split is at the deployed service + proto boundary.
10. Login/identity model: scotty/backend/decisions/login-and-identity-model-2026-09-08.md is
    authoritative. Its §6 items 1-9 are schematic platform work, sequenced after namespace
    vocabulary unification. Personal principals are device-first (synced keychain key,
    Secure Enclave fallback), IdP subject bound later as upgrade. Memberships are kernel
    state on the personal principal; one token per tenant minted via service_auds.

11. Which services are tenant-owned is declared by a service-level proto annotation in
    spec.proto, emitted by tacplus into registry.json. Operator, gateway, store wrapper all read
    it there. Gateway consults the bearer token's namespace for routing ONLY on services so
    flagged; shared services route by name as today.
12. Gateway routing upgrade is known new work, on Goal A's critical path.

Open:
- Topics/queues/leader records/fenced locks in kernel state: per-tenant object names vs v1 rule
  "tenant-owned services may not use topic/queue boundaries". Awaiting Jody.
- Eventbus S2S subscribe is fleet-wide; needs a tenant filter for tenant-owned subscribers.
- Gateway routing of tenant users to their tenant's instance: unverified.
- Delegate-name-keyed kernel paths (findDelegateByName, MintDelegatedToken, RevokeGrant) become
  ambiguous with N delegates named docmanager; key by (name, tenant).
- Kernel service_state sizing: identity agent said large/no seam, state agent found single
  codegen choke point (~20 fragments). Needs human check.
- Prior-research decision 4: refuse s2s_read ["*"] wildcard? Not yet discussed.

## Addendum 2026-09-17 (Jody) — a fourth class, 9d: shared dispatcher

Item 9 placed nothing for `scottyagent`, and none of a/b/c fit it: it HAS its own S2S reach
(subscribes to messaging.MessageBus.stream, takes kernel.FencedLock) like 9a, its rows are PER
USER (AgentCursor keyed by recipient principal), and it holds NO authority (no delegated tokens,
no LLM). A 9c service cannot do the job — it may not subscribe to a bus or take a lock.

   d. Shared dispatcher: /system/services/<name>, own S2S reach, one instance serving every
      tenant and every personal principal, NO authority. Its only job is to turn a fleet-wide
      S2S event into a per-user row that the user's OWN runtime (9c) reads. Rules: acts for
      nobody (never calls a tenant service or scottypersonal, never holds a user token); retains
      no user content (ids, reasons, timestamps only); per-user rows held by
      store_principal/store_namespace + generated egress filters. (scottyagent)

Decided A over: B (make it 9b, one per tenant — personal-principal recipients have no tenant pod,
so a shared instance is needed anyway) and C (retire into the per-user Scotty runtime — that
runtime is back-pocket, and something with reach must still consume the bus).

Two follow-ups filed, not built:
- tacplus: a consumer-scoped topic envelope so a subscriber group can receive
  {delivery_id, recipient, namespace} and never the body. Today scottyagent's process holds every
  tenant's full Message and reads only two fields — discipline, not a control.
- scottypersonal: watch its own principal's AgentCursor/AgentCycle rows (WATCH over pinned rows now
  generates — tacplus 2026-09-17, watch_egress_gen.go) instead of scottyagent poking anyone.
