# Looper redesign: one job per issue, a manager per repo (Jody, 2026-11-03)

The huddle record is `items/looper-job-per-issue.md`; this is what it decided. With `builds-in-containers-2026-11-03.md`.
It started from the first nightly recap: tacplus issues waited 365 h in a day (conflict 156, no free slot 149, limit
outlasts job 59).

## Decided
- **One GitLab job per issue** (a "direct report"), on its own branch, in its own container. Jody proposed it.
- **A manager per repo owns `main`** (Jody: "Each repo has a manager. That manager has direct reports. If there is a problem
  we are going to hold that manager responsible ... they should probably be the one doing any merges. Now if they want to
  delegate who am I to say no."):
  - **the only merger:** it merges its reports' branches and deletes them;
  - **conflicts are its call:** resolve them itself (regenerate lock and generated files rather than hand-merge them), or
    send the branch back to its report with the files named;
  - **the guardrail:** nothing reaches `main` without green gates;
  - **logged:** every merge, resolution or send-back is recorded on the issue;
  - **scored nightly by the recap:** report-to-merge time, conflicts resolved versus sent back, reverts or a red `main`,
    redone work.
- **The cap on concurrent reports:** 6 for tacplus, 3 for each other repo, in a committed config (Jody: "a is fine. but recap
  should let us know if we need to adjust"). The recap reports at-cap waiting, saturation and merge backlog, and advises a
  change when a threshold is crossed two nights running.
- **Agents never build** (`builds-in-containers-2026-11-03.md`). A report asks for a build, and the build runs as its own job
  in a build container. Images are built with rootless BuildKit. Caches live in the `ci` bucket and registry, keyed by content,
  and a cold cache gives the same verdict as a warm one.
- **Containers share VMs** (Jody: "why should we have a vm per job? we are running things as containers"):
  - each job container gets CPU, memory and disk limits;
  - no host socket, no privileged mode, no capabilities, a non-root user, a read-only root where possible;
  - containers and volumes are discarded per job, so VMs are reused;
  - the shared-kernel risk is accepted for now (gVisor is on PRODUCTION.md).
- **Capacity is manual** (Jody: "we have a vm generally running all the time ... somebody tells me and we spin up more vms
  and then spin them back down later ... I need to watch cost right now"):
  - one always-on VM normally runs everything, the control jobs included;
  - scaling up happens on Jody's word, with numbers; scaling down needs no ask.
  - The pools were cut on 2026-11-03.
- **Cache warming:** once the shared cache exists, briefly scale up and run a complete build of every repo to fill it, then
  scale back to one (Jody's proposal).
- **The suggestions:**
  - "Plan conflicts by file" and "Start a late issue capped at the deadline" are **superseded**: there are no conflict or
    time-left holds to tune any more;
  - "The kicker and plan on a runner of their own" is **folded in**: control jobs on the always-on VM.

## Issues
- tacplus-dev/looper: job per issue plus the manager; build requests in place of host-socket gates; the recap's manager
  and capacity reporting.
- tacplus-dev/tacplus: rootless BuildKit and cache configuration in the build image.
- tacplus-dev/cloud: the `ci` cache bucket and registry, and pools whose containers share VMs, with one always-on VM.
