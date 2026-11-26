# The agentic development flow, top to bottom

Status: **agreed in outline by Jody on 2026-11-25** (a huddle), then reviewed by a panel of three models (`agentic-dev-flow-review-2026-11-25/summation.md`). The section "Changes after the panel" below is part of the design and wins over the text after it where they differ; five points the panel did not settle, a new machine and the order of work wait on Jody. It replaces the merge train (`merge-train-2026-11-04.md`) and extends `looper-build-vms-2026-11-24.md`. Where the two disagree with this page, this page is the newer.

## Changes after the panel (2026-11-25)

All three reviewers kept the direction and found the same holes. These are adopted; the numbers refer to the summation.

1. **Gates first.** The slowest gate group and `pkg/**` mapping to everything are the largest bottleneck left. Sharding `current-generation` and per-language gate sets are step 1 of the migration (1).
2. **Builds on a VM are bounded.** A limit on builds running at once per VM, the wait shown, and a cap on builds and spend per issue. Capacity is sized from measurement (2).
3. **A looper builds on `main-green`**, the last commit the rolling check found green, never on main's tip, and need not rebase before opening an MR. Its evidence records its base (3).
4. **The merger is a small fixed program, not an agent.** It runs on a machine that runs no project code. It merges clean MRs and sends a conflicting MR back to its report, which resolves it. This replaces "the merger resolves conflicts" in the flow below (4).
5. **Trust.** The looper's VM holds neither the main-writing credential nor the accepting one, and build containers there must not reach the VM's metadata server. The merger trusts only a status a report cannot mint; a release trusts only its own independent rebuild (5, 6).
6. **Clean merges are unbuilt only inside a limit**: a bound on the merges main may hold unchecked, and no merging while the rolling check is red, down, bisecting or stale (7).
7. **The rolling check has its own reserved machine**, not spot, not a looper's VM, not the GitLab build project (8). Capacity: Jody's to approve.
8. **Recovery is a fixed procedure**: stop merging; run again to tell a flake from a break; revert only an isolated merge whose reverted tree builds; otherwise one "fix main" issue for one looper. A red no shipped change caused reverts nothing. A reverted change returns as a revert of the revert or as new commits (9, 10).
9. **Gate scoping fails closed**; the map is derived from the dependency graph and never rewritten by a red check (11).
10. **A release publishes exactly the commit that was checked**, release branches have a green status of their own, an older build cannot deploy after a newer one, and a red release files an issue (12).
11. **One run per issue**, claims that survive a restart for merges, reverts and deploys, and a watchdog on every hand-off (13). The cache disk's lifetime, leftover containers and checkpoint time are owned (14).

**Not settled by the panel** (summation, D1 to D5): a quick compile before a clean merge; the size of the unchecked window; what proves a revert; when tacplus's consumers are checked; the panel's time limit.

## Why

On 2026-11-25 tacplus landed 5 changes in 13 hours. The cause was not capacity. Three things set the pace:

1. **A looper learned of a failure late.** A report queued its change after a 5-minute unit gate. The real gates (`just gates`, 50 to 80 minutes) first ran in the merge train's batch build, outside the looper's control.
2. **One bad change stalled everything.** A red batch was rebuilt change by change (seven hour-long builds at once gave no verdict in three hours), then bisected: about six hours for one bad change.
3. **The machinery was the product.** The train, its managers, a build project on another forge and a chain of pins between three repositories each failed that week. One fix to the machinery took about three hours to reach production.

## Principles

1. **A looper never waits on something outside its control.** It builds for itself, on its own machine, as often as it likes.
2. **Finding out fast beats being right first time.** A change may reach main and be found bad afterwards, as long as that is found within one build and undone automatically.
3. **Main may be broken briefly. A deployment never is.** Only the output of a green release build is ever deployed.
4. **Every step is one of: a looper's own work, an agent's decision, or a pipeline.** No step waits on a person, except the ones Jody keeps (below).
5. **Jody is asked to push back on, and be told of, anything that makes a queue.** Every serial step in this page states how long it takes.

## The parts

