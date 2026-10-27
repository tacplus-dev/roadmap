# tacplus global registry and version floor

- **Status:** active (design agreed 2026-10-23, built on branch `registry-and-floor` of
  github.com/tacplus-dev/tacplus, not merged)
- **Owner:** Jody (design), Snoopy (build)
- **Opened:** 2026-10-23
- **Gets more expensive while held:** every new service allocated per repo in 1-99 is another
  potential collision to untangle when kernels are composed; doppelganger's lock points at the
  branch's behaviour and cannot be consumed by a released tacplus until this merges.
- **Records:** github.com/tacplus-dev/registry (the global registry, private, seeded 2026-10-23);
  tacplus `docs/USER.md` "Service Registry"; `pkg/registry/global.go`; `pkg/tacplusyaml/floor.go`;
  [doppelganger-fleet-service](doppelganger-fleet-service.md)

## What (Jody, 2026-10-23)

- **A registry repo any tacplus consumer can register with.** It controls the ids that must be
  globally unique and maps a service to its repo. Kernel, Stratum and the rest of schematic are
  listed in it; private repos are listed too.
- **A tacplus consumer declares the minimum MAJOR.MINOR it supports**, so new tacplus
  functionality can be required by the consumer that uses it.
- Context: doppelganger is the first service that is not in schematic's type tree but must
  deploy to a fleet. "Everything in the fleet is in our types" has to become "a deployment
  declares which registered services it loads and the kernel is built from that set".
  Kernel will be open-sourced (Stratum stays private; Hazelcast comes out of kernel, stays in
  Stratum).

## Design (built)

- Service and vservice ids are `TokenEntitlements` field numbers and `ServiceName` enum values,
  so they share one space across every proto repo composed into a kernel.
- **Global registry** = one `registry.yaml` in tacplus-dev/registry, today's schema plus `repo`
  and `visibility` per entry. Append-only; never renumbered. `tacplus registry register NAME
  --kind service|vservice --repo URL --visibility public|private --proto-dir <checkout>`
  allocates the next id; merge is the act of registration.
- **Ranges:** 1-99 legacy, frozen as allocated (schematic 5-58, sonar 54/70-79, docmanager
  80-81); 100 reserved (`azp`); new registrations 101 upward; 2047 ceiling (two-byte field
  numbers). First global registration: doppelganger = 101.
