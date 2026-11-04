# Before production

Jody, 2026-11-02: "we are getting ready for production not in production. keep a list of stuff that we will need to do
before we actual go production."

This is that list. tacplus-dev is **not in production**. Nothing here is started until Jody says it's time, unless an item
says it runs earlier. Every item says where it came from. The hub adds an item whenever a decision or finding defers
something "until production", and ticks it off with the commit or issue that did it.

## Cloud and access
- [ ] **The `prod` folder and project**, its own `cloud-prod-pipeline` identity, and the `apply-prod` gate (runs only on a
      commit already applied to `dev`; Jody plays it). Designed, not built (`decisions/cloud-pipeline-2026-11-02.md`, roadmap#2).
- [ ] **The production cluster** in `prod` (arm64 Axion), with its own cluster pipeline identity
      (`decisions/cluster-pipeline-2026-11-02.md`).
- [ ] **jody1-328023 drained and deleted** (Jody deletes it: data). SeaBeam's leftovers move to SeaBeam's own projects first
      (`decisions/cloud-pipeline-2026-11-02.md`).
- [ ] **No standing human or hub write access.** The hub's `jody-desktop@` key is gone, and the hub's project
      setIamPolicy and cluster-admin are removed. People are viewer and securityReviewer only. Break-glass is Jody's account,
      logged (roadmap#2).
- [ ] **Rotate every fleet secret before go-live.** Anything that existed while the old Owner token (`looper-runtime`, revoked 2026-10-30) could read protected variables is treated as exposed (suggestion rejected as done, 2026-11-03).
- [ ] **The hub has its own GitLab identity**, so no automation can play a job as Jody (caveat found on cluster#1, 2026-11-02).
- [ ] **Every GitLab setting from the cloud pipeline:** protected project variables, environments, runner registrations
      (roadmap#2).
- [ ] **Network exposure enforced by org policy:** VMs can't get external IPs (`compute.vmExternalIpAccess`), no ingress
      from 0.0.0.0/0 except to the load balancers, the control plane endpoint private, COS and no SSH everywhere
      (`decisions/network-exposure-2026-11-03.md`).
- [ ] **Org policies enforced:** no service-account keys, WIF from gitlab.com only, us-central1 only, uniform bucket access,
      domain-restricted sharing (roadmap#2).

## Identity and sign-in
- [ ] **Real identity providers** replace the Keycloak emulation, which is test only (Jody, 2026-11-01).
- [ ] **A login-continuity gate around every deploy:** same XID, links and memberships before and after, and sessions
      survive. "Logins must never break" (Jody, 2026-11-01).
- [ ] **The tenant-routing fix** shipped on every line: kernel stamps a `tid` claim; generated code reads only it
      (tacplus#284, kernel#32). The name guard is done (kernel#31).
- [ ] **The Anthropic federation rule** for llmproxy, set in the Anthropic Console once the fleet is live (hub handoff,
      2026-11-02). OpenAI and Vertex AI after Anthropic works end to end (Jody, 2026-11-02).

## Release and supply chain
- [ ] **Release-line images rebuilt weekly** from their own branches; patch tags build fresh images (tacplus#283).
- [ ] **No open HIGH or CRITICAL findings** in shipped images or dependencies, or an expiring, justified exception for each
      (looper#53 rule, Jody 2026-11-02).
- [ ] **Formal verification on every release tag and nightly** (tacplus#275). Rust: Kani harnesses work (tacplus#211). Java:
      OpenJML (tacplus#257, Jody: medium priority).
- [ ] **Nightly gates green** again (tacplus#282).
- [ ] **Production and dev fleets on released lines,** not branch commits (fleet-seatrials#33 question about the
      `tacplus_commit` pin).

## Reliability and operations
- [ ] **Chaos running in production**, opt-out per fleet: pod faults (deploymentproxy#10), then node faults; failures
      matched to recovered chaos faults are logged and ignored, others triaged (Jody, 2026-11-01; chaos#1).
- [ ] **Sandboxed runtime for agent containers** (gVisor or similar): containers share the VM's kernel, and agents run
      model-driven commands (looper redesign huddle, 2026-11-03; accepted risk until then).
- [ ] **Durable state backed up and restore tested:** kernel identity, stratum and eventbus. A restore drill, with a
      runbook, before production holds real data.
- [ ] **Multi-zone production:** a regional or multi-zone production cluster, with zone-aware Stratum backups. The zone
      must reach Stratum through the uni-config or a declared runtime fact, never downward-API env (operator#2,
      2026-11-02; dev is single-zone).
- [ ] **Monitoring and alerting** for the fleet and the pipelines, with who gets paged.
- [ ] **Runbooks:** deploy, rollback, restore, tenant deletion (`decisions/tenant-deletion-2026-10-27.md`), certificate
      problems (Let's Encrypt rate limits).
- [ ] **Capacity:** SSD quota measured and requested if needed (Jody skipped it 2026-11-02 until measurement); runner
      first-job failures fixed (looper#43).

## Deterministic deployment
- [ ] **Known-good uni-config promotion:** a verified uni-config is recorded immutably (by digest, secrets by version) and
      exactly that record is deployed to prod; rollback is an earlier record (`decisions/deterministic-deployments-2026-11-02.md`).
- [ ] **Reproducibility proven:** rebuild dev from the repositories, pins and secret versions alone, and compare.
- [ ] **Drift checks** on cloud, cluster and every fleet, with no unexplained differences.

## Release to production
- [ ] **Canary rollout for patches:** the latest patch of every pinned line deploys to prod automatically through
      canaries (canary slice, health judgement with chaos faults excluded, stepped widening, automatic rollback to the last
      known-good record). Only MAJOR.MINOR changes wait for Jody (`decisions/deterministic-deployments-2026-11-02.md`).
- [ ] **The approval gate for MAJOR.MINOR** in production pipelines: a line change is a protected manual job that only
      Jody plays.

## Before go-live
- [ ] **A security review** of the production layout: a round-table over cluster, cloud and identity.
- [ ] **Jody's go-live decision**, recorded in `decisions/`.
- [ ] Rolling Ratis upgrades: until then a Ratis minor bump (kernel#43, 3.2 -> 3.3) restarts the whole group at once; settle mixed-version groups or a planned group restart before go-live. (Hub, kernel#43, 2026-11-04)
