# Everything runs from GitLab; GitHub only hosts code (DECIDED 2026-10-27)

Jody, 2026-10-27 ("I feel that everything runs from gitlab").

- **GitHub hosts the code and the issues, nothing else.** No GitHub Actions workflows: none for CI, builds, releases, images,
  nightlies or planning checks.
- **All automation runs in the tacplus-dev GitLab group:** the loopers and scans (gitlab.com/tacplus-dev/looper) and **every build**
  through the build pipeline (decisions/deterministic-builds-2026-10-27.md): each repo's gates, tacplus's own CI, the
  `tacplus/build:X.Y` images, the looper image, service releases, nightlies, and the four-planning-files check (now a gate).
- **Credentials live in the tacplus-dev GitLab group,** and builds get only what they need: a **read-only** GitHub token to fetch
  source (and to post commit statuses), and a registry token that can push images. Loopers keep their own read-write token.
- Existing GitHub Actions workflows (tacplus, looper, nsproxy, llmproxy, website, and `planning-files.yml` everywhere) are retired once
  their GitLab replacement runs. The docs site may stay on GitHub Pages, published by a GitLab job.

## Addendum 2026-10-29: deploys, tests and secrets too (Jody)
"we should do deployments including test from gitlab. the secrets should be held by gitlab. there must be nothing on this mac."
- **Deploys and post-deploy tests run as GitLab pipelines,** one project per fleet (gitlab.com/tacplus-dev/fleet-seatrials first).
  Applying is a manual, protected job. The kubectl diff fails if it touches letsencrypt.
- **Secrets are held by GitLab,** as protected project variables of the deploy project, never group variables (those reach the
  loopers' agents). GCP access comes through GitLab OIDC and Workload Identity Federation, with no key files. seatrials' fleet key
  material is already there (FLEET_CRYPTO_YAML, FLEET_CA_PEM).
- **Nothing on a laptop:** no checkouts, no secrets. The kicker moves to a GitLab schedule too (tacplus-dev/looper#24).
- Until the deploy pipeline exists (tacplus-dev/fleet-seatrials#17), nothing is deployed from a laptop.

## Addendum, 2026-11-07: release images from a tag-triggered GitHub Actions workflow, for now

Jody (2026-11-07, "do what you think best, get us to deployment"; "I have approved everything to get the deployment up and running"): a service's release image is built and published by a `release.yml` GitHub Actions workflow on its `vX.Y.Z` tag, the way tacplus-dev/llmproxy already does: gates on the tag, both architectures, `ghcr.io/tacplus-dev/<service>:X.Y.Z` and the line tag `X.Y`, pushed with the workflow's own `GITHUB_TOKEN` (no stored secret, no personal token), and the GitHub release created. Packages are public, matching the open-source repositories, so fleets and clusters pull anonymously.

- Why: tacplus 2.6's `fleet build` locks each release to its published image, and seatrials' first deploy waits on it. The GitLab build pipeline (tacplus-dev/looper#17) needs a ghcr push credential that GitHub offers only as a classic personal token.
- This departs from this decision's "no GitHub Actions" for releases. looper#17 can take releases over later. The deterministic build stays the goal.
