Suggestion: The recap counts a failed Actions run as recovered only by a success on the same ref
Type: Operability
Summary: The recap reported tacplus/Images as '4 of 5 runs failed, a later run succeeded', but the success was v2.6.4. The tags that failed at 17:03Z (v2.1.15 to v2.5.15, five in GitHub against the recap's four) published no image, and neither did 11 earlier tags since 2026-11-08 00:03. Have looper-recap match a failure's recovery on the same workflow and ref, and name the refs still red
Why: Jody asked to be told when something needs adjusting (looper#72). Tonight the recap said the opposite: it marked tacplus/Images recovered at 0 h of waiting, while releases v2.1.13 to v2.5.15 on five lines (plus v2.6.3) had no image. The cause is that the recap counts any later success of the workflow, whichever tag or branch it ran on. The failures are themselves tracked (tacplus#283 diagnoses the Swift 6.1.3 plugins on main's base:latest), so the gap is only the recap's. A release workflow runs once per tag, though, so the next per-line break would be hidden the same way. Also serves Correctness: the recap's 'recovered' then means what it says.

## Evidence
- https://github.com/tacplus-dev/tacplus/actions/runs/42502995632 (Images, v2.5.15, 2026-11-08 17:03Z: build (amd64) smoke test, 'protoc-gen-swift: symbol lookup error'; v2.1.15, v2.2.12, v2.3.14 and v2.4.14 the same; v2.6.4 succeeded; also v2.1.13 to v2.5.13 at 00:03Z, and v2.1.14 to v2.5.14 and v2.6.3 at 09:02Z, all failed)
- https://github.com/tacplus-dev/tacplus/actions/runs/42502987123 (recap 2026-11-08, actions/tacplus/Images: '4 of 5 runs failed, a later run succeeded', recovered true, wait_hours 0)
- https://github.com/tacplus-dev/looper/blob/main/looper-recap (actions: rec = any success with a later id within the same repo/workflow, the run's head_branch not compared)
- https://github.com/tacplus-dev/tacplus/issues/283 (open: the tag runs fall back to main's base:latest, Swift 6.4.2 against plugins built with 6.1.3; the looper cannot dispatch the line rebuild, 403)

## Cost
Hours in looper: looper-recap keeps each run's head_branch. A failure is recovered only by a later success with the same workflow and head_branch, or for main, a later success on main. The summary names the refs still red. Tests in tests/looper-recap.test.sh: failed tags followed by a success on another tag are not recovered; a re-run that succeeds on the same tag is.

## Touches
looper#57 and #72 (the recap: 'if builds are backing up somebody tells me'), tacplus#283 (the failures themselves), decisions/tacplus-images-2026-10-27.md (an image per release line)

## Reuse considered
The recap's GitLab failures already count recovery per job (a retry of the same job), so this applies the same rule to Actions. image-failed (looper#58) opens an issue for a failed looper image on main, but tacplus's tags have no such step. That is a possible follow-up, not proposed here, since #283 already tracks this break.

## Advice
Recap: 2026-11-08 actions/tacplus/Images
Cause: looper-recap marks an Actions workflow recovered when any later run of it succeeded, on any ref, so per-tag release failures look recovered once another line's tag passes
Change: looper-recap's actions(): recovered only by a later success on the same head_branch; the summary lists the refs left red
Gain: 5 release-image failures tonight (16 since 2026-11-08 00:03Z, v2.6.3 on another cause: cargo kani setup on arm64) reported as unrecovered instead of hidden, so a broken release line reaches Jody's daily report the night it breaks

## Status
open
Rank: 3
Filed: 2026-11-09 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17842078231)
