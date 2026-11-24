# Loopers and builds leave the cluster: one spot VM per project (2026-11-24)

**Status:** direction decided (Jody, 2026-11-24, in the hub session). The migration plan below needs **Jody's approval before any change**; seatrials stays first in line.
**Supersedes:** `only-vms-are-k8s-nodes-2026-11-04.md` for the **build environment** (the deployment cluster still has only its own nodes), and the looper placement of 2026-11-07 and 2026-11-22 (loopers and builds as pods on the cluster's normal, looper and spot pools).

## Why
On 2026-11-24 the "loopers don't build" rule and in-cluster placement caused most of the day's delay:
- A report's feedback build goes through tacplus-dev/build as about 10 GitLab jobs. On GitLab.com's free plan the project caps at about 500 jobs in active pipelines, and **46 of 82 failed child pipelines between 12:00 and 21:00Z were GitLab refusing to create them**, not red. The managers sent those lands back as red, seatrials' last fix (fleet-seatrials#76) twice.
- Builds wait hours for 6 build slots, and reports can't read their own build logs, so a fix-and-verify cycle takes hours and needs the hub to read the traces.
- Managers and reports share one runner of 17 slots, so merges wait behind reports (cluster#60).

## Decision
1. **One looper+builder per project, on its own spot VM, outside the deployment cluster.** It runs in the CI project (tacplus-ci), never in the cluster that serves deployments. The build environment is kept apart from the deployment cluster.
2. **The agent never blocks.** A parent loop schedules and records state. Builds, tests and model calls run as **child processes**, as many as it wants. On a spot preemption notice (about 30 s), the parent stops its children and writes its state. Every child is safe to run again from the start (a build re-runs, a model call retries).
3. **The VM is completely ephemeral; its disk is only a cache** (Jody, 2026-11-24). Durable state lives where it already does: the work-in-progress branch pushed to the forge and the issue's checkpoint comments. A VM that finds the project's cache disk uses it (the checkout, the toolchain and build caches); a VM that finds none creates it and starts cold. Either way it reaches the same result, only slower. **It never gets stuck:** a missing, stale, half-written or corrupt cache is discarded and rebuilt, never waited on or repaired by hand. On a preemption notice the parent pushes its checkpoint; nothing is lost that is not also in the forge. No NFS, and nothing shared between projects.
4. **Loopers build for themselves**, as often as they like, and read their own logs. Their builds are feedback only.
5. **No SSH, in any form**, as before. The VMs come from the cloud pipeline (an instance group of 0..1 per project, Container-Optimized OS, the looper as a pinned container, no port 22). The kicker scales a project to 1 when it has work and to 0 when idle.
6. **Merging:** eventually a project never merges to main itself. It pushes a merge branch and opens a merge request, which a review panel and a separate merge-and-release build system (not yet assembled) handle. **Until then, today's managers and their gated builds stay as the interim merge path**, with no new investment beyond the fixes in flight.
7. **Security is not the focus of this step.** The worker VMs keep today's level; the control point becomes the future merge system.
8. **The deployment cluster shrinks to 3 small VMs** once loopers and builds have left: no spot build pool, no looper pool, no in-cluster CI runners. It holds deployments only.

9. **Each project specifies its own VM, in its repository, and it is right-sized over time** (Jody, 2026-11-24). A file in the repo (for example `looper.yaml`, beside `build.lock` and `gates.yaml`) names the project's ideal looper/builder: machine type (or CPU and memory), disk size, spot or not, and how many children may run at once. The cloud pipeline reads it to make that project's VM. It starts from measured builds (a Rust or Java project needs more than a small Go one) and is adjusted as use is measured: the nightly recap compares each project's measured peak CPU, memory and disk against its spec and files a change, up or down, when they drift. The change lands through that project's normal review like any other code.

## Migration plan (for Jody's approval; nothing changes before it)
Seatrials comes first: none of this starts until seatrials is deployed, unless Jody says otherwise.
1. **Design issue in tacplus-dev/looper:** the parent/child runner (`looper-vm`). It covers the state file format, the preemption handler, child re-runs, the per-project disk layout, and how a project VM takes its issues (the kicker's trigger, or the VM polling its own project).
2. **cloud (tacplus-ci):** an instance-group template per project, the cache disk optional and recreatable (spot, Arm, the machine size read from the project's own spec file, decision point 9), a persistent disk per project, no SSH and no port 22, egress through NAT only, the kicker's permission to resize, and the GitLab/GitHub credentials the looper already uses (Secret Manager through WIF, as today).
2b. **The spec file and the right-sizing loop:** the spec file's format (looper), an initial spec per repository from today's build measurements (each repo, one land), and the recap's drift check.
3. **Pilot on one project** (fleet-seatrials or redo) beside the current in-cluster looper. Compare time-to-feedback, preemption recovery and cost for a few days.
4. **Roll out the rest**, a few projects a day. Each project's in-cluster report jobs stop as its VM starts.
5. **Managers:** move the interim managers (and the kicker) to a small CI VM, or keep them on GitLab's runners until the merge system replaces them. To decide at step 4.
6. **Shrink the cluster:** remove the spot build pool, the looper pool and the in-cluster CI runners (control, kick, build, build-light, build-report). Size the remaining 3 nodes from the deployments' measured requests (seatrials' 13 services). The infra runner (cloud, cluster and deploy pipelines) moves to the CI VM side, or stays as one small runner. To decide at step 6.
7. **Retire** the runner VMs in jody1-328023 and gitlab-runner-mgr, the last interim VMs.

## In-flight work, and what happens to it
- **Kept, needed in the meantime:** looper#237 (a refused build is not red; at most 4 jobs a build) and looper#231 (managers log while they wait).
- **Kept until the migration, then removed with the pools:** cloud#107 and cluster#61 (looper pool to 4), and cluster#60, cloud#106 and looper#236 (managers' runner). They are the stopgap that keeps seatrials and tacplus merging this week.
- **Superseded by this decision:** no new work on in-cluster runner sizing (cluster#53's sizing follow-ups), the build-light and build-report split, or report-slot tuning.

## Touches
`only-vms-are-k8s-nodes-2026-11-04.md` (superseded for the build environment), `cluster-node-pools-2026-11-03.md`, `looper-job-per-issue-2026-11-03.md`, `security-gates-ship-only-2026-11-23.md`, `deploymentproxy-2026-11-02.md` (unchanged: the deployment cluster's privilege model), tacplus-dev/looper#86 (the merge train), #237, #231, cluster#60, cloud#107.
