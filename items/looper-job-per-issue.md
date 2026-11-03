# Looper redesign: one job per issue, a manager per repo (huddle, 2026-11-03)

- **Status:** in huddle with Jody. Decided points are marked. Issues follow when the huddle ends.
- **Started from:** suggestion "Plan conflicts by file, not by directory" (Jody: "I think accept but let's huddle"). The first
  nightly recap (suggestions/recap/2026-11-03.md): tacplus issues waited 365 h (conflict 156, no free slot 149, limit
  outlasts job 59).

## Decided so far
- **One GitLab job per issue,** each on its own branch and its own VM (Jody's proposal).
- **Every build in a container; vanilla COS VMs; agents never build; rootless BuildKit; caches in `ci`**
  (`decisions/builds-in-containers-2026-11-03.md`).
- **A manager per repo** (Jody: "Each repo has a manager. That manager has direct reports. If there is a problem we are
  going to hold that manager responsible ... that manager is responsible for main I think they should probably be the one
  doing any merges. Now if they want to delegate who am I to say no."):
  - the manager alone merges to `main`, and deletes merged branches;
  - conflicts are its call: resolve them itself, or send the branch back to its direct report with the files named;
  - **guardrail:** nothing reaches `main` without green gates;
  - **every decision is logged on the issue;**
  - the recap scores each manager: report-to-merge time, conflicts resolved versus sent back, reverts or a red `main`,
    redone work.
- **The cap on concurrent direct reports:** 6 for tacplus, 3 for each other repo, in a committed config (Jody: "a is fine.
  but recap should let us know if we need to adjust"). The recap reports at-cap waiting, saturation and merge backlog per
  repo, and advises a change when a threshold is crossed two nights running.
- **Control jobs** (kicker, planner, recap, drift) are deterministic and model-free, and share one small always-on VM.
  That's suggestion 5, folded in. Suggestions 3 and 4 are superseded.

- **Containers, not VMs per job** (Jody: "why should we have a vm per job? we are running things as containers"). With no
  host socket and no privileged mode, several jobs share a VM. Each gets CPU, memory and disk limits, a read-only root where
  possible, no capabilities and a non-root user. Containers and volumes are discarded per job, so VM reuse is fine. The
  shared-kernel risk is accepted for now (Jody: "not worried about that right now"); gVisor for agent containers goes on
  PRODUCTION.md.
- **Capacity and cost** (Jody: "we have a vm generally running all the time. if builds are backing up somebody tells me
  and we spin up more vms and then spin them back down later. Later we can completely automate this but I need to watch
  cost right now"):
  - one always-on VM normally runs everything in containers;
  - **scaling up is on Jody's word.** The hub (each watch pass) and the recap (nightly) tell him with numbers when ready
    work backs up past a threshold (for example a queue p90 over 30 min): issues waiting, queue time, the cost of one
    more VM;
  - scaling back down when the backlog clears needs no ask;
  - full automation comes later.

## Open
- Whether to cut today's autoscaler (up to 7 runner VMs) to the one-VM model now.
