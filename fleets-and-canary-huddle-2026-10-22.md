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

## The reflector is dropped (Jody, 2026-10-22, later the same day)

- Jody opened with the reflector as "just really a reflector": a kernel instance on a cluster proves
  its identity (cloud-auth / k8s for now), the reflector decides whether it may join the domain, and
  if so the instance generates its own private key and installs the public key. Then asked the prior
  question: do we need it at all? Clusters learn about each other and permit another cluster to join
  or not.
- Finding: the kernel already became the reflector on 2026-10-14. Of the reflector's three jobs,
  membership (the kernel spans clusters; the pipeline decides members) and key distribution (kernel
  state replicates across clusters over mTLS; a public key is one more record) are already the
  kernel's, and public key publication for outside relying parties is already the gateway's
  discovery/JWKS, which we kept. What remains is ADMISSION: a new kernel instance proves its cloud
  identity (GKE directly, zero config) and the existing members decide by policy; then it generates
  its own key and the public half replicates. No new component, no new trust root, no second store.
- Rules the reflector was quietly providing, now needed explicitly:
  1. Bootstrap: a new cluster finds the existing members through pipeline config or a fleet DNS name;
     the first member of a new domain is designated by the pipeline, not elected.
  2. Who says yes: admission is a decision of the existing members as a group (quorum), never one
     reachable instance, or one compromised member can admit anyone.
- A customer-run cluster never joins our domain (Jody): "I dont think so and if we ever decide to do
  that then we could make changes." His reason: "isolated instances will disrupt the value of network
  effect." If it ever comes, it is a read-only key export to an outsider, not a component.
- Status: the reflector design in the identity doc is retired, kept as history. Admission becomes a
  kernel operation to design (cloud-auth proof, quorum policy, key install, bootstrap).

## Quorum across clusters, and why more than one cluster (Jody, 2026-10-22)

- Why several clusters at all: **geographic diversity**. We run the solution ourselves, in the cloud,
  on one provider (GKE is fine); bandwidth between its regions is massive and latency small, so the
  WAN/LAN distinction is minor. Data at rest can be duplicated to another cloud altogether.
- Kernel state is convergent, so cross-cluster is not a worry there. Stratum is atomic, so the
  quorum question is Stratum's.
- **No third node type.** Stratum's writer/search split already separates "counted for durability
  and split-brain" from "scaled for reads" (`platform/services/stratum/docs/DURABILITY.md`). Across
  clusters it is the same model: one writer per location forms the quorum; search members are per
  cluster and dynamic so reads stay local. A witness (votes, holds nothing) would be a writer role
  with no data, not a new member kind, and only if we ever had two data locations.
- **Shape:** three GKE clusters in three US regions, each with its kernel instance, gateway and one
  Stratum writer; `min_durable_copies = 2`, so a write acknowledges after the nearest remote writer
  and losing any one region keeps a majority. Cross-cloud is the archive sink (GCS shipping already
  built on the stratum-archive branch) pointed at a second cloud's bucket: asynchronous by design,
  data at rest, never quorum.
- Numbers (Google backbone, for the record): zones in one region < 2 ms RTT; us-central1 to
  us-east1/us-west1 about 30-35 ms; us-east1 to us-west1 about 60-70 ms. Only the atomic ack pays
  it; firehose record types ride the archive stream, not a transaction.
- **To verify before the design assumes it:** one stretched Hazelcast cluster across three regions
  with heartbeat and partition-migration settings tuned for ~35 ms, with a region broken on purpose
  (a looper task). Hazelcast discourages stretched clusters and its WAN replication is asynchronous
  (not atomic). If the test fails: three zones in one region as the fallback (same quorum, no
  regional survival), and the first concrete reason for the Hazelcast replacement on the long-term
  list.
- **Transactions and the fenced lock (Jody):** use transactions wherever possible so the caller makes
  ONE request that takes the fenced lock, applies its changes and commits. When that feature is
  built, build it on BOTH the server side and the generated client side, so the client-side
  complexity (lock, fence token, retry, commit) is hidden from the user. Extends the post-launch
  "Transactions across maps" item in `sonar/backend/decisions/mvp-plan-2026-09-28.md`; the API is
  generated (tacplus owns it), Stratum owns the implementation.
