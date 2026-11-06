Suggestion: The in-cluster control runner on tags of its own, or registered paused
Type: Operability
Summary: cluster's control runner registers with the VM runners' tags (arm64, autoscale, autoscale-16), and the hub's step 7 registers it active. From then on GitLab gives any looper job to whichever runner asks first, and a report that lands in a pod has no Docker and at most 4Gi. Give it the normal-pool tag looper#84 plans (e.g. arm64, control), or register it paused until #84 switches
Why: looper#84 says the plan moves to 'the in-cluster runner's normal-pool tags' and drops the old ones (autoscale, autoscale-16). tacplus-dev/cluster does the opposite: its control runner takes exactly the old tags, and its README says jobs move once 'the VM runners holding the same tags are paused'. Its hub step 7, though, registers the runner with its tags and nothing about pausing it. If the runners are registered before #84 lands, the first symptom is reports, managers and scans failing at random. The ones that land in a pod find no Docker socket (no privileged pods, by cluster#6's admission policy), so every gate that runs tacplus or looper-docker fails, and a tacplus report gets 2Gi, raisable to 4Gi, where looper-docker gives Kani 12 GB. That would come on top of last night's 73 runner_system_failure jobs (30 never recovered), and read as a flaky runner rather than a routing mistake. The two repositories disagree today; cheaper to settle in a line now than in an incident during the move. Also serves Correctness: the docs of the two repositories agree.

## Evidence
- https://github.com/tacplus-dev/cluster/blob/main/gitlab-runner/runners/control.yaml (fbcc43a, 2026-11-05: 'tags: [arm64, autoscale, autoscale-16]', concurrent 8; memory request 512Mi, limit 2Gi, overwrite max 4Gi; 'No privilege ... no Docker socket')
- https://github.com/tacplus-dev/cluster/blob/main/README.md (What the hub does, 7: each runner 'created in GitLab as a group runner of tacplus-dev with its tacplus.tags'; Moving jobs over: 'runs here once these runners are online and the VM runners holding the same tags are paused')
- https://github.com/tacplus-dev/looper/issues/84 (open: 'the kicker, reports, managers and scans use the in-cluster runner's normal-pool tags'; 'The old runner VM tags (autoscale, autoscale-16, gce) are removed from the plan')
- https://github.com/tacplus-dev/looper/blob/main/ci/tacplus-dev.gitlab-ci.yml (looper-kick, looper-plan and every planned job tagged arm64, autoscale; LOOPER_JOB_TAGS_tacplus 'arm64, autoscale-16'): a job's tags are a subset of the control runner's, so it matches both runners
- https://gitlab.com/tacplus-dev/looper/-/jobs/17768002509 (recap 2026-11-06, failures/tacplus-dev/looper/runner_system_failure: 73 jobs, 43 recovered on retry, 36.57 h, 3 nights running; the noise a mis-routed job would hide in)

## Cost
Under an hour. Either a one-line change in cluster's gitlab-runner/runners/control.yaml (tags: [arm64, control], say), the README's table and 'Moving jobs over' updated, with looper#84's plan using the same tag; or step 7 registers both runners paused (GitLab's paused flag on the runner) and #84's switch unpauses them as it pauses the VM runners. The tag change is the safer one: it needs no one to remember a pause, and a job can only reach a pod once the plan asks for it.

## Touches
looper#84 (the switch), tacplus-dev/cluster#6 (CI in the cluster, no privileged pods), decisions/only-vms-are-k8s-nodes-2026-11-04.md and its addendum (GitLab's way: tags and per-job overrides decide placement), decisions/looper-job-per-issue-2026-11-03.md, the rule that broken work never lands

## Reuse considered
GitLab's own mechanisms: tags (a job runs only where its tags are a subset of the runner's) and the runner's paused state, so nothing is built. The build runner's tags (arm64, build) are those looper's README gives the build VM runner, which is not applied yet (STATE: 'no build runs until the hub applies the build runner'). Sharing them matters less there: a build job holds no agent and needs no Docker except for image builds, which wait on tacplus#315 anyway. Pausing the VM runners at the switch, as cluster's README plans, still works as a rollback either way.

## Status
open
Rank: 2
Filed: 2026-11-06 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17802225217)
