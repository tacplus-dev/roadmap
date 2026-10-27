# Tenant deletion: everything the tenant owns goes, people stay (DECIDED 2026-10-27)

Jody, 2026-10-27, after oauth#2 (tenant sign-in providers live in the tenant's config slice).

## The rule

Deleting a tenant is **destructive and complete**: all of the tenant's data is deleted. Nothing it owned may be
left orphaned anywhere. A data export ("download your data"), and perhaps requiring one before deletion, may come
later (an item, not part of this decision).

## What happens, per piece

| Piece | On tenant deletion |
|---|---|
| Tenant record, `tenant_config` (incl. its oauth providers), project configs | Deleted by kernel `DeleteTenant` (already so). Services that read them (e.g. oauth) keep no copy and drop them on the next watch event. |
| The tid | Tombstoned, never reused (kernel#3). |
| **Tokens** | The tenant can no longer **mint** access tokens (AT) or service tokens (ST). Tokens already issued are not chased: ATs and STs are short-lived, and tenant-to-tenant calls accept only STs that are close to expiry. |
| **People (XIDs)** | **Stay.** A person exists outside every tenant. Their membership of the deleted tenant is removed. |
| Stratum data and its redo log (in the `restore` tenant, both sites) | Deleted. |
| Kubernetes namespace `tac-<deployment>-<tid>` | Deleted. The flow is kernel → operator (sees the tenant gone, deletes its `TenantNamespace`) → nsproxy (deletes the namespace). The deployment pipeline manages the deployment, not tenants; nsproxy manages tenant namespaces only. |

## Related: XID ageing

XIDs are not deleted with a tenant. An XID that has **never signed in again for about a year** is aged out (deleted
like DeleteAccount: links revoked, person tombstoned, XID never reused).
