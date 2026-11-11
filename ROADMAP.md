# Roadmap: tacplus-dev (platform)

Owned by Jody; changed only by an agent at Jody's request. Loopers propose in PROPOSALS.md.

This file covers what spans repositories or has no repository yet. Each repository's own direction is in its
`tacplus-dev/<repo>:ROADMAP.md`. tacplus-dev must work by itself, without SeaBeam.

## Now

- **tacplus 2.6.x is the current line** (v2.6.0 2026-11-05, latest patch v2.6.6 2026-11-10; the older lines
  still get patches). The services declare the **2.6 floor** in `tacplus.yaml`, and those the fleets deploy have a `v0.1.x` source tag;
  `tacplus fleet build` on 2.6 locks each release to its published image, which most don't have yet (Next 2).
- **fleet-seatrials is not deployed now**: its old clusters are gone (`seatrials` 2026-10-29, `tacplus`
  2026-11-05). Its GitLab deploy and test pipeline targets tacplus-dev/cloud's `dev` cluster in seabeam.net
  (tacplus-dev/fleet-seatrials#39); the first deploy there waits on the pins (tacplus-dev/fleet-seatrials#1, Next 2
  and 3) and the move to `tac-seatrials` (tacplus-dev/fleet-seatrials#4). When it was live, sign-in worked with
  Google (routed to `alpha`), Microsoft (routed to `beta`) and Apple.
- **The loopers** run on the tacplus-dev GitLab group (`gitlab.com/tacplus-dev/looper`): **one job per issue**, a
  report working one `approved` issue on its own branch, up to each repository's cap, and a manager per repository
  that alone merges to main on green gates (`decisions/looper-job-per-issue-2026-11-03.md`;
  `tacplus-dev/looper:ROADMAP.md`).
- Planning files in every repository: `decisions/repo-roadmap-files-2026-10-27.md`.

## Next
- **Skipper and Admiral as tacplus platform components** (high; moved from SeaBeam): survey, huddle, then the skipper and admiral repos (`items/skipper-admiral.md`, `decisions/skipper-and-admiral-2026-11-01.md`) (Jody, 2026-11-01)

In priority order.

1. **Stratum: per tenant, in memory, durable through the redo service** (highest priority;
   `decisions/stratum-per-tenant-2026-10-25.md`). Done: the records write surface (tacplus-dev/tacplus#163),
   routing by target tenant (tacplus-dev/tacplus#170) and per-tenant, in-memory Stratum (tacplus-dev/stratum#2), all
   2026-10-28. The critical path from here:
   1. tacplus-dev/redo#1: the redo service, **a `/system` service** (Jody, 2026-11-07, on the issue): every tenant's
      stratum calls it, and it keeps each tenant's log apart by the caller's tid, never calling into a tenant. Not
      tenant-owned and not in the `restore` tenant; `pinned_tenant` and `RPC_FLAG_ANY_TENANT` go
      (tacplus-dev/tacplus#258). Its local step is on main; the second-site copy is parked on tacplus-dev/tacplus#204
      (cross-deployment, Later). `/system` stays the platform only (`decisions/products-layer-2026-11-08.md`).
   2. tacplus-dev/operator#2: render per-tenant Stratum and redo, redo as one `/system` instance; waits on
      tacplus-dev/stratum#26 and tacplus-dev/kernel#34 (each tenant's Stratum seed members).
   3. tacplus-dev/people#1 (done, 2026-10-29) and tacplus-dev/kernel#2: the identity graph stays in kernel as
      `scope: FLEET` roots; kernel#2 waits on tacplus-dev/tacplus#206 (flock state).
   4. tacplus-dev/letsencrypt#2: certificates written to Stratum directly (done, 2026-11-03).
   5. tacplus-dev/fleet-seatrials#3: seatrials adopts all of it.
   Roadmaps: `tacplus-dev/tacplus:ROADMAP.md`, `tacplus-dev/redo:ROADMAP.md`, `tacplus-dev/stratum:ROADMAP.md`,
   `tacplus-dev/people:ROADMAP.md`.
2. **Service releases**: every service tags `vMAJOR.MINOR.PATCH` (from `v0.1.0`, `0.x` until 2026-12-01) and
   publishes its image ("Release: ..." on each repo, e.g. tacplus-dev/kernel#7). Every service the fleets deploy has its `v0.1.x`
   source tag (Jody, 2026-11-07). What releases wait on now is the image: a tag-triggered `release.yml`
   publishes it to ghcr, public (`decisions/ci-on-gitlab-2026-10-27.md`, 2026-11-07 addendum; llmproxy is done;
   kmaker is retired, 2026-11-10). tacplus-dev/gateway#6 is first (priority high, seatrials' first deploy needs it), then the other open
   "Release:" issues (kernel#7, oauth#5, opa#2, eventbus#2, stratum#3, redo#2, doppelganger#2, tenantecho#2, coop#2,
   pigeon#2). tacplus-dev/looper#17 can take releases over later.
3. **Fleets pin by release line** (MAJOR.MINOR → latest patch, image by digest): done in tacplus
   (tacplus-dev/tacplus#180) and the fleets (tacplus-dev/fleet-seatrials#8, tacplus-dev/fleet-dev#2). **The pins**
   themselves, each fleet on tacplus 2.6 with every service at `0.1` (tacplus-dev/fleet-seatrials#1,
   tacplus-dev/fleet-dev#1), wait on those images (2).
4. **fleet-dev as the test fleet**: every service, an ephemeral cluster per run, a test IdP, end-to-end tests on
   every release: tacplus-dev/fleet-dev#3 (`tacplus-dev/fleet-dev:ROADMAP.md`).
5. **Tenant deletion is complete** (`decisions/tenant-deletion-2026-10-27.md`): tacplus-dev/oauth#11 (providers
   leave nothing behind). Done: tacplus-dev/kernel#19 (no new AT/ST, memberships removed, XIDs kept) and
   tacplus-dev/kernel#20 (age out XIDs idle for about a year).
6. **Several deployments per cluster** (`decisions/deployments-per-cluster-2026-10-26.md`):
   tacplus-dev/operator#3 is done (`tac-<deployment>` and `tac-<deployment>-<tid>` namespaces, tenant namespaces
   through deploymentproxy, formerly nsproxy); next tacplus-dev/fleet-seatrials#4 (seatrials moves, with a runbook).
   `tacplus-dev/operator:ROADMAP.md`, `tacplus-dev/deploymentproxy:ROADMAP.md` (renamed 2026-11-02, `decisions/deploymentproxy-2026-11-02.md`).

- **The cluster pipeline** (tacplus-dev/cluster#1, built 2026-11-02): the only holder of cluster-admin on `tacplus`. It installs deploymentproxy,
  creates each deployment's namespace and namespace-only roles, runs `plan` on every change, and leaves `apply` to Jody
  alone. It's on the critical path to tacplus.dev, ahead of fleet-seatrials#4 (`decisions/cluster-pipeline-2026-11-02.md`;
  approved by Jody 2026-11-02, roadmap#1). Agent, at Jody's request.

## Later
- **Native-messaging C2D transport** (Jody, 2026-11-03, after tacplus.dev is up): browser extensions reach a local daemon over native messaging (stdio framing, Chrome's 1 MB limit) as another C2D transport, generated for every C2D language. Host manifests and their per-OS install locations are fleet artifacts generated from the types, with an allowlist of extension IDs. First consumer: SeaBeam's Scotty browser extension. Needs a design issue, approval and a line in Next.
- **One agent across tenants, the platform side** (Jody, 2026-11-03, after tacplus.dev is up): an agent principal bound to a user's XID that holds one session per tenant; implicit deliveries in messaging to the sender's and human recipients' agents, tagged with the tenant; mailbox reads gated by the user's tenant session; an XID-owned personal scratch pad in the people tenant's Stratum and per-tenant pads in each tenant's Stratum. Generic tacplus pieces under the producer/consumer split; Scotty's behaviour stays SeaBeam's. Needs a design issue and approval.
- **Automatic security remediation with canary rollout** (Jody, 2026-11-01, "eventually"): any security vulnerability whose fix doesn't disrupt anything else is fixed, released and deployed automatically, first as a **canary** and then to 100% once the canary is healthy. Builds on: the nightly and deterministic scans (looper#49), the bug → fix → patch flow, the fleet canary design (`decisions/fleets-and-canary-huddle-2026-10-22.md`), and the post-deploy suite (fleet-seatrials#17). "Without disrupting anything else" needs a written test (contract unchanged, gates green, login continuity), designed before it's built. Chaos monkey runs in production too (Jody, 2026-11-01), so **a container being killed is not a disruption**. Failures matched to a chaos fault that recovered are logged and ignored; unrecovered ones, and ones with no matching fault, are triaged as real.
- Re-enable amd64 (x86_64) image builds for tacplus.dev once the change rate settles; arm64 only until then (decisions/deployments-per-cluster-2026-10-26.md, 2026-10-29 addendum)

- Convergent replicated state as proto annotations: tacplus generates the store, merge and gossip; first user the fleet-wide people store; eventually replaces Hazelcast in kernel; transparent behind generated interfaces (tacplus-dev/roadmap:items/people-fleet-wide-store.md; design, no hurry)
- **Multi-cluster** and region failover: three regions, active/passive with a witness, the tenant placement
  resolver (`items/multi-cluster.md`; "Kept for later" in `decisions/stratum-per-tenant-2026-10-25.md`; "Tabled"
  in `decisions/deployments-per-cluster-2026-10-26.md`).
- **Stratum transactions** across maps, server and generated client (`items/stratum-transactions.md`).
- **Data export before tenant deletion** ("download your data", perhaps required first;
  `decisions/tenant-deletion-2026-10-27.md`).
- The identity graph's remaining steps (`items/identity-graph.md`), the looper hierarchy's project looper
  (`items/looper-hierarchy.md`), doppelganger as a fleet service (`items/doppelganger-fleet-service.md`).
- Explorations, not plans: protos living with their owners (`items/explore-proto-ownership.md`), the application
  sensor (`items/tacplus-application-sensor.md`).
- Ongoing: the tacplus security sweep (`items/tacplus-security-sweep.md`) and the per-repo advisory issues.

## Not doing
- **We are not an identity provider, and will never become one** (Jody, 2026-10-29). No passwords, password resets, MFA enrolment, magic links, email or SMS codes, or user directories of our own. People sign in only through external OIDC IdPs (Apple, Google, Microsoft, or a company's own). Anyone who wants a self-hosted IdP can use one of the free ones (Keycloak, Authentik, Zitadel, …) and connect it as their OIDC IdP. If asked "will you implement passwords/…", the answer is no.

- **More SeaBeam services**: nothing else comes over that is not already in tacplus-dev, except stratum. Every
  other SeaBeam source is retired, never ported (Jody, 2026-10-27; tacplus-dev/looper#5, `ports.yaml`).
- **SeaBeam's test harness**: testing uses fleet-dev (tacplus-dev/looper#5; tacplus-dev/fleet-dev#3).
- **The `linked` claim**: retired; XIDs and request-named routing replace it (Jody, 2026-10-26;
  tacplus-dev/kernel#4, tacplus-dev/tacplus#171; `decisions/tenant-routing-2026-10-26.md`).
- **A grace period on tenant-namespace deletion**: deletion is immediate; keeping data is the caller's job
  (`decisions/deployments-per-cluster-2026-10-26.md` item 4; tacplus-dev/nsproxy#1).
- **Durable state in Stratum** (volumes, its own redo log, checkpoints) and **eventbus as a database**: durability
  is the redo service; eventbus is messaging only (`decisions/stratum-per-tenant-2026-10-25.md` items 3, 5).
- **Replacing Hazelcast** (it stays in Stratum) and **the reflector** (`decisions/fleets-and-canary-huddle-2026-10-22.md`).
- **Automatic region failover now**: resilience at the active site plus a backup at a second site only
  (`decisions/stratum-per-tenant-2026-10-25.md`, "Scope now").
