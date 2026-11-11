Suggestion: A build nobody waits for any more does not run its gates
Type: Performance
Summary: When a manager or report gives up on a build ('no verdict within 3h'), it deletes the build branch but leaves the pipeline queued. The build fetches its commit by SHA, so the abandoned gates still run later on the 2-VM build pool, ahead of the retried lands that wait 3 h in turn. Have looper-build's gates job end at once, as 'abandoned', when its pipeline is older than the requester's LOOPER_BUILD_WAIT
Why: tacplus-dev/build's queue was over its threshold a second night: p90 131.2 min, max 402.4 min over 209 jobs, with every pinned repository's lands and reports building on the 'tacplus-ci build' pool (113 jobs, 2 VMs). The requester's wait is 180 min (LOOPER_BUILD_WAIT 10800). So a build can start more than 3 h after its request, when no one reads its verdict any more. Tonight the managers of cloud, blobstore, opa, operator and llmproxy all requested at about 02:00Z and all gave up between 05:00 and 05:25Z with 'no verdict within 3h' (fleet-seatrials#42 at 09:25Z). gateway#29 has had four lands end this way (looper#113). Each give-up re-lands later and requests a new build, which then queues behind its own abandoned one. repo-looper's build_gates deletes looper/build-<repo>-<n> after the wait, but nothing cancels the pipeline, and looper-build fetches BUILD_SHA by hash, so the deleted branch does not stop it. looper#113 is about builds that never post a status. This is the load those builds leave behind, which stays after #113 lands and after #84 moves builds to spot pods. Also serves Cost (spot build time spent on verdicts nobody reads).

## Evidence
- https://gitlab.com/tacplus-dev/build/-/jobs (recap 2026-11-11, queue/tacplus-dev/build, recurring 2 nights: queue time p90 131.2 min over 209 jobs, p50 17.2, max 402.4 (24142 s) against the requester's LOOPER_BUILD_WAIT of 10800 s. runner_pools 'tacplus-ci build': 113 jobs on 2 VMs, 29 failed before ready. its failures tonight: 30 runner_system_failure, 21 script_failure, 21 gate_failure)
- https://github.com/tacplus-dev/looper/blob/53011292ebb197c35f72af5cb0b717929766cfb2/repo-looper#L1069-L1097 (build_verdict returns 2 on 'no verdict within' BUILD_WAIT; build_gates then runs 'git push -q origin --delete looper/build-REPO-N' and nothing cancels the pipeline; looper-build fetch_commit fetches 'origin BUILD_SHA' by hash, so the deleted branch does not stop the build)
- https://github.com/tacplus-dev/opa/issues/10 (manager job https://gitlab.com/tacplus-dev/looper/-/jobs/17920632071: 'landing looper/opa-10 ... 02:01:29Z', 'no verdict within 3h' 05:01:42Z. The same on blobstore#18 (02:24 to 05:25Z), operator#26 (02:00 to 05:00Z), llmproxy#25 (02:03 to 05:03Z) and cloud#29 (02:01 to 05:01Z, pipeline 3052481207))
- https://github.com/tacplus-dev/looper/issues/113 (comments of 2026-11-11: opa#9's and gateway#29's builds set 'pending building' at 02:21Z and 02:22Z, and never changed: 'the job ran, the gates never got a slot'; gateway#29 four lands ended this way)

## Cost
Hours: looper-build's gates job (the child pipeline's) first compares CI_PIPELINE_CREATED_AT with now. Past a new input, wait (the requester's LOOPER_BUILD_WAIT, which start_build passes beside repo, sha, gates and base), it writes the verdict 'failure: abandoned: no verdict wanted after Nh' and runs no gates. A nightly, a release and a verify run pass no wait and are never abandoned. Tests in tests/looper-build.test.sh: an old pipeline exits at once, a fresh one builds. No credential is needed: the trigger token cannot cancel a pipeline, and the build job can end itself.

## Touches
decisions/builds-in-containers-2026-11-03.md and decisions/merge-train-2026-11-04.md (builds async on spot: abandoned work is spot time), decisions/only-vms-are-k8s-nodes-2026-11-04.md's addendum of 2026-11-07 (loopers all at once on the always-on nodes, builds async on spot), looper#71 (the manager's gates as build jobs), looper#113 (builds that never report), looper#84 (build jobs on spot with retry:), looper#86 (the merge train, whose one build per batch makes an abandoned batch build costlier still)

## Reuse considered
Considered: (1) the requester cancels its pipeline: it holds only BUILD_TRIGGER_TOKEN, which cannot cancel, and giving the manager a GitLab write token on tacplus-dev/build widens the manager's credentials (looper#90, #83). (2) GitLab's interruptible: only for newer pipelines on the same ref, and each build runs on main with inputs, so it would cancel unrelated builds. (3) A longer LOOPER_BUILD_WAIT: it holds the manager and every land behind it longer. (4) More build VMs: capacity is manual and VMs only as k8s nodes (decisions/only-vms-are-k8s-nodes-2026-11-04.md). A self-check in the build job uses what GitLab already gives every job (CI_PIPELINE_CREATED_AT). Not in an open issue: looper#113 is about the missing status, cloud#29 about the pools' stuck VMs.

## Advice
Recap: 2026-11-11 queue/tacplus-dev/build
Cause: a build whose requester stopped waiting after LOOPER_BUILD_WAIT (3 h) still runs: build_gates deletes its branch but cannot cancel the pipeline, and the build fetches the commit by SHA, so on a 2-VM build pool (queue max 402.4 min) abandoned gates hold the slots the re-landed builds then wait for
Change: looper-build gates: end as 'abandoned' when CI_PIPELINE_CREATED_AT is older than a new build input wait, which repo-looper start_build passes (LOOPER_BUILD_WAIT); never for nightly, release or verify runs
Gain: every build started past its requester's 3 h runs no gates: on tonight's queue (p90 131.2 min, max 402.4 over 209 jobs), the full gates of each land the managers gave up on (at least 6 tonight: cloud#29, blobstore#18, opa#10, operator#26, llmproxy#25, fleet-seatrials#42, 30 to 60 min each, run p90 1810 s) go back to the builds someone waits for

## Status
open
Rank: 2
Filed: 2026-11-11 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17928642546)
