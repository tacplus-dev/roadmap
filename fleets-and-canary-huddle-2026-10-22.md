# Fleets, identity and canary huddle (2026-10-22, Jody + Snoopy)

Status: HUDDLE. Nothing decided. Follows `identity-graph-huddle-2026-10-21.md` and the identity
design doc ("Federated Workload Identity: Satellites and Reflectors", Claude Docs).

## How the user identity layout fits the reflectors

- Reflectors (2026-10-14) are key publication between INDEPENDENT fleets: a fleet's kernel installs
  its public keys under an issuer name at oidc.seabeam.net; other fleets fetch that JWKS; policy on the
  reflectors says who may publish under which name. Deferred until a second fleet exists.
- Yesterday's layout (`/users/<xid>`, graph owned by the kernel, durable in Stratum, session token +
  per-tenant tokens) is entirely within one fleet. A fleet is a trust domain:
  `spiffe://<fleet>/users/<xid>`.
- Across fleets it works unchanged: fleet B verifies a token from fleet A against A's keys from the
  reflectors and sees `spiffe://A/users/<xid>`, a foreign identity, authorized by B's policy, default
  deny. Reflectors never see XIDs or mappings, only keys.

## One identity fleet (Jody)

- There is ONE identity fleet: XIDs and TIDs are minted in exactly one place. "We are a single
  product." Identity is a role of the one production fleet, not a separate fleet to run.
- Other fleets (if any) are relying parties through the reflectors; their kernels mint service and
  workload identities, never users.

## No staging, no dev fleet: canary and stair-stepping (Jody)

- Jody: "I dont really know if I want dev or staging at all anymore. I feel like our model should be
  canary release and stair stepping."
- Staging goes: a staging fleet is a fake production. Canary in prod is a truthful test with real
  traffic, and the fleet already spans clusters, which gives the stair: one pod, one cluster, all
  clusters. Stair-stepping by cluster is what the cross-cluster kernel was built for.
- Dev stays a BUILD, not a fleet: the gates image, the loopers, local runs against a single node. CI,
  not an environment. Nothing in it needs real identity.
- Consequence for identity: one fleet, one identity, reflectors stay deferred until a customer runs
  their own fleet (the case they were designed for). The "does dev trust prod's users" question
  disappears.

## What canary demands that staging was hiding

1. Two versions of a service side by side, with gateway and S2S routing able to split by version.
   The kernel's endpoint pool has to know a version, not just an address.
2. Compatible state across versions: kernel state is convergent, so a newer writer and an older
   reader must tolerate each other; Stratum schema changes go expand then contract, never in one
   step.
3. Breaking generated changes cannot be canaried. House rule: a breaking generated change is a
   rebuild, not a rollout; that is what a major is for. Everything on a minor or patch must be
   canary-safe by policy (`tacplus/docs/VERSIONING.md`).
4. A signal that decides the step: the data gate's signals with the Admiral deciding "fine" or
   "roll back" (already on the post-launch list in `sonar/backend/decisions/mvp-plan-2026-09-28.md`,
   "Alerting moves into the Admiral"); same work, sharper reason.
- Risk accepted: a bad canary touches real users, bounded by the first step (one pod, one cluster)
  and automatic rollback on the signal.

## Open

- Version-aware routing in gateway and kernel endpoint pools (design needed).
- The canary step policy: sizes, dwell times, what the Admiral measures, who can override.
- Expand/contract discipline for Stratum record types (ties to the transactions item).
