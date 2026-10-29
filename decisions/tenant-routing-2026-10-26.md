# Tenant routing: the request names its tenant — DECIDED 2026-10-26

Huddle Jody + the kernel identity session, 2026-10-26. Follows `deployments-per-cluster-2026-10-26.md`
(tenant ids, namespaces, the tabled placement resolver).

## The problem (Jody)

In SeaBeam each tenant has a crew; a user of a tenant can speak to their own crew or another tenant's crew.
There are exceptions — Scotty is always your dog, only members of a tenant can talk to the Skipper, only the
Skipper can talk to the Admiral — but those are **authorization policies, not reachability**. So something in
the request must say which tenant it is directed to, and it cannot and should not be the token. Whether a request
originates outside the cluster or inside it must not matter. Nothing is ever an exception.

## Decided

1. **Three separate facts.** The **request** says where it goes (the target tenant). The **token** says who is
   asking (the person or service, and the tenant they act in). **Policy at the target** decides whether that is
   allowed. Cross-tenant traffic is not a routing exception.
2. **A route field on the request message**, declared in the proto: every RPC of a tenant-owned service names the
   request field carrying the **target tenant id** (the 8-character tid). That field decides which tenant's
   instance receives the call and nothing else. It is **required**: a call without it is refused, never defaulted
   from the token. A field, not a header: typed, in the contract, and unchanged by the REST translation.
3. **Platform services are never addressed by tenant**; their RPCs declare no route field.
4. **One rule inside and outside the cluster**: the gateway (external calls) and the tacplus-generated service
   clients (internal calls) route by the same declared field, from code tacplus generates once.
5. **Authorization is policy**: the existing level-or-below check on `target_namespace_field` becomes one policy
   among others rather than something routing implies; the generated policy must be able to say "any
   authenticated user of any tenant may talk to this crew" as well as "members only". SeaBeam's own rules (the
   Skipper, the Admiral, Scotty) are SeaBeam's policy, not tacplus's.

Scotty: a person's own dog lives in their personal space, so a request to Scotty targets the `people` tenant
(`00000001`) and policy admits only that person.

## Issues

- tacplus-dev/tacplus#170 — the route-field annotation and generated routing (gateway route tables and service
  clients), with authorization separated from routing.
- tacplus-dev/gateway#3 — route tenant-owned calls by the declared field and the instance registered for that tid.

## Related, SeaBeam

Tenants talking to each other — the Admiral discovers that two tenants have a relationship and tells each
Skipper, who asks whether they want to connect: `seabeam/roadmap/items/tenant-relationships.md`.

## Addendum 2026-10-28: instance identity by tid

Jody, deciding tacplus-dev/gateway#3: a tenant-owned service instance is identified by its tenant's **tid**,
not its name. Its SPIFFE ID is `spiffe://<td>/tenants/<tid>/services/<svc>` (kernel#23). The gateway matches the
request's target-tenant tid against the instance identity directly, with no tid → name lookup. User and session
token namespaces (`/tenants/<name>`) are not changed by this.

## Addendum 2026-10-29: the server refuses too (Jody)
"The black box hazelcast traffic must not allow accidental tenant to tenant connections. The mTLS certs and policy must not allow
this." Routing the client to the right tenant is not enough on its own:
- A tenant-owned instance refuses any call meant for another tenant, whoever the caller is. A `/system` caller included. The check
  compares the tid in the instance's certificate (kernel#23) with the call's tenant.
- Traffic between a tenant-owned service's members (Stratum's Hazelcast, for example) admits only members holding that tenant's
  certificate, with network policy as a second layer. Shared cluster names and multicast join are not allowed
  (tacplus-dev/stratum#17; the record plane is tacplus-dev/tacplus#207).
