# The cloud pipeline owns the cloud accounts; people read only (Jody, 2026-11-02)

Jody, 2026-11-02: "One thing we also need to do is create a pipeline that will manage the cloud account and then read only
for us. Everything change will come from the cloud."

The hub asked two questions. Jody answered:
1. "yes, then we will cleanup and remove the old": tacplus-dev gets **new** GCP projects, built by the pipeline from the
   first day. jody1-328023 is cleaned up and removed once everything has moved off it.
2. "yes": GitLab settings (projects, protected variables, environments, runner registrations) are managed the same way.

And: "we need to prepare ourselves for eventual production."

## Decided

- **Every change to cloud and CI configuration comes from a pipeline.** People, the hub included, hold read-only access.
  The only exceptions are the bootstrap below and break-glass access (Jody's account, logged).
- **New GCP projects for tacplus-dev**, built and owned by the pipeline. jody1-328023 is not imported: it is drained and
  then deleted, by Jody, since a project deletion is a data deletion.
- **GitLab settings under the same pipeline**, now.
- **Production is planned in from the start**, not added later: production gets its own project(s) and its own approval.

## Open at the time (the design issue, approved below)

- The tool, the layout of organization, folders and projects, the bootstrap, and the production gate:
  tacplus-dev/roadmap#2.

## Approved (Jody, 2026-11-02)

Jody approved the design in roadmap#2: "approve. and we are getting ready for production not in production. keep a list of
stuff that we will need to do before we actual go production."

- **OpenTofu**, the `google` and `gitlab` providers, state in GCS.
- **A Cloud Identity Free organization on tacplus.dev**, with the org policies listed in roadmap#2.
- **Folders and projects:** `bootstrap`, `ci` and `dev` are **built now**. `prod` is **designed but not built**: its project,
  its separate `cloud-prod-pipeline` identity and the `apply-prod` gate wait for the production list (`PRODUCTION.md`).
- **Ordering:** finish the first tacplus.dev bring-up on jody1-328023. Build the cloud pipeline in parallel, then move
  dev over.
- **The production list** is kept in `PRODUCTION.md` at the root of this repository. Every item says where it came from.
  The hub adds to it whenever a decision or finding defers something "until production".
