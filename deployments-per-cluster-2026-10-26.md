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

## Tabled: a tenant placement resolver (Jody, 2026-10-26)

Day 1 is a single cluster. Later the gateway looks up a request's target tenant in a **resolver** and gets the
cluster it is active on (its public name), which may be the gateway's own cluster (the gateway knows its own
cluster id, and dials in-cluster when local) or another (it forwards to that cluster's gateway with the original
token). For failover the resolver returns **both the primary and the secondary**. Notes from the discussion:

- It answers *where the tenant is active now*, with the lease **term**, and the gateway forwards only to the
  active cluster for that term. The gateway never decides failover itself (a blip between one gateway and the
  primary must not split traffic); the term changes only when the witness promotes the secondary (see "Kept for
  later: region failover" in `stratum-per-tenant-2026-10-25.md`), and the redo service fences writes on the same
  term. Returning both lets the gateway keep warm connections and switch at once.
- In the failover phase it must answer while a cluster is down, so it runs outside either cluster (beside the
  witness, at a third provider, or replicated). Day 1 it does not matter.
- Gateways cache placements and watch for changes; no lookup per request.
- A deployment spanning clusters shares one trust domain, and kernel's signing keys fail over with it, so a
  forwarded token verifies on the other side.
- Moving a tenant between clusters is the same operation: change its placement, bump the term.
- Not built Day 1. The gateway keeps all routing behind its single resolve-a-target seam so the resolver slots in
  as a new source of answers.
- Open: does placement belong to the tenant record in kernel, or is it the resolver's own data?
