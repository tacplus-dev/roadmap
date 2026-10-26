# Several deployments per cluster: namespaces, tenant ids, nsproxy — DECIDED 2026-10-26

Huddle Jody + the kernel identity session, 2026-10-26, after the Stratum huddle
(`stratum-per-tenant-2026-10-25.md`). Work is filed as approved issues on the tacplus-dev repositories,
worked by their loopers (`seabeam/roadmap/items/looper-hierarchy.md`).

## Decided

1. **More than one deployment on one Kubernetes cluster** (more than one kernel, gateway, ...), separated by
   Kubernetes namespaces named from the deployment's name plus fixed values, as short as possible:
   - platform services: **`tac-<deployment>`**;
   - each tenant: **`tac-<deployment>-<tid>`**, holding that tenant's tenant-owned services, its Stratum and its
     pigeons.
   A deployment name is at most 16 characters, `[a-z0-9-]`, starting with a letter.
2. **Tenant ids (tid)**: exactly 8 characters of `[0-9a-z]`, immutable, never reused (a Kubernetes namespace
   cannot be renamed, so a tenant's name is not usable there). **All-digit ids are reserved for the platform**,
   recognizable at a glance: **`people` is `00000001`**, fixed in every deployment; `00000002` and up stay free.
   Minted ids are random and start with a letter, so they never fall in the reserved range. Using the tid in
   SPIFFE paths and grants instead of the tenant name is a separate, later decision.
3. **Trust domain per deployment**, derived from its name (for example `prod.tacplus.dev`).
4. **nsproxy**: a deployment runs in `tac-<deployment>` with no rights outside it, so it cannot create its
   tenants' namespaces. nsproxy, one per cluster, deployed by the top pipeline, does it on the deployment's
   behalf:
   - **an RPC**, two idempotent calls: ensure a tenant namespace exists, delete it;
   - **tenant namespaces only**; the top pipeline creates `tac-<deployment>`; only tenants are dynamic;
   - **deletion is immediate**; keeping the data is the caller's responsibility;
   - the caller is identified by Kubernetes (its service-account token, verified by TokenReview), never by a
     deployment's kernel; it acts only for its own deployment, only on names of the pattern, only on namespaces
     nsproxy created; it sets baseline isolation (default-deny NetworkPolicies, reach to its own
     `tac-<deployment>` only) and a ResourceQuota itself, so tenant isolation does not rest on the deployment.
5. **Seatrials moves** from `schematic` to `tac-seatrials` and `tac-seatrials-<tid>`, cleanly: the Let's Encrypt
   certificate and account key and kernel's state are carried over (no new certificate order), and the
   trust-domain change (`local.dev` to `seatrials.tacplus.dev`) signs everyone out while XIDs, links and
   memberships survive. A runbook, executed by the hub.

## Issues

- tacplus-dev/kernel#3 — tenant ids (and the reserved range, `people` = `00000001`).
- tacplus-dev/nsproxy#1 — the service.
- tacplus-dev/operator#3 — deployment names, both namespace kinds, full service names, NetworkPolicies, the
  trust domain, cluster-scoped names per deployment, tenant namespaces through nsproxy.
- tacplus-dev/fleet-seatrials#4 — the seatrials config and the migration runbook.

## Open

- The top pipeline itself (creates `tac-<deployment>`, deploys nsproxy): part of the deploy huddle.
