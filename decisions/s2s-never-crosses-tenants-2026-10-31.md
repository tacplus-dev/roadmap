# S2S never crosses tenants (Jody, 2026-10-31)

Jody, 2026-10-31: "S2S is defined at build time where C2S is defined at runtime (with a build time entitlement system).
So we have no way for S2S across different tenants and we don't want it. We do allow tenants to dial into the system and
technically we can have system to tenant but we probably won't use that."

## Decided

- **C2S** is a user, or something acting on a user's behalf (a delegated token, a virtual user), calling a service. It is
  decided **at run time**, within the entitlements fixed at build time. Cross-tenant traffic, where it exists, is C2S, and
  the target's policy authorizes it (`tenant-routing-2026-10-26.md`).
- **S2S** is one service calling another, and it is fixed **at build time**: the fleet's protos declare every edge
  (`(spec.grpc_s2s_client)`). The allowed edges are:
  - **tenant → `/system`**: a tenant's instances call platform services. Allowed.
  - **`/system` → `/system`.** Allowed.
  - **`/system` → tenant**: technically possible (the record plane, fleet-wide readers), but not expected to be used. It
    is not a pattern to design for.
  - **tenant → another tenant: does not exist, and isn't wanted.** No build-time edge, no runtime override.
- **Member (black-box) traffic** stays inside the service's own pods, behind the member port, on loopback (#212). It
  never crosses a tenant.
- **Reachability never implies authorization.** Isolation rests on identity checked at every listener, not on network
  layout. The gateway is a proxy, not an originator; the originator's identity is what counts.

## What enforces it (tacplus)

- A tenant-owned instance admits S2S callers only from its own tenant or `/system`. Another tenant's **service** token is
  refused at the door, always. `RPC_FLAG_ANY_TENANT` opens an RPC to other tenants' **users** (C2S) only, never to their
  services.
- A tenant-owned caller's generated S2S client stamps the route field with its **own** tid, and refuses a call that names
  another tenant before sending, the same rule MCP already uses. A service can't even ask to cross.
- Black-box protocols bind to loopback only and cross pods only through the member port.

Related: `tenant-routing-2026-10-26.md`, `black-box-traffic-2026-10-29.md`, tacplus-dev/tacplus#212.

## Addendum 2026-11-01: tenant data never lives in /system (Jody)

Prompted by the blobstore, which had been built as one shared `/system` instance writing into every tenant's Stratum. Jody:
"Yes it has to be per tenant. Otherwise it creates dependencies to system that should not exist."

- A service that **holds or serves a tenant's data** is `tenant_owned`: one instance per tenant, with its own storage. `/system`
  never stores a tenant's data, and never needs to call into a tenant to do its job.
- So `/system` → tenant, "technically possible", is **not** a way to centralize tenant data. A design that relies on it must say why
  it can't be tenant-owned, and gets Jody's decision.
- First application: tacplus-dev/blobstore becomes tenant-owned.

## Addendum 2026-11-07: restated, and redo (Jody; tacplus-dev/tacplus#258)
Jody: "Tenants must be allowed to call into system based on build time policy. Tenant services never talk to other tenant services. This is not the same as user. User traffic can originate from external and is permitted to any tenant and AAA is a runtime policy based on entitlements, etc. Now because a service can act on behalf of a user the same traffic can go between tenants."

- `RPC_FLAG_ANY_TENANT` opens an RPC to other tenants' **users** (C2S), including a service acting on a user's behalf (a delegated token whose subject is the user, checked at run time against the user's entitlements). It never opens one to another tenant's **service** principal. #258 is fixed as written, in every generated language. No reserved-tenant exception (option A) and no new flag (option B).
- A platform tenant (all-digit tid: `people`, `restore`) is a tenant, not `/system`.
- **redo:** it held every tenant's write log as one instance in the `restore` tenant, called by each tenant's stratum. That's tenant → another tenant, and it contradicts the 2026-11-01 addendum, since a service that holds a tenant's data is tenant-owned. redo becomes **tenant-owned**, one instance per tenant beside that tenant's stratum (as blobstore did), and stratum → redo is a same-tenant S2S edge. Its second-site copy is cross-deployment, which is parked (tacplus#204).

## Addendum 2026-11-07 (2): where user traffic flows (Jody)
Jody: "If the traffic is from outside it must go thru the gateway. but tenant to tenant user traffic does not have this requirement. because they have the same kernel in system they can directly access each other thru the internal connectivity."

- **From outside:** always through the gateway, the only public entry (network-exposure-2026-11-03.md).
- **Tenant to tenant, as a user (C2S, including on a user's behalf):** direct, over the cluster's internal network, with no gateway hop. Both tenants trust the same kernel in `/system`, so each verifies the other's user tokens. Isolation rests on identity at every listener, not on network layout ("reachability never implies authorization").
- So within a deployment, a tenant namespace may reach another tenant namespace at the network layer. Each door then admits only users, and refuses another tenant's service principal (this decision). Across deployments, nothing is reachable.
