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

## Open (the design issue, for Jody's approval)

- The tool, the layout of organization, folders and projects, the bootstrap, and the production gate:
  tacplus-dev/roadmap#2.
