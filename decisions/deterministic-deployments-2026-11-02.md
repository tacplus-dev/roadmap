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
