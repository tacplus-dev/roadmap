# Suggestions

Written nightly by the roadmap scan (tacplus-dev/looper#50), reviewed by Jody. Accepted ones go to a ROADMAP.md (by an agent at
Jody's request). Rejected ones stay here with the reasoning, and aren't raised again without new evidence. Types: [TYPES.md](TYPES.md).

## Open
| Suggestion | Type | Summary | Status |
|---|---|---|---|
| [Only the manager can write main](2026-11-04-only-the-manager-can-write-main.md) | Security | A ruleset on main in the 28 public repositories, with only a manager identity allowed to update it, and reports pushing branches with a token that is not Jody's, so 'nothing reaches main without green gates' holds against a model | accepted (Jody, 2026-11-04): looper#83 |
| [The deterministic security checks in the pipelines that apply infrastructure](2026-11-04-the-deterministic-security-checks-in-the-pipelines-that-apply-infrastructure.md) | Security | Run looper#49's floor (OSV-Scanner, govulncheck -mode=binary on pinned tools, gitleaks, Trivy on the CI image, an enforced security-exceptions.yaml) as a job in cloud's, cluster's and fleet-seatrials' own GitLab pipelines, which tacplus-dev/build's checks never reach | accepted (Jody, 2026-11-04): cloud#10, cluster#5, fleet-seatrials#36 |
| [Fill the tacplus VM before adding one: four reports a VM on the 16-CPU pool](2026-11-04-fill-the-tacplus-vm-before-adding-one-four-reports-a-vm-on-the-16-cpu-pool.md) | Performance | tacplus's cap is 6 reports, but its 16-CPU pool gives each job a whole VM (capacity_per_instance = 1, set for the retired parent's 5 children a job), so its reports run one at a time and waited up to 4.7 h for a VM; pack four a VM first, then decide on a second VM with the numbers | open (rank 3, 2026-11-04) |
| [Let the recap see the build pipeline](2026-11-04-let-the-recap-see-the-build-pipeline.md) | Operability | The nightly recap has been blind to tacplus-dev/build (HTTP 404) and Cloud Build (PERMISSION_DENIED) two nights running; make tacplus-looper-bot a Reporter of tacplus-dev/build and give looper-runner Cloud Build read, or drop the Cloud Build source if nothing builds there | open (rank 4, 2026-11-04) |
| [Bring the looper's roadmap up to the redesign](2026-11-04-bring-the-looper-s-roadmap-up-to-the-redesign.md) | Documentation | looper's ROADMAP.md still says the kicker runs on the Mac and the image is the 2.4 build environment, and its Next lists 10 closed issues, among them #32's parent-and-children design that #70 retired; the platform ROADMAP still says one looper per repository. Move the done items to Now and mark #32 retired | open (rank 5, 2026-11-04) |

## Accepted
| Suggestion | Type | Summary | Status |
|---|---|---|---|
| [Each looper model as its own unprivileged user](2026-11-03-each-looper-model-as-its-own-unprivileged-user.md) | Security | Run every child's model as a non-root user of its own, with docker reached only through looper-docker as a broker, so models can't read the parent's credentials or signal the lander and other children | accepted (2026-11-03) |
| [The kicker and plan on a runner of their own](2026-11-03-the-kicker-and-plan-on-a-runner-of-their-own.md) | Performance | Move looper-kick and looper-plan off the 5-worker autoscaled pool (tags arm64, autoscale) onto a small always-on runner tag, so dispatch never queues behind hours-long looper jobs | accepted (2026-11-03) |

## Deferred
| Suggestion | Type | Summary | Status |
|---|---|---|---|

## Rejected
| Suggestion | Type | Summary | Status |
|---|---|---|---|
| [Apply looper#25's narrowed credentials on the live pipeline](2026-11-03-apply-looper-25-s-narrowed-credentials-on-the-live-pipeline.md) | Security | Replace the looper jobs' GITLAB_TOKEN (Jody's group-Owner looper-runtime) with the read_api bot token and revoke it, move the workers off build-box, and have Jody decide fleet-seatrials' re-key | rejected (2026-11-03) |
| [Plan conflicts by file, not by directory](2026-11-03-plan-conflicts-by-file-not-by-directory.md) | Performance | The parent's plan treats a 'touches' directory as conflicting with everything under it; have the plan name files, and hold two issues back only for a shared file, contract or named semantic conflict | rejected (2026-11-03) |
| [Start a late issue capped at the job's deadline instead of holding it](2026-11-03-start-a-late-issue-capped-at-the-job-s-deadline-instead-of-holding-it.md) | Performance | When a free slot opens but an issue's limit outlasts the job's time left, start it anyway with its limit capped at the job's deadline (the #35 path); it checkpoints and resumes in the next job | rejected (2026-11-03) |
