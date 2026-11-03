# Every deployment is deployed and managed through deterministic methods (Jody, 2026-11-02)

Jody, 2026-11-02: "We also need to make sure we have this rule in place. all deployments must be deployed and managed via
deterministic methods. AI and me can do anything we need to do but we must managed thru deterministic methods. everything
must be reproducable. Once we have a known good uni-config that must be take and deployed."

It extends `deterministic-builds-2026-10-27.md` (a build is fully determined by its commit) from builds to deployments.

## The rule

- **Every deployment is changed only through a deterministic path.** That means a committed, versioned input, applied by a
  pipeline, with the same input giving the same result. It covers the cloud accounts (tacplus-dev/cloud), the clusters
  (tacplus-dev/cluster) and every fleet (its own pipeline and uni-config).
- **People and AI may do anything they need to.** They may investigate, experiment, debug and try fixes. But a change that
  is to stay is **made deterministic**: it is committed and applied by the pipeline. Anything done by hand to unblock
  something is either captured as a commit and re-applied by the pipeline, or reverted. The nightly drift checks report
  anything that differs from the committed state.
- **Everything is reproducible:**
  - from the repositories, the pins (tool versions, image digests, release lines) and the referenced secrets, the
    deployment can be rebuilt identically;
  - nothing resolves "latest" at deploy time;
  - nothing depends on state that exists only in someone's shell or an agent's session.

## Known-good uni-config