| Part | What it is | Where it runs |
|---|---|---|
| **Issue** | The unit of work. An issue labelled `approved` is ready; a feature is approved by Jody, a bug by triage. | GitHub (tacplus-dev) |
| **Kicker** | A small scheduled job. When a project has a ready issue and its VM is off, it starts the VM. It decides no work. | GitLab schedule, in-cluster runner |
| **Project VM** | One spot VM per heavy project (tacplus), one shared VM per org for the rest (tacplus-dev; SeaBeam later). Stock Container-Optimized OS, one container, no SSH, a cache disk that is only a cache. It sizes itself to zero when idle. | tacplus-ci (Google Cloud), outside the deployment cluster |
| **Looper** | The parent process on the VM. It lists its projects' ready issues and runs one child per issue, as many at once as it likes. It never blocks. | The VM's container |
| **Report** | A looper's child: one model run that works one issue in its own checkout, commits, builds, and opens a merge request. | The VM |
| **Builder** | Not a service: a local build. A report asks for the gates of a commit and the looper runs them in the project's pinned build image, as a sibling container through the VM's container socket, sharing the VM's dependency caches. | The VM |
| **Merge request (MR)** | A report's finished work: its branch, its description, and its evidence (which gates it ran, on which base commit, and their result). | GitHub pull request |
| **Merger** | One agent per project. It takes the project's open MRs one at a time and merges them. It resolves conflicts itself. | The project's VM (see Open questions, 1) |
| **Rolling check** | A build of main's head, the full gates, started whenever main has moved since the last one, back to back. It publishes nothing. | Its own job (see Open questions, 3) |
| **Revert agent** | When the rolling check goes red, it finds the merge that broke main and reverts it. | With the rolling check |
| **Release build** | The build that publishes: images and artifacts, by digest, attested. Main: nightly, and on a "build main now" trigger. A protected release branch (`release/X.Y`): when it has something new. | The build pipeline |
| **Deploy pipeline** | Rolls a deployment to a release build's output. Deterministic, no agent. | Per deployment (fleet-seatrials for dev) |
| **Panel** | Several vendors' models (Anthropic, OpenAI, others) reviewing an MR, accept or reject. **Not built until after the first release** (Jody: the spend waits). | Later |

## The flow

1. **An issue becomes ready.** The kicker sees it and starts the project's VM if it is off (about 3 minutes to a working looper, less with a warm cache disk).
2. **The looper starts a report for it**, at once, beside every other issue it is working. Nothing is queued.
3. **The report works the issue.** It edits, commits and builds locally as it goes. A local build is the gate set the change's diff calls for (the project's `gates.yaml` maps paths to sets), not a token check. The report sees its own failures in minutes and fixes them. It waits on nothing but its own build.
4. **The report opens an MR** when its gates are green on its commits rebased onto current main. The MR says which gate set ran, on which main commit, and links the log. A report whose gates are red opens no MR. A build with no verdict (the VM was preempted) is run again, never waited on.
5. **The merger merges it.** One MR at a time per project:
   - **Clean merge** (the MR applies to current main with no conflict): merged at once, **with no build**. Seconds. The looper's own green build and the rolling check behind it are the protection.
   - **Conflict**: the merger resolves it, builds the gates the resolved files map to, locally, and merges on green. Minutes to one gate's length. If it cannot resolve it, the MR goes back to its issue with the conflict, and the issue's next report takes it up.
   - Until the panel exists, there is no review step here. When it exists, the panel accepts or rejects the MR before the merger takes it, and re-reviews only a conflict resolution (the difference between what it accepted and what would land), never the whole day's merges.
6. **The rolling check builds main.** If main moved since its last build, it builds main's head, the full gates. Green: nothing more. Red: the revert agent bisects the merges since the last green head (a handful, because the check runs back to back), reverts the one that broke main, and puts the failure on that change's issue, which becomes ready again with its branch kept. Main is red for about one build plus a short bisect, not a day.
7. **The release build publishes.** Nightly for main, at any time by the "build main now" trigger, and for a release branch when it has something new. It builds a commit the rolling check found green, and publishes images by digest. A red release build publishes nothing.
8. **The deploy pipeline deploys.** Dev (seatrials) rolls to each green main release build. Production rolls from a green build of its protected release branch, through canary. A version change of production's branch is Jody's.

## What a looper waits on, and for how long

| Step | Waits on | Today (2026-11-25) | In this flow |
|---|---|---|---|
| Start work on a ready issue | The VM | minutes | minutes |
| Know its change is good | Its own local build | 50 to 80 min, in another system's queue, after queueing | The gate set of its diff, on its own VM; target under 15 min for most changes (see Gates) |
| Get merged | The merger | 1 to 6+ hours (batch build, red handling) | Seconds when clean |
| Learn it broke main | The rolling check | Up to a day (nightly), or never | About one full build |

## Gates

The gate is the speed limit of the whole flow: a looper's feedback, a conflict's rebuild and the rolling check are all one gate long.

- **Split.** tacplus's `just gates` is about 19 independent groups run by one job. Run as parallel jobs its wall time is the slowest group (about 30 minutes today, `current-generation`), not the sum.
- **Scope.** A report and the merger build only the sets a diff can break (`gates.yaml`). Today `pkg/**` maps to everything; per-language sets are the work that brings most changes under 15 minutes. The rolling check always builds everything, so a wrong map is found within one build, not shipped.
- **Cache.** The VM's dependency and compiler caches are shared by every build on it and outlive a build. A cold build is the exception.

