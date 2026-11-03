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

## Open
- Fresh VM per job (`max_use_count = 1`) for direct reports and build jobs. Hub recommends it.
