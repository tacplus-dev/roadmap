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

## Addendum 2026-11-18: a user's bug report reaches a fix within an hour (Jody)

Jody: "A user (human or ai) may report a bug to skipper. Skipper will validate and report to Admiral. Admiral may report it to
the appropriate repo. Once fixed the cluster will automatically update. This way the user reporting an issue to resolution will
be within an hour."

- **The path:** user (a person or an agent) → **its tenant's Skipper**, which **validates** the report (reproduces it, attaches
  the evidence: trace, logs, versions; drops duplicates and non-bugs) → **the Admiral**, over the Skipper's bidi channel (the
  allowed tenant → `/system` direction) → the Admiral **files it in the repository that owns the cause** (one issue per cause,
  with the evidence) → the looper fixes and lands it → a release → **the cluster updates itself** from it, with no person in
  the loop outside production.
- ~~**The target: report to resolution within an hour.**~~ Superseded the same day (below): no fixed resolution time; the Skipper
  keeps the user updated instead. Fast lands still shorten the path (tacplus#812, #841, #842, #843, #340).
- **Consistent with what is decided:** the Admiral holds no tenant data (the report it files carries the Skipper's validated
  evidence, not the tenant's data); a tenant never talks to the Admiral directly; the Admiral asks and pipelines act (it files an
  issue, it never changes a repository or a cluster itself); deploys only through deterministic pipelines.
- **Open (Jody):** how the Admiral files across organizations (a tacplus-dev repository vs a tenant's or a product's own); the
  report's schema and what "validated" must prove; what the user sees while it is being fixed (status back down the same path);
  the automatic update's rollout rules (canary, rollback) outside production, and that production keeps Jody's approval.

Jody, 2026-11-18, on the same path: "Ideally we will get most of the tacplus bugs so user facing bugs are more likely to be in a
services user code. And technically skipper runs in the tenant namespace and has no privileges in system. So this will be a
joint venture between skipper and admiral."

- **Where the bugs are expected:** the platform's own bugs are meant to be found before users see them (the gates, the nightly,
  the loopers). A user-facing bug is then most likely in a **service's user code** (the tenant's or product's own logic, not the
  generated chassis), so the common case is routing it to the repository that owns that service, not to tacplus.