## Failure handling

| What fails | What happens | Who waits |
|---|---|---|
| A report's local build | The report fixes it. No MR. | Only that report |
| The VM is preempted (spot) | Each report checkpoints its branch within 25 s; the kicker starts a new VM; reports resume from the forge. | Minutes, that project |
| An MR conflicts | The merger resolves and rebuilds, or sends it back | Only that MR |
| The rolling check goes red | Bisect, revert, issue reopened with the failure | Loopers of that project build on a red main for about one build |
| The rolling check is red for a reason no merge caused (a flake, an outside dependency) | See Open questions, 4 | |
| A release build goes red | Nothing is published; the deployment stays on the last green one | Nobody |
| A deployment fails | The pipeline's own rollback; nothing in this flow changes | |

## What Jody keeps

Approving features. Production: its release branch's version, and anything that touches it. Spend and capacity. Design. Nothing else in this flow waits on a person.

## What this retires, and what stays

**Retired:** the managers and the merge train (batches, red rounds, bisecting batches); the pre-land gate; `looper:landing`; builds for loopers and merges in the GitLab build project; the pin of that project's CI file through two other repositories.

**Stays:** issues and their labels; the kicker; the VMs and the looper on them; `gates.yaml`; the pinned build image per project; the build pipeline, for release builds and (unless the panel says otherwise) the rolling check; deterministic deploy pipelines; no SSH, no secrets in metadata, only load balancers public.

## Migration, in order

1. **Gates**: `current-generation` sharded and `pkg/**` split into per-language sets, then the groups in parallel, measured as a shadow run on real changes (required by nothing). The panel's first priority.
2. **MRs**: a report on a VM opens a pull request with its evidence instead of queueing a land.
3. **The merger**, per project, with clean merges unbuilt.
4. **The rolling check and the revert agent.** Not before step 3 is the only way onto main, and not after: steps 3 and 4 go live together, or main has no protection.
5. **Release builds by branch**: nightly main, the build-now trigger, release branches. Dev deploys from them.
6. **Retire** the managers, the train and the GitLab land builds.
7. **After the first release: the panel.**

## Open questions for the panel

1. **Where does the merger run, and as whom?** It must write main. Today a VM's account may not hold the identity that can write main, on purpose: a VM runs model-written code with the container socket, so anything on it can take the VM's credentials. Options: (a) the merger on the project's VM with the main-writing identity, accepted as the build environment's risk; (b) the merger on a small VM of its own that runs no project code and no builds, and asks the project VM for a conflict's rebuild; (c) the forge merges: the merger only pushes to MR branches and sets an "accepted" status, and GitHub's own merge (auto-merge or its merge queue with no required build) writes main. The author's recommendation is (c), with (b) for conflict resolution.
2. **Is "no build on a clean merge" sound?** Two MRs each green on an older main can break main together with no textual conflict. The flow accepts that and relies on the rolling check and the revert. Is there a cheap check worth adding to the merger (for example, a compile of the touched languages) that does not rebuild the train?
3. **Where does the rolling check run?** Not on a project's VM (it must not compete with the loopers, and its verdict should not come from the box that wrote the code). The build pipeline's runners, or a VM of its own per heavy project?
4. **Reverts.** What does the revert agent do with a flaky gate, a failure no single merge explains, a revert that itself conflicts, or a change other merges already build on? When does it stop and leave main red for a looper to fix forward?
5. **Evidence.** What must an MR carry so that a merger (and later a panel) can trust a looper's own build: the log, a signed attestation from the VM, a rebuild by someone else on a sample?
6. **Gate scoping.** How do we know a per-language gate set covers what a path can break, other than the rolling check finding out?
7. **Several repositories.** Service repositories consume tacplus releases (the latest patch of a pinned MAJOR.MINOR). Does anything in this flow need to change for a change that spans tacplus and a service?
8. **The panel, later.** How does a multi-vendor review avoid becoming the new queue: parallel reviewers, a time limit, a rule for disagreement?
9. **What is missing?** Any step here that will queue under load, and any failure with no owner.

## Appendix: what exists on 2026-11-25

- tacplus's VM (`looper-vm-tacplus`, c4a-standard-8 spot) runs reports and local builds; its reports build their change's own gate set before queueing (looper#247, deployed that day, not yet measured).
- A shared VM for the org's other repositories (`looper-vm-tacplus-dev`) was created that day and is unverified.
- Merging is still the merge train: a manager per repository in GitLab, batching queued branches, one full build a batch in the GitLab build project.
- Release builds: images build on a weekly schedule and nightly per supported line (tacplus#966, landed that day). There is no rolling check of main and no automatic revert.
- Dev deployment (seatrials) is not yet deployed; its apply moves to an in-cluster operator (fleet-seatrials#76, open).
