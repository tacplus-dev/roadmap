Suggestion: A red build tells its report which gate failed and its first error
Type: Performance
Summary: cloud#45, top priority on the GKE path, was built five times and refused five times for one comment line that cloud's policy lint refuses. Each time its report saw only 'no verdict within 3h' or 'the build failed', because a report can't read tacplus-dev/build's logs. It took Jody reading the build job (05:08Z) to end it. Have the gates job write the failing gate and its first error line into build-verdict, carry it into the commit status, and have the manager quote it on the issue
Why: With looper#135 landed (07:00Z), a red build now reports, but its status says 'the build failed' whenever the gates' child pipeline failed (looper-build cmd_status). That covers every gate failure, which is where the report needs the reason. The manager's red comment quotes the tail of its own output (build URL, 'no verdict', git push noise), never the gate's. So a report can only guess, re-run and re-queue: cloud#45 was redone 10 times tonight (the recap's redone list for cloud: #45 ten times out of 12), for 23.5 h from its first land (08:57Z) to its merge (08:26Z), while cluster#21 and cloud#11 queued behind it on Jody's GKE order. 139 build gate_failures tonight carry the same blind spot. looper's PROPOSALS has this as an open entry ('The child pipeline's verdict as the commit status' description', #130), and #135 named it as its follow-up, but no issue holds it. Also serves Developer experience: a red gate that says why

## Evidence
- https://github.com/tacplus-dev/cloud/issues/45#issuecomment-6844722359 (hub, 2026-11-13 05:08Z: 'built five times and refused five times for the same line ... policy: refused: modules/projects/dev/cluster-pipeline.tf:79: names allUsers', job https://gitlab.com/tacplus-dev/build/-/jobs/17993164551; the manager's four red comments before it quote only 'the gates failed (exit 2) ... no verdict within 3h')
- https://github.com/tacplus-dev/looper/blob/423a103c4bf657a1b722ecc8e69a1b67502f310f/looper-build#L665 (cmd_status: with no build-verdict from the child pipeline the description is 'the build failed'), and repo-looper#L1329-L1331 (the red comment is the tail of the manager's own output)
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('The child pipeline's verdict as the commit status' description', 2026-11-11, #130: open, no issue); https://github.com/tacplus-dev/looper/issues/135 ('Follow-up (not this fix): carry the child gates job's verdict line into the status description, so the waiter reads why')
- recap 2026-11-13, manager/cloud/backlog: merge backlog p90 3, lands waited 16.07 h for the manager; managers.cloud: 21 lands, 9 merged, 4 red gates, redone [45 x10, 46, 48]; failures/tacplus-dev/build/gate_failure: 139 jobs, 0 recovered (https://gitlab.com/tacplus-dev/build/-/jobs/17967625493)

## Cost
About a day in tacplus-dev/looper, by a looper. looper-build's gates job writes 'fail: <gate set>: <the first line matching error|refused|FAIL|panic in gates.log, else its last line>', redacted and cut to 140 characters, into build-verdict. report-fail reads the child's build-verdict through the bridge's downstream pipeline with the job token (the PROPOSALS entry's way), and cmd_status posts it, with the gates job's URL as target_url instead of the pipeline's. repo-looper's red verdict quotes the status description and links the job. Tests in tests/looper-build.test.sh and tests/repo-looper.test.sh. No money; nothing secret leaves, since the status is on the repository's own commit and redact already runs on the manager's comments

## Touches
decisions/throughput-first-2026-11-11.md (merges first; Jody's GKE order on cloud#45, 2026-11-12), looper#71 (gates as build jobs whose logs a report cannot read), looper#135 (its follow-up), looper#130 (the PROPOSALS entry), looper#127 (curl -f hides why: the same class of silence), looper#86 (a batch's bisect needs the same reason per member)

## Reuse considered
The build-verdict file, the commit status and the manager's verdict comment all exist, so this only carries one more line through them. Considered: (1) giving reports read access to tacplus-dev/build's job logs: the logs hold more than the verdict, and the build project is kept read-only to the looper by design. (2) Having the report run the gates locally before landing: it does, but the build's gate set and image differ (the lint ran only in the build), which is the point of building. (3) The recap: it sees the failure class, not the line, and only the next morning

## Advice
Recap: 2026-11-13 manager/cloud/backlog
Cause: a red build's reason stops in tacplus-dev/build: the status says 'the build failed' and the manager's comment quotes its own output, so cloud#45's report re-ran and re-queued the same failing line five times (10 redone runs) until the hub read the job log
Change: tacplus-dev/looper: looper-build's gates job writes the failing gate and its first error into build-verdict, report-fail carries the child's verdict into the commit status description (target_url: the gates job), repo-looper's red verdict quotes it (an issue from PROPOSALS' 'The child pipeline's verdict as the commit status' description')
Gain: most of cloud#45's 23.5 h from first land to merge (4 of its 5 red builds, about 12 manager-hours at 3 h each) and of the 16.07 h cloud's lands waited; across repositories, a red land fixed in one round instead of several (139 gate_failures tonight)

## Status
open
Rank: 3
Filed: 2026-11-13 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17999306134)
