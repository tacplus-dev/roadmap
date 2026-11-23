# Security gates block on what ships; pipeline tooling is fixed nightly (2026-11-23)

**Status:** decided (Jody, 2026-11-23: "yes go with your recommendations"). Builds on Jody 2026-11-20, "we dont want to let
security slow us down at all right now", and supersedes the parts of the security-floor decisions that make a finding in
CI-only tooling fail a land.

## Why
In the 7 days to 2026-11-23, 121 of 357 closed issues (34%) were security findings or their fixes, and 71 of 266 open
issues (27%). Almost none are flaws in our code: they are new upstream advisories against tools and base images that
run only in our pipelines (Debian packages in the build image, `lego`, `osv-scanner`, OpenTofu, tflint, buildx). Each
new advisory turns every gate that scans that tool red at once, and unrelated work stops until someone bumps the tool
or writes an exception. On 2026-11-23 that was about half of seatrials' critical-path delay (Debian krb5 overnight:
cloud#97, cluster#48, fleet-seatrials#64; lego/osv-scanner: fleet-seatrials#71; and cluster#54, a side effect).

## Decision
1. **A land fails only on findings in what ships:** the repository's own code, the dependencies it ships, and the images
   it publishes or deploys (by digest). Findings in CI-only tooling (the build image, pipeline tool binaries such as
   tofu, tflint, lego, osv-scanner, gitleaks, buildx, kubectl in CI) are **reported, never blocking**. The scanners still
   run on them; the report keeps them visible.
2. **Build images are rebuilt and re-pinned automatically, nightly:** tacplus's Images workflow rebuilds each supported
   line's build image every night from current packages, publishing an immutable dated tag; a nightly job re-pins every
   repository's CI image to its line's newest dated tag through the merge train (one land a repository). Repositories
   pin immutable tags only (dated or patch), never a floating one.
3. **One nightly job fixes tool advisories:** for each repository, it bumps a tool pin where a fixed upstream release
   exists, else adds a 14-day scoped exception naming the advisory and the release it waits on; one land a repository.
   Unfixable shipped-code findings still go to an issue as today.

## Not changed
- Findings in shipped code, dependencies and images still block, and still lead every report (security first).
- Secrets scanning (gitleaks on the repository's own history and tree) still blocks.
- PRODUCTION.md's security bar is unchanged.
