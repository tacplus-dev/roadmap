Suggestion: The GitLab projects read looper main's CI files at each pipeline, not a copy the hub PUTs by hand
Type: Operability
Summary: tacplus-dev/build's .gitlab-ci.yml is looper's ci/build.gitlab-ci.yml 'copied there as is', and the looper project's is ci/tacplus-dev.gitlab-ci.yml, both by the hub's curl PUT after a land (README). So a merged change to the pipeline waits for a hand: looper#195 (merged 2026-11-19 09:30Z, the report input and BUILD_REPORT_SLOTS) 'takes effect once the hub copies ci/build.gitlab-ci.yml' and starts a build again without its input when GitLab refuses it; #185's nightly slots (2026-11-18), #173's retry, #172's tags and the wait input (#130, #158: 'once this project's copy declares it') went the same way, and STATE.md records the looper project's copy 'still a691d64's (2026-11-03) until the hub copies main's'. Set each project's CI/CD configuration file to the raw file on looper's main (GitLab's remote configuration path, from cloud's gitlab stack), or to the commit the image pin names and moved with it, so a land of ci/ is live at the next pipeline and the copy step and its drift are gone
Why: Tonight's build queue (suggestion 3) is the one #195 and #185 reorder in favour of the lands, and neither is in the pipeline that ran. Every change to the pipeline files is a model's land on a manager-only main with green gates, the same trust the image already carries (the plan's jobs take main's attested image); the hub's copy adds a step nothing prompts and a file that drifts. Also serves Performance (the lands' builds go first the night the fix merges) and Security (one source for what the build project runs, 'reviewed and landed like the code')

## Evidence
- https://github.com/tacplus-dev/looper/blob/main/README.md (The pipeline file: 'reviewed and landed like the code, and copied into that project as is, on its protected main, by the hub (like ci/build.gitlab-ci.yml and ci/instance.gitlab-ci.yml)', then the curl -X PUT .../repository/files/.gitlab-ci.yml step; Build pipeline: 'whose .gitlab-ci.yml is ci/build.gitlab-ci.yml here, copied there as is')
- https://github.com/tacplus-dev/looper/blob/main/ci/build.gitlab-ci.yml (its own header: the wait input 'repo-looper passes it once this project's copy declares it' (looper#130, #158); 'report: true is a report's --build (looper#195)')
- https://github.com/tacplus-dev/looper/issues/195 (closed 2026-11-19 09:30Z: 'When GitLab refuses the input because the build project's CI file hasn't been copied yet, repo-looper starts the build again without it'); https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('Report builds' slots, after the copy', 2026-11-19, #195: 'takes effect once the hub copies ci/build.gitlab-ci.yml')
- https://github.com/tacplus-dev/looper/blob/main/STATE.md ('the file on GitLab is still a691d64's until the hub copies main's': https://github.com/tacplus-dev/looper/commit/a691d64 is of 2026-11-03; 'the hub's one-time step' for process_mode=newest_first, #104)
- https://github.com/tacplus-dev/looper/commits/main/ci/build.gitlab-ci.yml (0557b8a #195 2026-11-19, 33cf306 #185 2026-11-18, 97af996 #174 and a8fcb99 #173 2026-11-17, 62d00a2 #172 and 89fe34a #164 2026-11-16: six pipeline changes in four days, each waiting for a PUT)
- recap 2026-11-19 manager/looper/merge-time: report to merge p90 358.8 min over 4 merges, build queue max 311.6 min, lands waited 10.46 h (https://gitlab.com/tacplus-dev/looper/-/jobs/18173621242); queue/tacplus-dev/build p90 117 min over 817 jobs, 7 of 7 nights (/builds/tacplus-dev/looper/looper-runs/20261119T093741Z-scan/context/recap.json)

## Cost
Hours: cloud's gitlab stack sets each project's CI configuration path to the raw URL of the file on looper's main (or at the commit the image pin names, moved with it), with a plan test; the README's PUT step retired; looper-ci-lint already checks the files as GitLab reads them (#94)

## Touches
decisions/cloud-pipeline-2026-11-02.md (every GitLab setting from a pipeline, people read only), decisions/deterministic-deployments-2026-11-02.md, the accepted 2026-11-04 suggestion (only the manager writes main: what main holds is what runs), looper#56 (the pipeline file lives in looper), looper#94 (the CI files linted as GitLab reads them), the open 2026-11-18 suggestion (the image half of the same gap)

## Reuse considered
GitLab reads a project's CI configuration from a remote URL (the project's CI/CD configuration file setting), and cloud's gitlab stack already manages the projects' settings from a pipeline. Considered: a job in looper's pipeline that PUTs the file on a merge touching ci/ (a write credential on the build project for a looper job, which the 2026-11-05 decision keeps out of jobs); leaving the copy to the hub (six changes in four days waited for a hand nothing prompts); pinning the path to the image pin's commit (keeps the two in step, and moves the copy's lag onto the pin's)

## Advice
Recap: 2026-11-19 manager/looper/merge-time
Cause: looper's batch of #194, #195 and #196 queued 311.6 min for its build behind report builds: the land-first slots #195 merged at 09:30Z, and #185's nightly slots of 2026-11-18, are in ci/build.gitlab-ci.yml on looper's main, which tacplus-dev/build runs only once the hub copies it by hand
Change: tacplus-dev/cloud, the gitlab stack: the build project's and the looper project's CI/CD configuration file set to looper main's ci/build.gitlab-ci.yml and ci/tacplus-dev.gitlab-ci.yml (or the commit the image pin names); looper's README drops the PUT
Gain: up to the 10.46 h looper's lands waited for a build slot tonight (its batch build queued 311.6 min) once #195's land-first slots run in the pipeline that builds; every later fix to ci/ live the night it merges instead of after a PUT

## Status
open
Rank: 4
Filed: 2026-11-19 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18176428236)
