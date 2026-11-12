Suggestion: A new runner manager build drains the old manager before it is deleted
Type: Operability
Summary: At 03:48Z, 10 looper jobs died at once with runner_interrupted (23.11 h lost): five managers, tacplus's verdict, the kick and three reports. Cause: apply:ci replaced the tacplus-ci runner manager. cloud#46 now makes the replacement create_before_destroy, but cloud's own STATE.md says the jobs on the old manager are still lost when it goes. Have the old manager stop taking jobs and finish its running ones (gitlab-runner's graceful SIGQUIT) before its delete
Why: Since the pools moved to tacplus-ci, every looper job runs through one runner manager VM, and it is replaced on each new build: runner version, plugin, startup script or image. Three such changes landed in cloud in a day (#40 step 2, d30423c; #44, e40e196, runner v19.5.3 kept in step with cluster#21; #46, b15a1c9). More are coming as Go advisories move the pinned runner (cloud#45, cluster#21). #46 fixed the outage (destroy-first, no manager at all), but not the lost jobs: cloud STATE.md says the first apply with it 'replaces the manager once (new name), and that apply's own job dies when the old manager goes', and a lost apply can leave the state locked (README, Never without a manager). Each interrupted manager loses its lands' place and has to be re-kicked. Tonight that was opa's, looper's and operator's, after they had queued 657 min for a slot, the night's longest queues. Under throughput first, merges are what's scarce. Also serves Resiliency

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/17934383222 (recap 2026-11-12, failures/tacplus-dev/looper/other:runner_interrupted: 10 jobs, 0 recovered on retry, 23.11 h, 2 of 7 nights; every one finished 03:48:09-03:48:26Z: looper-opa-manager here, looper-looper-manager 17934383225, looper-operator-manager 17934383227, looper-cloud-manager 17955013678, looper-cluster-manager 17954518570, looper-tacplus-verdict 17947653711, looper-kick 17954902940, looper-cloud-44 17955013679, looper-cloud-45 17955013680, looper-cluster-21 17954518571)
- https://github.com/tacplus-dev/cloud/issues/46 (closed 2026-11-12: apply:ci replaced the manager destroy-first at 03:47Z, pipeline 3057435167, about 40 running looper jobs dropped to 0; fix b15a1c9, create_before_destroy under runner-manager-<hash>)
- https://github.com/tacplus-dev/cloud/blob/main/PROPOSALS.md ('Drain the old runner manager before it goes', #46, 2026-11-12: SIGQUIT instead of SIGTERM, or a terraform_data the old instance's destroy waits on until it has no job; and 'Two managers on the same groups, briefly')
- https://github.com/tacplus-dev/cloud/blob/main/STATE.md (Never without a runner manager, #46: not yet seen live; 'that apply's own job dies when the old manager goes')

## Cost
About a day in tacplus-dev/cloud, by a looper, with no hub step (throughput-first point 4 keeps hub changes to cluster#20, and this is pipeline code). manager-startup.sh gets a shutdown path that sends gitlab-runner SIGQUIT and waits. runners.tf gets a terraform_data between the new manager's create and the old one's destroy, which pauses the old one's intake and polls until it has no running job, bounded by the longest job limit (11 h). ci/policy.sh refuses a manager replacement without it. Tests with the existing stubs (tests/stub/manager). Settle the two-managers-on-one-group question in the same issue. No money

## Touches
cloud#46 (never without a manager: its unfinished half), cloud#11 and decisions/only-vms-are-k8s-nodes-2026-11-04.md (the manager goes with the VMs in the end; until then it is every looper job's single point), decisions/throughput-first-2026-11-11.md, looper#84 (the in-cluster runner, where a runner Deployment's rolling update drains by terminationGracePeriod), cluster#21 and cloud#45 (the next runner bumps)

## Reuse considered
cloud's PROPOSALS already names both ways to do it, so this promotes a proposal rather than inventing one. Considered: (1) applying build changes only in a quiet window: by hand, and against throughput first's no-hub-changes rule. (2) Waiting for cloud#11 or looper#84: the decided end state, but runner bumps keep coming before it. (3) GitLab Runner's own graceful stop (SIGQUIT, shutdown_timeout) is the mechanism; only the delete has to wait for it

## Advice
Recap: 2026-11-12 failures/tacplus-dev/looper/other:runner_interrupted
Cause: apply:ci replaced the tacplus-ci runner manager VM, and the old manager's running jobs died with it (10 jobs at 03:48Z). cloud#46 removed the outage (destroy-first) but keeps the loss: a create_before_destroy replacement still deletes the old manager under its running jobs
Change: tacplus-dev/cloud: modules/projects/ci/files/manager-startup.sh gets a SIGQUIT drain on shutdown, and modules/projects/ci/runners.tf a terraform_data the old manager's destroy depends on, which waits until it has no running job (bounded at 11 h). ci/policy.sh refuses a manager replacement without the drain (an issue from cloud's PROPOSALS entry)
Gain: the 23.11 h of job time lost tonight (10 jobs, 0 recovered), per manager replacement; in-flight managers keep their lands' place instead of re-queueing (opa, looper and operator had queued 657 min each before they were cut)

## Status
open
Rank: 2
Filed: 2026-11-12 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17962167524)
