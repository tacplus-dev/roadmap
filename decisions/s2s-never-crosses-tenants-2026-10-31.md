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
