# A product layer: `/products/<name>`, between `/system` and the tenants (Jody, 2026-11-08)

Jody, in a huddle about where a product's own services run: *"my gut instinct is that system should be the base platform
that comes from tacplus.dev and we should not run stuff in there."* And: *"I think we need a home for [the] product. tacplus
owns /system but we need a place for stuff that the … product will provide above system."* On the shape below and on the
custodian rule for tenant data: *"I think your lean is right."*

## Decided

- **`/system` is tacplus.dev's platform and nothing else.** A fleet deploys no product code there.
- **A product's shared services live at `/products/<name>`.** That's a trust-domain path, e.g.
  `spiffe://<td>/products/<name>/services/<svc>`, the same shape as `/tenants/<tid>/services/<svc>`. A fleet may hold more
  than one product, and every token and audit line names whose code it is. In a cluster it runs in its own namespace
  (`tac-<deployment>-<name>`, following the 2026-11-07 terms); as with `/system`, the rules are about principals, not
  namespaces.
- **The fleet owns the product layer, and tacplus.dev owns `/system`.** A tacplus release never ships product code, and a
  product can't change the platform.
- **S2S edges, all fixed at build time:**

  | Edge | |
  |---|---|
  | tenant → `/products/<name>` | allowed, like tenant → `/system` |
  | `/products/<name>` → `/system` | allowed |
  | `/products/<name>` → tenant | **no** |
  | `/products/<a>` → `/products/<b>` | **no** |
  | `/system` → `/products/<name>` | **no**: the platform never depends on a product |

  User traffic (C2S, including a service acting on a user's behalf) is unchanged (s2s-never-crosses-tenants, 2026-11-07
  addenda).
- **Tenant data: custodian only, like redo.** A product service may keep data that the calling tenant sends it, keyed by
  the caller's tid **taken from its token**, never from the request. A tenant reaches only its own data. The data is
  deleted with the tenant (tenant-deletion-2026-10-27). The product layer never calls into a tenant to fetch anything. This
  is the 2026-11-01 rule ("tenant data never lives in `/system`") with redo's custodian exception made the product layer's
  rule.

## Open (for the design)

- **Reads across tenants.** Who, if anyone, may read across tenants' custodian data, for example a product owner's
  billing view over per-tenant usage. Not decided; the design proposes it and Jody decides.
- **Mechanics:**
  - how a service declares the layer (`(spec.service)`, `tacplus.yaml`, `fleet.yaml`);
  - kernel audiences and the `/products/<name>` SPIFFE paths;
  - door enforcement in all three generated languages (refusing product → tenant and product → product);
  - the operator's namespace;
  - the registry `visibility` of a private product's services.

Related: `s2s-never-crosses-tenants-2026-10-31.md`, `tenant-routing-2026-10-26.md`, `tenant-deletion-2026-10-27.md`,
`stratum-per-tenant-2026-10-25.md`.
