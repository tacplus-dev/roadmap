# Skipper and Admiral are tacplus platform components (Jody, 2026-11-01)

Jody, 2026-11-01: "Originally skipper and admiral were going to be Seabeam. But I now think they are part of tacplus. Seabeam
may give them a persona but they are necessary for the fleet. There is one skipper per tenant. Skipper will span across
clusters for the same tenant across the same trust domain. Then there is admiral. Admiral sits at the deployment level inside
system and operates across the deployment fleet. Tenants may talk to their skipper but not the admiral. Skippers may talk to
their admiral."

## Decided

- **Skipper and Admiral are tacplus**, built and generated like any platform component. SeaBeam (or any consumer) may give them
  a persona. It doesn't own them. (Producer/consumer: the roles are the platform's; the personality is the product's.)
- **Skipper: one per tenant.** Tenant-owned (`/tenants/<tid>`). It **spans clusters** for that tenant within the same trust
  domain, so a tenant has one Skipper however many clusters its instances run in.
- **Admiral: one per deployment, in `/system`.** It operates across the deployment's fleet.
- **Who may talk to whom:**
  - a tenant (its users and services) **may** talk to **its own** Skipper;
  - a tenant **may not** talk to the Admiral;
  - a Skipper **may** talk to **its** Admiral (tenant → `/system`, an allowed S2S edge,
    `s2s-never-crosses-tenants-2026-10-31.md`).
- **Consistent with earlier decisions:**
  - no tenant-to-tenant S2S;
  - the Admiral, being `/system`, holds **no tenant data** (addendum 2026-11-01);
  - the trace roles proposed on tacplus-dev/tacplus#265 map here: the tenant trace role is the Skipper's, the platform trace role
    is the Admiral's.

## Open (Jody)

- ~~Whether the Admiral may call a Skipper.~~ Settled below.
- What each one does: its responsibilities and API.

## Addendum 2026-11-01: the Skipper dials out, over a bidi channel (Jody)

Jody: the Admiral needs to reach Skippers, "but we might make it so that the skippers reach out to the admiral and maintain a
bidi channel."

- **Every Skipper opens and holds one bidirectional stream to its Admiral.** That's tenant → `/system`, the allowed direction.
  The Admiral uses that channel to reach the Skipper. It never dials into a tenant, so there is still **no `/system` → tenant
  call path**.
- **What that gives:**
  - the Admiral can only reach Skippers that are connected;
  - each message arrives on a channel the tenant's own Skipper opened and authenticated, with its own token;
  - it works however many clusters the Skipper spans.
- **Messages are typed, both ways, on a declared API.** What the Admiral sends is a **request**. The Skipper decides, by its own
  policy, whether and how to act. The Admiral never reaches past the Skipper into the tenant's other services. Every message is
  audited.
- **It's a long-lived stream,** so it uses make-before-break token rotation (tacplus-dev/tacplus#259) and explicit `last`
  signalling. The Skipper reconnects with backoff, and the Admiral treats a Skipper whose channel is down as unreachable, never
  as a reason to dial in.

## Addendum 2026-11-01: the Admiral is LLM-driven; the operator stays separate (Jody)

Jody: "I see the admiral as LLM driven. And because that means we need llmproxy in place which means we need operator it is
good to keep them separate."

- **The Admiral is LLM-driven.** It reaches models only through **llmproxy**, like every model user (no model credential, no
  vendor SDK).
- **The operator stays a separate, deterministic component:** uni-config in, cluster state out, per cluster. It holds the
  infrastructure power: namespaces, Secrets, deployments, RBAC.
- **The Admiral asks; the operator acts.**
  - The Admiral drives the operator only through a **narrow, typed API** (e.g. promote or roll back a canary, pause chaos, roll
    back a service). This is the nsproxy and chaos-wedge pattern.
  - The operator checks every request against the uni-config and its own rules, does only what it was built for, and every call
    is traced and audited.
  - A confused or prompt-injected Admiral can at worst **ask** for something the operator refuses.
- **Bring-up order:** operator → llmproxy (deployed by the operator) → Admiral. Skippers connect to their Admiral once it's up.
- **Prompt-injection hardening applies to the Admiral,** as for loopers (tacplus-dev/looper#40): every message from a Skipper
  is data in nonce-keyed provenance blocks, never instructions. Direction comes only from the Admiral's own configuration and
  its declared API.
- **Open:** whether the Skipper is LLM-driven too.

## Addendum 2026-11-02: GetConfig (Jody)

Jody: "we need to expose a get_config RPC that returns the config for system and for tenants. admiral will call the
get_config for the deployment and skipper for a tenant. I think admiral can also call for a tenants config too." The hub
pointed out that the Admiral calling a tenant directly would cross the 11-01 addendum (no tenant data in `/system`) and
`s2s-never-crosses-tenants-2026-10-31.md`. Jody: "Ok, so right. skipper can get its tenant_config. Admiral can talk to
skipper".

- **`GetConfig` returns a uni-config's redacted twin, never secret values.** Secret version references stay visible.
  The redaction is generated (tacplus#274). Real values reach services only through the deterministic config delivery
  path (`deterministic-deployments-2026-11-02.md`). This is the hub's recommendation, accepted with "Ok, so right".
- **The Admiral** calls `GetConfig` for the **deployment** (the operator serves it, as part of its narrow typed API).
- **A Skipper** calls `GetConfig` for **its own tenant only.** Whatever holds tenant uni-configs serves it; where that is,
  is still the open huddle in `deterministic-deployments-2026-11-02.md`.
- **The Admiral never reads a tenant's config directly.** It asks that tenant's Skipper over the channel the Skipper
  dials out to the Admiral, and the Skipper answers for its tenant. The Admiral doesn't store it.
- **Every `GetConfig` is traced and accounted** like any call.