- **A joint venture, by privilege:** the Skipper runs in the **tenant's namespace** and holds **no privileges in `/system`**; the
  Admiral is in `/system` and sees the deployment, not the tenant's data. So neither does it alone:
  - the **Skipper** validates from inside the tenant: it reproduces the report against the tenant's services, gathers what only
    the tenant may see (its traces, logs, request), and decides what of it may leave the tenant;
  - the **Admiral** adds what only `/system` sees (versions, the platform components on the path, fleet-wide signals such as
    the same failure in other tenants), decides where the cause lives (a service's own repository, or tacplus), and files there;
  - the two exchange this over the Skipper's channel, as typed requests, each acting only within its own privileges.

Jody, 2026-11-18: "And 1 hour is too tight. Instead skipper keeps the user updated on the progress and findings."

- **No resolution-time promise.** The one-hour target is dropped.
- **The Skipper owns the user's side of the report:** it acknowledges it, and keeps the user updated on progress and findings
  (validated or not, where the cause was found, the issue it became, the fix landing, the update reaching the tenant) until it
  is resolved or closed. The Admiral reports each step back to the Skipper over the same channel; the user never talks to the
  Admiral.

Jody, 2026-11-18: "Of course skipper has contextual awareness of the audience. So don't get too technical unless it's appropriate."

- **The Skipper speaks to its audience.** The same finding is told differently to a non-technical end user (what happened, what
  is being done, what to expect), to a developer or operator of the tenant (the cause, the issue, the fix), and to an agent
  (structured, complete). The Skipper chooses the level from who reported it and how they asked; it goes technical only when
  that audience wants it. The record the Admiral files stays fully technical either way.

## Addendum 2026-11-19: open source, Skipper v1, tenant config, the LLM rule (Jody, huddle with the hub)

- **Skipper and Admiral are open source** (tacplus-dev, like every platform component). What a consumer such as
  SeaBeam wants kept private (prompts, playbooks, product knowledge) lives as configuration in its own private repo,
  which the open Skipper loads; the mechanism is never forked.
- **Skipper v1 is tickets plus assisted tenant configuration**, both on the user's authority. Skipper holds no privilege
  of its own: a config change is applied with the authorized user's on-behalf-of token, scoped to that one write and
  short-lived (never stored past the request). Tickets do not depend on the config work and come first.
- **Tenant existence is system data; tenant settings are tenant data.** The tenant registry (a tenant's id, namespace and
  the trust its workloads run under) lives in the system tenant's own Stratum, written by the operator when a tenant is
  approved (one versioned registry write, not a deploy). A tenant's settings live in that tenant's Stratum, versioned
  with a redacted twin (kernel#35). The kernel holds only its pipeline config plus the tenant registry; a tenant's
  workloads get their identity from its namespace and the registry, so the kernel can serve a tenant before it has any
  of that tenant's settings. The tenant files in the fleet repo (`deploy/bootstrap/tenants/<name>/`) are wrong under
  this and go.
- **Tenant config reaches the kernel by push, tenant to system.** The kernel declares the slice of tenant config it needs
  (a tenant-scope `config_slice`: sign-in providers, join mode, domains, durations); tacplus generates a publisher on
  the tenant's Stratum side that holds one stream to the kernel, sends a full snapshot on connect and then each new
  version. The kernel keeps it in memory only; after a restart every tenant re-sends. Fail closed: until a tenant's
  snapshot arrives it has no sign-in providers. The kernel accepts a slice only from that tenant's own Stratum identity
  and only at a higher version than it holds.
- **Skipper's config writes go to the tenant's Stratum** (not to the kernel, which learns of them through the push), as
  per-field writes, not the whole-replace `UpdateTenantConfig` of today.
- **Platform rule: an LLM may propose; it never holds a token or a write path.** Skipper is one service and one binary
  with a deterministic core and an LLM advisor. The core alone writes, and works with no LLM at all (a user or UI can call
  its typed config API directly). The advisor's tools are read-only plus "propose", producing a typed `ConfigChange`
  that becomes a write only after the core validates it against the schema and the user's rights, shows the diff and
  gets the user's explicit approval with their token. Enforced in the build: the advisor's module gets no state client
  and no token handle, and a gate fails the build if it imports one. This applies platform-wide (the Admiral and future
  agents).
- **The operator and the Admiral stay separate services.** The operator holds standing cluster privilege, runs per
  cluster, must work with nothing else up (it deploys llmproxy, which the Admiral needs), and stays small, deterministic
  and LLM-free. The Admiral, per deployment, is where untrusted text (Skippers' messages, users' reports) arrives, so a
  prompt injection must land in a process with no cluster credentials. The Admiral asks through the operator's narrow
  typed API; the operator authorizes and logs every request.
- **Follow-ups:** kernel#35 (tenant config in the tenant's Stratum), operator#17, operator#19 and fleet-seatrials#28
  re-scoped to this; new: the tenant registry in system Stratum, the generated tenant-to-kernel config push, the
  per-field tenant config write, the LLM-rule build gate, and the `tacplus-dev/skipper` and `tacplus-dev/admiral`
  repositories.

## Addendum 2026-11-19 (later): the report contract and where reports go (Jody, huddle 2)

- **Every report surfaces.** Validation sets a report's label and priority, never whether it is filed. A reproduced
  report is filed as a bug; one not yet reproduced is filed as "not yet reproduced" with what the Skipper tried and saw.
  The Skipper never tells a user "everything is fine" on its own judgment: it says what it checked, what it could not
  confirm yet, and that the report is filed and watched, and it keeps the user updated until it is closed. The Admiral
  groups unreproduced reports by symptom across tenants (the symptom only, no tenant data); several of the same
  escalate as a bug of their own.
- **"Validated" means:** a minimal reproduction against the tenant's own services with the tenant's own access, re-run;
  the evidence (requests, responses, trace ids, log lines, versions), secrets redacted and no other tenant's data; and a
  classification (the tenant's code, a platform service, the generator) with a confidence.
- **The Admiral files; nothing goes back to the tenant.** Tenants are not expected to have a ticketing system. The
  Admiral triages and opens the issue itself: with the owning tacplus-dev service when the cause is in tacplus, and in a
  private GitLab repository otherwise. Its deterministic core holds the credentials (a GitHub App with issues: write on
  tacplus-dev; WIF/OIDC to the allowed GitLab projects) and files only what the LLM proposed after checking the target
  against an allow-list, the body for secrets and tenant identifiers, and duplicates (the LLM rule above).
- **For now: every report's full record goes into a private GitLab triage repository first.** When the cause is in
  tacplus, the Admiral also opens a public tacplus-dev issue holding only sanitized evidence (trace data is a typed
  tacplus message, so its sanitizing is generated and deterministic) and a reproduction from generated fixtures, linked
  to the private record by id only. Tenant data never reaches a public repository.
- **Considered and set aside for now:** an encrypted evidence blob in the public issue (each repository publishes a
  public key; its looper decrypts through a KMS key that never leaves KMS; the blob in its own comment, deleted when the
  issue closes). It works, but public ciphertext is permanent (a later key leak opens every past report), so the
  private triage repository wins for now.
- **REVISIT (Jody): a reporting service is the long-term answer.** A tacplus service that takes reports (from Skippers,
  and later from others), holds their records, and is where the private evidence lives, instead of a GitLab repository
  standing in for one. Come back to this once tickets work end to end.
