# Roadmap: tacplus-dev (platform)

Owned by Jody; changed only by an agent at Jody's request. Loopers propose in PROPOSALS.md.

This file covers what spans repositories or has no repository yet. Each repository's own direction is in its
`tacplus-dev/<repo>:ROADMAP.md`. tacplus-dev must work by itself, without SeaBeam.

## Now

- **tacplus 2.4.x is the current line** (v2.4.0 and v2.4.1, 2026-10-27): the global registry and the declared
  compiler floor. The services are moving to the **2.4 floor** ("Declare the tacplus floor: 2.4" on each repo;
  kernel is done), then to versioned releases.
- **fleet-seatrials is live** (GKE `seatrials`, https://seatrials.tacplus.dev): kernel, gateway, oauth, opa,
  eventbus, stratum, letsencrypt, kmaker, llmproxy, doppelganger, tenantecho, coop, pigeonhost. Sign-in works with
  Google (routed to `alpha`) and Microsoft (routed to `beta`), and Apple.
- **The loopers** run on the tacplus-dev GitLab group (`gitlab.com/tacplus-dev/looper`): one issue-driven looper
  per repository, working `approved` issues (`tacplus-dev/looper:ROADMAP.md`).
- Planning files in every repository: `decisions/repo-roadmap-files-2026-10-27.md`.

## Next

In priority order.

1. **Stratum: per tenant, in memory, durable through the redo service** (highest priority;
   `decisions/stratum-per-tenant-2026-10-25.md`). The critical path, each step blocked by the one before:
   1. tacplus-dev/tacplus#163: the generated records write surface, entry/map locks, durability levels, Watch.
      **Needs Jody's decision on D1–D6** (on the issue).
   2. tacplus-dev/tacplus#170: route tenant-owned RPCs by a declared target-tenant field
      (`decisions/tenant-routing-2026-10-26.md`).
   3. tacplus-dev/redo#1: the redo service, in the reserved `restore` tenant (`00000002`).
   4. tacplus-dev/stratum#2: per-tenant, in-memory Stratum, no event bus.

   Then on the same path: tacplus-dev/operator#2 (render per-tenant Stratum and redo), tacplus-dev/people#1 and
   tacplus-dev/kernel#2 (the identity graph moves to the people service), tacplus-dev/letsencrypt#2 (certificates
   written to Stratum directly), tacplus-dev/fleet-seatrials#3 (seatrials adopts all of it).
   Roadmaps: `tacplus-dev/tacplus:ROADMAP.md`, `tacplus-dev/redo:ROADMAP.md`, `tacplus-dev/stratum:ROADMAP.md`,
   `tacplus-dev/people:ROADMAP.md`.
2. **Service releases**: every service tags `vMAJOR.MINOR.PATCH` (from `v0.1.0`, `0.x` until 2026-12-01) and
   publishes its image ("Release: ..." on each repo, e.g. tacplus-dev/kernel#7). All wait on the org secret
   tacplus-dev/looper#12 (Jody). The gateway's release also needs a decision: tacplus-dev/gateway#6 (its image is
   shaped by the fleet).
3. **Fleets pin by release line** (MAJOR.MINOR → latest patch, image by digest): tacplus-dev/tacplus#180, then
   tacplus-dev/fleet-seatrials#8 and tacplus-dev/fleet-dev#2. Both fleets move to 2.4 first
   (tacplus-dev/fleet-seatrials#1, tacplus-dev/fleet-dev#1).
4. **fleet-dev as the test fleet**: every service, an ephemeral cluster per run, a test IdP, end-to-end tests on
   every release: tacplus-dev/fleet-dev#3 (`tacplus-dev/fleet-dev:ROADMAP.md`).
5. **Tenant deletion is complete** (`decisions/tenant-deletion-2026-10-27.md`): tacplus-dev/kernel#19 (no new
   AT/ST, memberships removed, XIDs kept), tacplus-dev/oauth#11 (providers leave nothing behind),
   tacplus-dev/kernel#20 (age out XIDs idle for about a year).
6. **Several deployments per cluster** (`decisions/deployments-per-cluster-2026-10-26.md`):
   tacplus-dev/operator#3 (`tac-<deployment>` and `tac-<deployment>-<tid>` namespaces, tenant namespaces through
   nsproxy; nsproxy itself is done), then tacplus-dev/fleet-seatrials#4 (seatrials moves, with a runbook).
   `tacplus-dev/operator:ROADMAP.md`, `tacplus-dev/nsproxy:ROADMAP.md`.

## Later

- Convergent replicated state as proto annotations: tacplus generates the store, merge and gossip; first user the fleet-wide people store; eventually replaces Hazelcast in kernel; transparent behind generated interfaces (tacplus-dev/roadmap:items/people-fleet-wide-store.md; design, no hurry)
- **Multi-cluster** and region failover: three regions, active/passive with a witness, the tenant placement
  resolver (`items/multi-cluster.md`; "Kept for later" in `decisions/stratum-per-tenant-2026-10-25.md`; "Tabled"
  in `decisions/deployments-per-cluster-2026-10-26.md`).
- **Stratum transactions** across maps, server and generated client (`items/stratum-transactions.md`).
- **Data export before tenant deletion** ("download your data", perhaps required first;
  `decisions/tenant-deletion-2026-10-27.md`).
- **The deploy pipeline**: the top pipeline (creates `tac-<deployment>`, deploys nsproxy), with Jody's sign-off on
  each deploy, then a panel. No issue yet (`decisions/deployments-per-cluster-2026-10-26.md`, "Open").
- The identity graph's remaining steps (`items/identity-graph.md`), the looper hierarchy's project looper
  (`items/looper-hierarchy.md`), doppelganger as a fleet service (`items/doppelganger-fleet-service.md`).
- Explorations, not plans: protos living with their owners (`items/explore-proto-ownership.md`), the application
  sensor (`items/tacplus-application-sensor.md`).
- Ongoing: the tacplus security sweep (`items/tacplus-security-sweep.md`) and the per-repo advisory issues.

## Not doing

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
