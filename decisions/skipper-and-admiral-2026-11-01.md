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

- Whether the Admiral may call a Skipper (`/system` → tenant: "technically possible, probably unused").
- What each one does: its responsibilities and API.