- **Locks:** a proto repo's `registry.yaml` names its authority with `global:`. `tacplus
  registry` syncs it; an unregistered declared service is an error naming the registration
  step; a lock never allocates (`ServiceID` refuses) and never renumbers. Builds read only the
  lock, so generation is reproducible offline. A `registry.yaml` without `global:` is unchanged.
- **Floor:** `tacplus: "MAJOR.MINOR"` in `tacplus.yaml`, enforced at config load: MAJOR equal,
  running MINOR >= floor, patch component rejected (pin rule), unstamped `0.0.0` build fails
  closed, absent = no floor.

## Agreed order of the tacplus work (Jody, 2026-10-23: "Agree")

Two generation modes with a hard line: **service generation** (in the service's repo: own proto + sys
contract + protos of declared dependencies; checks a service can answer about itself) and the **fleet
build** (from a deployment manifest: registry + services with versions, transitive dependency
resolution, whole-fleet checks, output = kernel/gateway/operator). "You build for the fleet." Tacplus
work comes first; common things go into tacplus:

1. Kernel declared as the host of state labels in the sys contract, so a service generates alone.
2. Service dependencies declared in `tacplus.yaml` by registry name, with the generated client for each.
3. The fleet manifest and the fleet build (whole-fleet checks move here).
4. The registry and floor (this branch) merged first, since the rest reads the registry.

Also decided: budget lanes do not belong in llmproxy; they become tacplus-generated code running in the
owning service (github.com/tacplus-dev/llmproxy issue #1). Model egress stays a service, not a primitive.

## Progress on item 1 (2026-10-23, branch `registry-and-floor`)

Standalone generation of a service (llmproxy as the test case, crew import replaced by a local enum in
scratch) hit these whole-fleet assumptions in order; each is now a sys-contract fact:
1. routing label "kernel" counts as served (DONE); 2. config state routes to the well-known host when no
store is loaded (DONE); 3. `ServiceHealth` moved into `sys/servicehealth.proto`, field numbers frozen; the
host imports it (DONE in tacplus; the public kernel.proto still declares its own copy — next);
4. The minting contract is `sys/minter.proto` (`minter.Minter`, vservice id 102, hosted at label "kernel",
the one authoritative `(spec.token_minter)`); the public kernel.proto implements it and keeps its policy RPCs
in `Mint` (DONE). 5. A sys proto declaring a state root gets its state service (DONE).
**Result: llmproxy generates AND compiles from its own proto + the sys contract alone** (scratch, crew
import replaced by a local enum). Item 1 is closed for service generation.
Not yet verified (needs the fleet build, item 3): kernel hosting `minter.Minter` and the sys `ServiceHealth`.
Deferred (Jody, 2026-10-23: "leave it for later"): 19 cross-test fixtures declare their own health root / minter and must become fleets whose host is
named kernel; the private schematic is untouched (pinned tacplus).

Kernel copied wholesale (with its nested-repo history) to github.com/tacplus-dev/kernel (private) on
2026-10-23, Jody: *"We will copy kernel over to its github home and only make changes to it. we will keep
these as ships in the night for now. For the new stuff we want to do it the right way."* Registry: kernel 8
-> that repo. The private schematic kernel is unchanged and runs the launch fleet.

## Item 2 done (2026-10-23, branch `registry-and-floor`)

`tacplus.yaml` `depends: [llmproxy, weather@1.2]`: each name resolves through the global registry the
project's lock names (`global:`) to its repo; the repo's proto tree is fetched at the version (MAJOR.MINOR
-> latest patch; none -> latest), staged without its dependencies.yaml, and merged into the generation.
Direct dependencies only; transitive following is the fleet build's (item 3). Verified: a consumer with
`depends: [llmproxy]` generates types for both and builds. The S2S client appears once the consumer's proto
declares `(spec.grpc_s2s_client) = {service: "llmproxy.LlmProxy"}`.

## Registry refinements agreed 2026-10-23 (evening, Jody)

- **The global registry is the id authority, not a publication list.** An entry there means the id is
  globally unique and the contract has one home; that home may be public, shared with a partner, or
  private. No reachability rule. `visibility` is a note.
- **Entries name a `contract`** (where the proto lives), not an implementation. **Which implementation a
  fleet runs is the manifest's choice**, always stated as the repo, no default: *"it should be possible to
  use your own implementation of anything ... you should use the same id"*. The fleet build checks an
  override's lock carries the registry's id and its proto declares the name.
- **Ranges:** 1-99 frozen legacy (wherever the service sits); 100 reserved; `scope: global` allocates
  101-1023; `scope: private` (a fleet owner's registry for services that never need a global id)
  allocates 1024-2047.
- **Registries a fleet loads are disjoint.** A lock / the fleet build names them in order under
  `registries:`; they are unioned and a name or id in two of them is an error, never a precedence call.
- **No versions in any registry.** The fleet manifest pins what a fleet runs (MAJOR.MINOR -> latest
  patch); a service's `depends` names the oldest MAJOR.MINOR it was built against.
- Fleet manifest shape agreed in outline: `registries: [...]`, `tacplus: "X.Y"`, `services: [{name@X.Y,
  repo}]`; the build resolves transitively, runs whole-fleet checks, emits kernel/gateway/operator and a
  commit-level lock. Where the manifest lives: a deployment repo per fleet (recommended; not yet decided).

## Item 3, first deliverable done (2026-10-23, branch `registry-and-floor`)

`tacplus fleet build` in a deployment repo (one per fleet, Jody 2026-10-23): fleet.yaml = registries
(ordered, disjoint, unioned) + tacplus floor + services each with the implementation repo (required, no
default). Resolves each at MAJOR.MINOR -> latest patch, checks lock id == registry id and the proto
declares the name, checks the dependency closure (every `depends` runs in the fleet at a satisfying
version), merges protos with the fleet's own dependencies.yaml, runs whole-fleet checks (routing + C2S
host labels, SMB authority, one minter, registry), writes fleet.lock (commits). First fleet:
github.com/tacplus-dev/fleet-dev (private): kernel, llmproxy, doppelganger — resolved clean once
doppelganger's main was fast-forwarded (the build caught the stale lock on main, id 1 vs 101).
Not yet: generating kernel/gateway/operator from build/proto as part of the build; residency checks
(they run at kernel generation).

## Ports and the fleet of eleven (2026-10-23, late)

Jody: *"the following will also need to be ported over to tacplus-dev ... blobstore gateway kmaker letsencrypt
eventbus oauth"* — DONE as private repos under github.com/tacplus-dev (history carried; each carries its own
proto under proto/; PROVENANCE.md records the commits; registry contracts repointed). Found on the way: those
service repos .gitignore'd `proto/` (the platform build cloned the shared proto there) — fixed in the copies.
The private platform's pushed mains are from 2026-08-29; everything since (86–97 commits per repo) is on
unpushed local branches, so contracts fetched from GitLab are August state. The ported copies were taken from
the local (October) state. Kernel's OPA-eval input contracts (vservices/entitlements.proto, proxy-policy.proto)
now live in the kernel repo; the fleet build follows option-level package references to pick them up.
Fleet build verified against 11 services (kernel, opa, gateway, stratum, eventbus, kmaker, oauth, letsencrypt,
blobstore, llmproxy, doppelganger; opa and stratum from the local platform checkout since they are not ported).
Kernel generation from that merged tree now fails only on the record query plane: blobstore/letsencrypt records
need a `(spec.trait)` hosting the record query plane, which today only Scotty's `vservices/query.proto` (host
stratum, id 71, contract = sonar) declares. Its right home is stratum's contract (the platform's record store).
Also asked: `tacplus.yaml` `kind: service|client|library` (field + validation DONE; checks and fleet use next)
and flattening the proto layout (`proto/<name>.proto` instead of `proto/services/`), not started.

## Milestone 2026-10-23 night: kernel generates from the dev fleet

opa and stratum ported too (Jody: yes); the record query plane trait (`vservices/query.proto`, id 71) now
lives in stratum's contract. `agentsensor` (id 42) became the first **library** repo (contracts only,
`kind: library`), pulled into fleets by reference. Fleet build additions: contract-only vservices referenced
by a listed contract resolve through the registry as libraries (recorded in fleet.lock); a contract includes
files its repo hosts for the service (traits with `host:` = the service). The dev fleet
(github.com/tacplus-dev/fleet-dev) now resolves 11 services + 1 library entirely from tacplus-dev repos, and
`tacplus build --src build/proto` in the kernel repo generates clean, with the sys `minter.Minter` server and
the sys `ServiceHealth` state service generated for kernel to host. Next: kernel's Java implements
minter.Minter (its four mint RPCs moved there), then gateway/operator generation from the same tree, then
images and the small cluster. The private platform's unpushed state (since 2026-08-29) is Jody's to push.

## Where the private services go (Jody, 2026-10-23 night)

- **Dead, boneyard only:** saml, sentinel, argus (Jody: *"saml, sentinel and argus are all dead. we will bring back
  sentinel somewhat as admiral"*), timedb, graphdb (Jody named both), llmsettings (replaced by llmproxy), vanilla
  (template; the public tacplus has its own example), **personalauth** (Jody, same night), and **agentsensor**
  (Jody: *"if we have any agent sensor stuff in anything at all that is a mistake. that is from a long long dead
  service and was put in the wrong place"*; its library repo was deleted, gateway's trace publish to its topic
  removed in the public copy; test fixtures in stratum/eventbus/tacplus still use the name as sample data).
  Their registry ids stay frozen, never reused; their entries keep no contract home.
- **Moved to tacplus-dev** (ships in the night; private copies run the launch fleet): kernel, opa, gateway, stratum,
  eventbus, kmaker, oauth, letsencrypt, blobstore, operator, messaging, mailbox; llmproxy; doppelganger.
  (personalauth and llmsettings had no git remote in GitLab at all.)
- **Stay private, Seabeam product:** everything under sonar (scotty, scottyagent, scottypersonal, crew-runtime, work,
  weather, profile, scotty-harness, docmanager, brand, webpage, clients), the scotty-ios client, harness, and the tools
  (sideband, buddy, looper, pipewatch). Platform clients cavalier/webapp/policy/sentinel/llmsettings: undecided.
- The platform repos' GitLab `main` branches have not moved since 2026-08-29; see
  `~/Library/CloudStorage/Dropbox/PLATFORM-GIT-STATE-2026-10-23.md` (copy in the schematic checkout) for the measured
  state and the questions for the agent in charge (stratum: 51 commits never pushed).

## 2026-10-23, late night: kernel implements the sys contracts

Kernel (github.com/tacplus-dev/kernel) compiles against the dev fleet's merged tree: IdentityServiceImpl
implements `minter.Minter`'s four RPCs (same methods, now on the sys handler), ClusterServer registers the
generated Minter wrapper beside the slimmer Mint, the sys health root's access policy is declared in
kernel's InfraStatePolicies (every service writes its own instances, reads all), and the health map is read
from the sys backend. Kernel's codegen refused to generate a state server without a policy for the sys root —
the fail-closed check did its job. Operator joined the dev fleet (12 services). In flight: kernel's test
suite, gateway generation + cargo check, operator generation + go build from the same tree.

## 2026-10-23, night: fleet-side services generate from the dev fleet; operator is a library

- kernel: implements minter.Minter, hosts the sys health root; compiles and 438/441 tests pass against the dev
  fleet tree (remaining 3 are tests pinned to launch-bundle sample services, being rebased).
- gateway: proxies minting to the sys minter, reads the sys health root; cargo check + 88 tests green. Tests
  for personalauth (dead) and Scotty records (product) removed from the public copy.
- **operator is a library, not a service** (Jody: *"Operator might should be a library and not a service within
  tacplus-dev because it does not know about upstream services that will exist."*): `kind: library`; each
  deployment repo builds its own operator against its merged tree. Its hand-written per-service defaults
  table is gone (the generated holder defaults recurse). Follow-ups: Go module split so a deployment repo can
  import it; fleet-specific rendering (the test-harness CronJobs, service special cases) moves to the fleet
  layer. That also settles harness: it is rendered by the fleet layer, not imported by the library.
- Found on the way and fixed in tacplus: `--src` overrides `depends`; a dependency's lock contributes wire
  facts only; a contract may import another service's proto; a service without `(spec.service)` is
  identified by package; operator.proto now declares its identity.

- 2026-10-24: `depends` is transitive (a contract's tacplus.yaml depends are followed). Every tacplus-dev
  repo now generates alone: kernel→opa; blobstore/letsencrypt/messaging/mailbox→stratum (record query
  plane) + eventbus (topics); letsencrypt→kernel; mailbox→messaging. Go state clients exist for sys state
  roots; the injected health state no longer counts as "sys state" (it had switched on the runtime-config
  subscription for every service). Operator (library) is green standalone.

- 2026-10-24: `service_s2s {auth_services_from_fleet: true}` — the fleet build stamps the caller list
  from declared clients (Jody: fleet build, not runtime config). doppelganger is a C2S fleet service
  calling llmproxy; **fleet-seatrials** is the proof fleet (8 services) on a disposable cluster;
  **tenantecho** (id 103) is the two-tenant gateway-routing proof, being written.

## Open

- Local client: a program that authenticates as a user, gets a token, and calls doppelganger through
  the gateway (Jody: "locally we will have a client that can auth and get a token"). Not started.
- Cluster create/destroy for fleet-seatrials, images, and the deployment of the 8 services + 2 tenant
  namespaces: the fleet-layer operator work (Admiral/Skipper shape).

- Operator still carries the launch fleet's composition in hand-written code (gateway TLS mode, eventbus
  node role, blobstore volume groups, example config, embedded registry.json, llmproxy test fixtures); its
  `depends` list is the inventory. Move to the fleet deployment repo / Skipper's tenant spec. For Skipper the
  library also needs an external-kernel input (address + CA) instead of bootstrapping kernel itself.

- Composition manifest: how a deployment declares which registered services it loads and how
  the kernel build fetches them (the registry gives name -> repo; the manifest and build step
  do not exist yet). This is what retires the interim "clone doppelganger into schematic" idea.
  **First requirement (Jody, 2026-10-23): the dependency runs from the kernel build to the
  services, never the other way.** *"this is where when kernel is built with support for
  Doppelganger it should follow the dependency"*: the manifest names doppelganger, doppelganger's
  registry entry names llmproxy, and the kernel build fetches both. Today the reverse is forced:
  generating llmproxy ALONE fails with `LlmProxyState declares (spec.service_state).label
  "kernel" but no loaded service serves that label`, so every service build must import kernel.proto
  from schematic's private platform proto. Fix: tacplus's sys contract declares that the kernel
  serves the state labels (the primitive every fleet provides); a service generates against its own
  proto plus sys; only the kernel build loads the services. Dependencies between services
  (doppelganger -> llmproxy) are declared on the registry entry or in tacplus.yaml and followed by
  the kernel build.
- Merge `registry-and-floor` into tacplus main and release a MAJOR.MINOR that carries it; then
  schematic's and sonar's `registry.yaml` files gain `global:` and become locks (no id changes).
- Tacplus quirk found on the way: with `pkg` omitted, the derived Go package is `<module>/tacplus`
  while the default output segment is `model`, so a consumer must set `output-name: tacplus` or the
  generated imports do not match the on-disk layout. Either derive with the output name or document it.
