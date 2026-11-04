Suggestion: Let the recap see the build pipeline
Type: Operability
Summary: The nightly recap has been blind to tacplus-dev/build (HTTP 404) and Cloud Build (PERMISSION_DENIED) two nights running; make tacplus-looper-bot a Reporter of tacplus-dev/build and give looper-runner Cloud Build read, or drop the Cloud Build source if nothing builds there
Why: Jody asked the recap to say when builds are backing up (looper#72: 'if builds are backing up somebody tells me'). Its build data comes from tacplus-dev/build, where the build jobs (#71), the nightly formal verification (#74) and the security checks (#49) are all moving. Yet both nights it has run, it could read only tacplus-dev/looper: the bot behind GITLAB_TOKEN is a Reporter of tacplus-dev/looper alone, and the scan's looper-runner has no Cloud Build role. As builds move into tacplus-dev/build, a queue there would look like nothing at all, not like a backlog. Reading another project needs only its Reporter role, which can't read CI variables or start pipelines.

## Evidence
- https://github.com/tacplus-dev/roadmap/blob/main/suggestions/recap/2026-11-03.md (Not read: cloud-build 'PERMISSION_DENIED ... authenticated as looper-runner@jody1-328023'; gitlab:tacplus-dev/build and gitlab:tacplus-dev/cluster 'GET projects/.../jobs: 404')
- https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356 (recap 2026-11-04, sources: the same 3 of 7 sources unavailable again, cloud_build null, and github-main partial on an HTTP 500; 0 build-project jobs read on either night)
- https://github.com/tacplus-dev/looper/blob/main/README.md#credentials (GITLAB_TOKEN: tacplus-looper-bot, read_api, 'a Reporter of tacplus-dev/looper alone ... it reaches no other project')
- https://github.com/tacplus-dev/looper/blob/main/STATE.md (Known gaps: the recap 'reads Cloud Build only where the scan job's gcloud is logged in'; build jobs #71 and the security checks #49 run in tacplus-dev/build)

## Cost
Minutes for the hub: add tacplus-looper-bot as a Reporter of tacplus-dev/build. For tacplus-dev/cluster, only after checking that its plan artifacts are developer-only (artifacts:access), since a Reporter can download artifacts; never tacplus-dev/cloud, whose saved plans carry secret values. Either grant looper-runner roles/cloudbuild.builds.viewer on jody1-328023, or remove cloud-build from looper-recap's sources if nothing is built there any more (the build pipeline replaced it). Then one recap run shows the sources ok.

## Touches
looper#57 and looper#72 (the recap; Jody: 'if builds are backing up somebody tells me'), looper#25 (least-privilege credentials: a Reporter reads, nothing more), looper#71 and #74 (builds and nightly verification in tacplus-dev/build), looper#49 (security checks there)

## Reuse considered
looper-recap already collects any GitLab project it can read: it tried tacplus-dev/build and cluster and recorded the 404s. No code is needed, only the membership. The read_api bot is the narrow identity #25 made, and widening it by one Reporter membership keeps it read-only. A separate token for the build project would be another secret to rotate ('Rotate the bots' tokens' in looper's PROPOSALS).

## Advice
Recap: 2026-11-04 sources
Cause: the recap's own sources: GITLAB_TOKEN's bot is a Reporter of tacplus-dev/looper alone (tacplus-dev/build and cluster answer 404), and looper-runner holds no Cloud Build role (PERMISSION_DENIED), so 3 of its 7 sources are unavailable both nights
Change: the hub: tacplus-looper-bot a Reporter of tacplus-dev/build (and of tacplus-dev/cluster once its artifacts are developer-only); roles/cloudbuild.builds.viewer for looper-runner on jody1-328023, or cloud-build removed from looper-recap's sources
Gain: no hours tonight; the recap sees every build job's queue and failures as they move to tacplus-dev/build (#71, #74, #49), instead of 0 jobs read there on both nights

## Status
accepted (Jody, 2026-11-04): tacplus-dev/cloud#12, tacplus-dev/looper#85; Cloud Build dropped instead of granted
Rank: 4
Filed: 2026-11-04 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356)
