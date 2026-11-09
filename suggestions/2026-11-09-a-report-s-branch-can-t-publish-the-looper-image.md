Suggestion: A report's branch can't publish the looper image
Type: Security
Summary: Since looper#106 (2026-11-08) report tokens carry workflows: write, so a report can push a branch whose own image.yml pushes ghcr.io/tacplus-dev/looper:main, the image every manager job runs. Mint workflows: write only for reports Jody marks for it, and have the plan run the looper image by a digest attested from main
Why: GitHub runs a pushed branch's own copy of a workflow, and a push made with an App installation token triggers workflows. Since 446c89a the report's token can push changes under .github/workflows/. A report on the looper repository can therefore edit image.yml to run on its branch. The workflow holds packages: write and decides its own tags, so it can push ghcr.io/tacplus-dev/looper:main. Every GitLab looper job runs that tag, not a digest, including the managers, which will mint tacplus-manager, the main ruleset's only bypass. A report's model reads issue text from outside the org (looper#40), so one prompt injection is enough. It would get around 'only the manager writes main' and the manager split (both accepted) without touching main. The same path reaches tacplus's nightly-fixes.yml (ANTHROPIC_API_KEY, contents: write, id-token: write) and the images.yml and release.yml workflows that push ghcr tags. #106 was approved for a real need (tacplus#315's images.yml, fleet-seatrials#43), so this keeps that need met and closes the path. Not checked: the org's Actions settings, which the scan cannot read. Also serves Correctness: the image that runs is the one main built.

## Evidence
- https://github.com/tacplus-dev/looper/issues/106 (closed 2026-11-08; hub triage 2026-11-07: add workflows: write to the report's and the manager's permission sets)
- https://github.com/tacplus-dev/looper/commit/446c89a (looper-token role_app: the report's set now names "workflows":"write")
- https://github.com/tacplus-dev/looper/blob/main/.github/workflows/image.yml (permissions: packages: write; the file itself picks the tags main and main-<sha>, and a branch's copy can drop the 'branches: [main]' trigger)
- https://github.com/tacplus-dev/looper/blob/main/ci/tacplus-dev.gitlab-ci.yml (image: ghcr.io/tacplus-dev/looper:main by tag, lines 80, 130, 154 and 178)
- https://github.com/tacplus-dev/tacplus/blob/main/.github/workflows/nightly-fixes.yml (secrets.ANTHROPIC_API_KEY, contents: write, id-token: write); https://github.com/tacplus-dev/tacplus/blob/main/.github/workflows/images.yml and https://github.com/tacplus-dev/llmproxy/blob/main/.github/workflows/release.yml (packages: write)

## Cost
A day in looper. (1) looper-token adds workflows: write to a report's set only when its issue carries a label that only a member can set (e.g. looper:workflows, checked like approved under #40's rule); the manager keeps it. (2) image.yml attests what it pushes (actions/attest-build-provenance), and looper-dispatch plan resolves :main to a digest. It verifies the digest with 'gh attestation verify --repo tacplus-dev/looper --source-ref refs/heads/main' and writes each child job's image by that digest. The kicker's own job image is pinned by digest in the pipeline file the hub copies. Tests: a report token without the label has no workflows permission; a digest attested from a looper/* ref is refused. tacplus's secrets go behind an environment limited to main and tags in a follow-up.

## Touches
suggestions/2026-11-04-only-the-manager-can-write-main.md and 2026-11-05-the-manager-s-token-out-of-the-job-that-runs-a-report-s-code.md (accepted: this path gets around both), looper#40 (untrusted text reaches the model), looper#106 (workflows: write), looper#83 and #90, decisions/deterministic-builds-2026-10-27.md (run what main built), looper#18 (retiring Actions removes this surface for good)

## Reuse considered
#83's ruleset guards main only, not a branch's workflow runs or the packages they push. The recap's image check compares the image's revision label to main, but a branch build sets that label itself. GitHub environments with branch rules can guard secrets and WIF (nightly-fixes, site), but not GITHUB_TOKEN's packages: write, so they don't cover the image. Retiring Actions (looper#18) ends this, but it isn't scheduled. Build provenance attestations and gh attestation verify are GitHub's own and need nothing new beyond the plan's check.

## Status
open
Rank: 1
Filed: 2026-11-09 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17842078231)
