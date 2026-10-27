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
