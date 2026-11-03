# Suggestions

Written nightly by the roadmap scan (tacplus-dev/looper#50), reviewed by Jody. Accepted ones go to a ROADMAP.md (by an agent at
Jody's request). Rejected ones stay here with the reasoning, and aren't raised again without new evidence. Types: [TYPES.md](TYPES.md).

## Open
| Suggestion | Type | Summary | Status |
|---|---|---|---|
| [Apply looper#25's narrowed credentials on the live pipeline](2026-11-03-apply-looper-25-s-narrowed-credentials-on-the-live-pipeline.md) | Security | Replace the looper jobs' GITLAB_TOKEN (Jody's group-Owner looper-runtime) with the read_api bot token and revoke it, move the workers off build-box, and have Jody decide fleet-seatrials' re-key | open (rank 1, 2026-11-03) |
| [Each looper model as its own unprivileged user](2026-11-03-each-looper-model-as-its-own-unprivileged-user.md) | Security | Run every child's model as a non-root user of its own, with docker reached only through looper-docker as a broker, so models can't read the parent's credentials or signal the lander and other children | open (rank 2, 2026-11-03) |
| [Plan conflicts by file, not by directory](2026-11-03-plan-conflicts-by-file-not-by-directory.md) | Performance | The parent's plan treats a 'touches' directory as conflicting with everything under it; have the plan name files, and hold two issues back only for a shared file, contract or named semantic conflict | open (rank 3, 2026-11-03) |
| [Start a late issue capped at the job's deadline instead of holding it](2026-11-03-start-a-late-issue-capped-at-the-job-s-deadline-instead-of-holding-it.md) | Performance | When a free slot opens but an issue's limit outlasts the job's time left, start it anyway with its limit capped at the job's deadline (the #35 path); it checkpoints and resumes in the next job | open (rank 4, 2026-11-03) |
| [The kicker and plan on a runner of their own](2026-11-03-the-kicker-and-plan-on-a-runner-of-their-own.md) | Performance | Move looper-kick and looper-plan off the 5-worker autoscaled pool (tags arm64, autoscale) onto a small always-on runner tag, so dispatch never queues behind hours-long looper jobs | open (rank 5, 2026-11-03) |

## Accepted
| Suggestion | Type | Summary | Status |
|---|---|---|---|

## Deferred
| Suggestion | Type | Summary | Status |
|---|---|---|---|

## Rejected
| Suggestion | Type | Summary | Status |
|---|---|---|---|