- A deployment is defined entirely by its **uni-config** (tacplus-dev/operator#17).
- **Recorded as known good:** once a uni-config has been deployed and verified (the deployment's tests and gates pass), it
  is recorded as known good. The record is immutable and identified by a digest. It holds the non-secret document with
  every image pinned by digest, and every secret named **by version**.
- **Deployed as it is:** a known-good uni-config is taken and deployed exactly as recorded. It is not re-assembled from
  newer inputs on the way. Promotion to the next environment deploys that same record. Rollback redeploys an earlier one.

## Consequences (the hub's reading; Jody can change these)

- **Secrets must be versioned.** Today the uni-config is assembled in a job's memory from GitLab protected variables, which
  can change underneath it. For a known-good record to be reproducible, it names each secret by an immutable version
  (Secret Manager versions, managed by tacplus-dev/cloud). It never embeds the value: the record stays non-secret.
- **The operator applies only a recorded uni-config:**
  - a new one is recorded first, as a candidate;
  - it becomes known good only after verification;
  - promotion takes a known-good digest.
- **The loopers and the hub never deploy by hand.** This is already the deploy-authority rule; this rule makes the reason
  explicit.

## Where it lands

- tacplus-dev/operator#17 (the uni-config) and tacplus-dev/fleet-seatrials#28 (assembling and applying it): versioned
  secret references, a candidate → known-good record, and deploy-by-digest.
- tacplus-dev/cloud#1 and tacplus-dev/cluster: already plan → apply of a committed input; drift checks report hand changes.
- `PRODUCTION.md`: promotion of a known-good uni-config from dev to prod is a gate before production.

## Addendum: deployment and tenant uni-configs never mix (Jody, 2026-11-02)

Jody: "Another rule - the uni-config must contain everything for the deployment except tenants which have their own
uni-configs. we dont have tenant configs in our config and vice versa. we can uplift tenant config into our state but not
the config."

- **The deployment's uni-config holds everything about the deployment, and nothing about any tenant.** It covers services,
  pins, settings, identity and trust material, platform secrets, and the deployment's own sign-in.
- **Each tenant has its own uni-config.** It holds that tenant's configuration (its settings, its tenant-owned services, its
  own sign-in providers and secrets), and nothing about the deployment or other tenants.
- **Never across:**
  - no tenant configuration in the deployment's uni-config, and none of the deployment's in a tenant's;
  - a deployment's config never lists, embeds or overrides a tenant's settings.
- **Uplift into state, not config.** The platform may read a tenant's uni-config and reflect what it needs into the
  deployment's **state**: a tenant record, its tid, its placement. That state is derived and never written back into the
  deployment's config document.
- **The rules above apply to tenant uni-configs too:**
  - deterministic and reproducible;
  - recorded by digest, with secrets by version;
  - a known-good record is deployed as it is;
  - tenant secrets stay out of the deployment's secrets and vice versa.

**Settled 2026-11-02 (addendum "where the configs live" below). Was open:** where a tenant's uni-config lives and what applies it. Candidates are a tenant pipeline, the
tenant's Skipper (`skipper-and-admiral-2026-11-01.md`), or the kernel's existing `SetTenantConfig` path. Who may author it
also matters, given the tenant's own users and "tenants talk to their Skipper".

## Addendum: no magic environment; the uni-config is the config (Jody, 2026-11-02)

Jody: "no magic env vars set by k8s, etc. the uni-config is the config. so that if an LLM is helping us it can get a
redacted copy and know what should be happening."

- **No configuration arrives any other way.** No environment variables set by Kubernetes or anything else:
  - Kubernetes' service-link variables (`<SVC>_SERVICE_HOST` and the like) are off: `enableServiceLinks: false`;
  - no downward-API env;
  - no `envFrom` ConfigMaps or Secrets;
  - no image-baked defaults that change behaviour.
- **One fixed place.** A service learns everything from its part of the uni-config, delivered through the generated config
  path at one fixed, well-known file location. The location is a platform constant, not an environment variable
  (tacplus#262).
- **Explicit defaults.** Defaults are declared in the config's types or the uni-config itself, never hidden in code or
  platform behaviour. If a value matters, it's visible in the uni-config.
- **A redacted copy for reading.** Because the uni-config is complete, a **redacted copy** tells a reader (a person or an
  LLM helping us) what the deployment should be doing. Redaction is generated from the types, not hand-written:
  tacplus#274's `Redact<T>`, with sensitive fields annotated. Secret values never appear; the secret **version references**
  do, so the reader knows which secret is in use without seeing it. Each recorded uni-config (candidate or known good) gets
  a redacted twin that the pipeline can hand to an assistant.
- **Applies to tenant uni-configs the same way.** A tenant's redacted copy is shown only within that tenant's scope.

## Addendum: what a redacted value looks like (Jody, 2026-11-02)

Jody: "I dont see why skipper needs to actually see secrets. but this does bring up something. we should probably replace
a redacted string with the word "REDACTED"". This confirms that the Skipper gets the redacted twin only, like the Admiral.

- **A set sensitive string or bytes field becomes the literal `REDACTED`.** An empty or unset one **stays empty**, so a
  reader can tell "a secret is configured" from "none is".
- **Only string and bytes fields** (and messages, by marking their string and bytes leaves) may be annotated sensitive. The
  gate refuses the annotation on numbers, booleans and enums, because no replacement value there could be told apart from
  a real one.
- **Redaction is deterministic.** The same input gives the same output, so redacted twins of two records can be diffed.

## Addendum: required secrets are checked for entropy (Jody, 2026-11-02)

Jody: "We already validate types which naturally includes configs. A secret can be required. If it is required and it is
set we should check it for entropy."

The hub's concrete reading, adjustable by Jody:
- **`required` on a sensitive field:** an unset or empty value fails validation, as for any required field.
- **When it is set, it must clear a minimum entropy estimate.** The default is **128 bits**; a field may raise it.
  - bytes: length × 8;
  - strings: estimated conservatively as length × log2(the size of the character classes actually present).
- **Known placeholders always fail:** `REDACTED` (so a redacted twin can never be deployed as a real config), `changeme`,
  and single-character repeats.
- **Errors name the field and the rule, never the value.**
- **It runs in the generated validators** (every language, deterministic, no network), so it applies both when the
  pipeline assembles a uni-config and when a service loads it.
- **What it can't do:** the estimate shows a value *looks* random. Secrets we generate get their real guarantee from a
  cryptographic random source in the init job.

## Addendum: approval only for production (Jody, 2026-11-02)

Jody, asked why he must play `apply`: "I only want approval for production and we are not production. Dont wait for me".

- **Outside production, nothing waits for Jody.** Cloud (`ci`, `dev`), the cluster pipeline and every non-production fleet
  pipeline **apply automatically** on `main`, once the plan for that commit succeeds. The saved plan is what gets applied,
  and the run is traced and logged.
- **Production keeps Jody's approval:** `apply-prod` and any production fleet deploy are protected manual jobs that only
  Jody plays (`PRODUCTION.md`).
- **Deleting a deployment** stays a deliberate deletion pipeline, started on purpose with the deployment named
  (tacplus-dev/cluster#2). That isn't an approval gate; it's intent. The hub never starts one: deleting data is Jody's.
- **Until the pipelines are changed,** the hub plays the existing manual dev jobs itself, on Jody's word above.

This supersedes "Jody plays apply" in `cluster-pipeline-2026-11-02.md` and `cloud-pipeline-2026-11-02.md` for everything
except production.

### Refined: production approves MAJOR.MINOR only; patches roll out by canary (Jody, 2026-11-02)

Jody: "And let me take some of that back. I want approval on changes to production MAJOR.MINOR. I always want the latest
patch deployed using canaries."

- **Production needs Jody's approval only for a change of a pinned line,** MAJOR.MINOR: a new minor or major, of tacplus
  or of any service or component.
- **The latest patch of every pinned line deploys to production automatically, by canary.** No approval is needed.
  Patches are fixes only (the versioning rules), so this is how fixes, security ones first, reach production.
  - The new patch goes to a canary slice first. It's judged on health, error rate and the fleet's checks, with chaos faults
    told apart from real failures (`chaos`), then widened in steps.
  - It rolls back automatically, to the previous known-good record, if the canary fails.
  - Every step is traced. A failed canary is reported and the patch is held.
- **Everything is still deterministic.** Each step deploys a recorded uni-config, by digest. Picking up "the latest patch"
  is a recorded pin bump, a new candidate record, never "latest" resolved at deploy time.
- **Production infrastructure changes** (prod cloud, prod cluster) keep Jody's approval, as `apply-prod`.

### Corrected: env vars are fine if the uni-config produces them (Jody, 2026-11-02)

Jody: "I dont mind using env vars. They just need to come from the uni-config and not be created directly". This replaces
the "no environment variables" wording in the no-magic-environment addendum above.

- **Allowed:** an environment variable that the operator renders **from the uni-config**, where the uni-config names it
  and its value. It's then visible in the uni-config and its redacted twin like any other setting. Secrets rendered this
  way follow the same redaction.
- **Not allowed: environment created directly**, from anywhere other than the uni-config:
  - Kubernetes' service-link variables (pods still set `enableServiceLinks: false`);
  - downward-API env (e.g. a pod's zone or node label);
  - hand-written `env` or `envFrom` in manifests;
  - image-baked defaults that change behaviour.
- **A test** over the rendered manifests: every env var on every pod traces back to a uni-config field.

## Addendum: where the configs live (Jody, 2026-11-02); settles the open huddle above

Jody: "There is the system config and tenant configs. The system config will reside in kernel state. The tenant configs
reside in per tenant kernel state. The system config is not writable by the platform at all. We do need to handle
refreshing but that is already implemented AFAIK. The only thing we have left is the tenant config at rest. The tenant
config is of course mutable within the platform. We do need the tenant config to persist of course. I think it goes in
stratum but I could see a future where it does not and exist as a file"

- **System config (the deployment's uni-config)** lives in **kernel state**.
  - It's **not writable by the platform at all.** No RPC changes it. Only the deployment pipeline sets it, through the
    operator, which applies a recorded uni-config.
  - Services pick up a new one through the existing config refresh.
  - The full deterministic rule applies: recorded, known good, deployed as it is.
- **Tenant configs** live in **each tenant's own kernel state**, and are **mutable within the platform.**
  - This corrects the earlier "the rules above apply to tenant uni-configs too": tenant configs are **not**
    pipeline-deployed.
  - They're reproducible through **history**: every write is a new version, and any past version can be read or restored.
    Each version has its redacted twin.
  - Writes are typed, authorized, traced and accounted.
- **Tenant config at rest: the tenant's own Stratum** (per tenant since operator#2). A file is a possible future backing
  store, behind the same interface, so callers don't change.
- **`GetConfig`, the tenant side:** a Skipper reads its tenant's config (redacted) from that tenant's kernel state. The
  Admiral still asks the Skipper (`skipper-and-admiral-2026-11-01.md`).
- **The uplift: tenant OAuth config becomes oauth's state** (Jody: "There is tenant config that we bring up into our state
  which is tenant OAUTH config. by our I mean system". Corrected: "no it does not get saved in system. system is all state
  the config comes from the pipeline. tenants have config and we uplift the relevant config into our state. This means we
  grab the per tenant oauth config (could be more in the future) and bring it into oauth and it becomes state").
  - **Nothing tenant-related is saved in the system.** The system's config comes only from the pipeline. Everything else
    the system has is state.
  - **The mechanism already exists in tacplus** (docs/USER.md, "Per-Tenant Config Held in a State Root"):
    - oauth declares a tenant-scope `(spec.config_slice)`;
    - the kernel holds each tenant's `TenantConfig`;
    - tacplus generates a read-only, server-enforced Get/Watch subscription for each slice owner, projected to its own slice;
    - a `/system` caller sees every tenant's entry, and a tenant caller only its own.
  - **oauth watches its slices and holds them as in-memory state.** More uplifts later ("could be more in the future") are
    more `config_slice` declarations, not new mechanisms.
