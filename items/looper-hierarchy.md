# Looper hierarchy: one looper per repo, one project looper working from epics

- **Status:** active (design agreed 2026-10-22, build not started)
- **Owner:** Jody (design), Snoopy (build)
- **Opened:** 2026-10-22
- **Gets more expensive while held:** every day the per-repo loopers run under the old engine
  (`~/workspace/loopers/_engine`, moonshot) the two systems diverge.
- **Records:** `tacplus/scripts/fix-looper/run.sh` (the working prototype of a repo looper);
  `~/workspace/loopers/STATE.md` (the engine being replaced); `tacplus/CONTRIBUTING.md` (intake and
  triage rules the repo looper enforces)

## What (Jody, 2026-10-22)

- **Each repo has its own dedicated looper.** It checks its own issues and processes them if any.
  Exactly one instance per repo.
- **A top-level project looper is in charge and works from epics.** It writes what needs to be done
  down to the repo as issues, calls that repo's looper, and that looper works until the issue is
  complete. The project looper also knows about tacplus-dev (GitHub).
- **Epics are the roadmap repo's issues** (`seabeam/roadmap`). Free tier, so GitLab Epics are not
  available; one roadmap issue per item is the epic, and the roadmap stays the source of truth for
  what gets dispatched.

## Design

**Repo looper** = the tacplus fix looper generalized and ported from `gh` to `glab` (the GitHub
repo keeps `gh`):
- single instance: lock with pid check; a stale lock from a dead run is removed;
- trigger: a 24 h schedule plus a notify (touch file now; pipeline trigger later);
- picks the oldest open request not already in progress / waiting (labels: `fix-request`,
  `feature-request`, `feature:approved`, `nightly:in-progress`, `nightly:pr-open`, `needs-repro`,
  `needs-decision`);
- triage first: bug vs feature; unapproved features get a review comment and `needs-decision`;
- fresh clone, branch `fix/issue-N`, gates, MR against main, backports per supported release line
  where the repo has them;
- reports on the issue when it stops without an MR; never pushes to main or a release branch.

**Project looper** (new, small):
- reads open epics in `seabeam/roadmap` (issues labelled `epic`), each carrying an ordered plan of
  steps, one step = one repo + one issue body;
- for the current step: opens the issue in that repo with a back-link to the epic, notifies that
  repo's looper, then waits for the issue to close with a merged MR (or a `needs-decision` /
  `needs-repro` that it escalates to Jody on the epic);
- then writes the next step; cross-repo ordering (tacplus change, then schematic regeneration) lives
  only here;
- knows tacplus-dev (GitHub) through `gh`, every GitLab repo through `glab`;
- single instance, same lock pattern; local launchd now, GitLab runner later when a model credential
  exists (Jody has held that).

## Runtime (Jody, 2026-10-22)

- **GitLab runner, our own runner (the Tender), not shared runners.** The model credential is Jody's
  subscription token from `claude setup-token`, held as a masked protected CI variable
  `CLAUDE_CODE_OAUTH_TOKEN`; usage shares his plan's rate limits; moves to an API key with its own
  budget when looper volume competes with his interactive use (one env var).
- **Looper image** = `FROM ghcr.io/tacplus-dev/tacplus/build:<minor>` + claude CLI (npm, pinned) +
  `glab` + the looper script, pushed to the seabeam GitLab registry. The public build image stays
  free of agent tooling. Home: `seabeam/tools/looper` (beside sideband, pipewatch).
- **Looper identity:** the job's GitLab CI OIDC `id_token` (aud = our fleet) is exchanged at the
  kernel's cloud-auth facade (`AZP_CLOUD_AUTH`, 10-14 plan) for a platform token. SPIFFE path from
  the immutable claims, e.g. `spiffe://<fleet>/system/workloads/gitlab/<namespace_id>/<project_id>`,
  carrying `ref_protected` and `pipeline_source`; policy at each service decides, default deny
  (only our group's namespace ids, only protected refs). Identity is for our services (doppelganger,
  llmproxy); the Anthropic credential stays separate until WIF (D22) lets the platform identity
  stand in for it.

## Identity: what exists today (surveyed 2026-10-22)

- **Human OIDC sign-in is live**: the oauth service brokers Google, Apple, Microsoft (issuer-template
  mode) and Keycloak; the kernel relays the provider claims (`origin_iss`, `groups`, `email`) to OPA
  and mints from the answer.
- **Workload OIDC is design-only**: `MintServiceTokenRequest.external_oidc_token` (tag 9, decided
  2026-09-20, `looper/schematic/decisions/token-mgmt-item3-cloud-proof-oidc-2026-09-20.md`) exists in
  the proto, and the kernel has NO arm for it. The Kubernetes projected-token arm is a seam
  (`KubernetesTokenVerifier`: TokenReview and local-JWKS implementations, `Unwired` default). The
  GitLab id_token exchange IS the external_oidc arm: verify against gitlab.com's JWKS (the kernel's
  `LocalJwkSource`/remote JWK code is reusable), exact issuer and audience, identity from the
  immutable claims. Size M.
- **Entitlements are an OPA policy today**: the kernel calls the `entitlements` package with
  `input.claims`; it returns `namespace`, `token_entitlements.grants` (generated grant symbols such as
  `DOCMANAGER_ANY`, `KERNEL_ANY`), `data_grants` and `principal_scope`, and the kernel stamps that
  into the token's `TokenEntitlements` claim, which the generated interceptors enforce. Roles come
  from `idp/*.rego` (trusted issuer + `groups`, or per-person stopgaps for Google/Apple) and
  `grants/static.rego` (`sonar/backend/bootstrap/services/opa/`).
- **So for loopers:** one new idp rule (`origin_iss == https://gitlab.com`, `namespace_id` in our
  group, `ref_protected`) yielding a `looper` role, one grants rule mapping it to the services a looper
  may call (doppelganger, the LLM path, the connectors), and the kernel copying the selected GitLab
  claims onto the token (it already copies provider claims for people) so doppelganger can decide on
  them. The policy is the mapping; no code decides who a looper is.

## Build state and cutover plan (2026-10-22)

**Built:** `seabeam/tools/looper` (image `registry.gitlab.com/seabeam/tools/looper:main`, built in CI,
smoke-tested: claude CLI 2.1.308, glab 1.124.0, gh, just, tacplus 2.1.0), `repo-looper` (the tacplus
script generalized over `gitlab:`/`github:`), the `looper-run` CI job (one instance per repo via
`resource_group`, requests a fleet-audience OIDC id_token, 5 h ceiling). Verified end to end on the
GitHub forge path (an unintended live run on tacplus-dev #26 posted a feature review; no code).

**Not built:** the project looper; the kernel's `external_oidc_token` arm (2-3 days if the GitLab
workload fits the existing `workloads` path shape); doppelganger as a fleet service; the closed
schematic tacplus in the image; the Tender runner registered on the group; a `2.0` minor tag on the
public build image (one-line change in tacplus `images.yml`).

**Cutover, in order:**
1. **tacplus-dev fix looper to CI now.** It never consults doppelganger. Jody sets three masked,
   protected group CI variables (`CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`, `GITLAB_TOKEN`
   with api + write_repository, `GH_TOKEN` with repo + workflow) and one 24 h schedule
   (`REPO=tacplus-dev/tacplus`, `FORGE=github`). Snoopy uninstalls the launchd looper after the
   first green scheduled run. Jody 2026-10-22: "ok." **DONE 2026-10-22:** variables set by Jody
   (Snoopy drove Chrome), schedule `looper: tacplus-dev/tacplus (24h)` at 09:17 America/Denver on
   `seabeam/tools/looper`; targets live as a versioned `parallel: matrix` in `.gitlab-ci.yml` because the
   group forbids pipeline variables (that policy stays). First scheduled run (pipeline 2978388704, 2 h 5 m)
   took tacplus #90 and opened PR #116 plus backports #118 (release/2.0) and #119 (release/2.1). The
   launchd looper on Jody's Mac is uninstalled. Adding a repo = one matrix row.
2. **Kernel `external_oidc_token` arm** for gitlab.com id_tokens + the OPA `looper` role and grants
   + the kernel copying the selected GitLab claims onto the token.
3. **Doppelganger as a fleet service** (see its item), callable by loopers with that identity.
4. **Closed tacplus into the looper image; Tender runner on the group.**
5. **Project looper** working from `seabeam/roadmap` issues labelled `epic`.
6. **Schematic and SeaBeam loopers to CI**, retiring the old engine and moonshot.

**Jody's walk-through items (one at a time, with Snoopy driving Chrome):** `claude setup-token`; the
three group variables; the tacplus schedule; registering the Tender runner on the group; later, the
OPA grants for the `looper` role and the doppelganger disclosure table.

## Blocked (2026-10-25)

The GitLab free-tier compute quota is exhausted (`ci_quota_exceeded`): the 10-24 CI rewrite runs one
2-hour job per waiting issue in parallel (max 6), which burned the month in two days. Every scheduled
run since has failed at the plan job. Unblock = **register the Tender runner on the seabeam group**
(Jody) and pin the looper jobs to it, or wait for the Nov 1 reset with parallelism capped at 1-2.
Stale `nightly:in-progress` labels from the dead jobs were released on 94, 114, 128, 135, 139.

## Open

- ~~Who merges~~ **DECIDED (Jody 2026-10-25): pre-production, no merge gate.** Snoopy (or the looper)
  merges feature PRs once gates are green; no human review step. Features land on main only (next
  MINOR), never on a `release/MAJOR.MINOR` line; patch releases carry fixes only (docs/VERSIONING.md).
  Before production the release lines become protected branches and the gate returns there.
- Completion signal: issue closed + MR merged is the proposed definition; alternatives are a label
  the looper sets.
- What the epic body looks like (ordered steps, acceptance per step).
- Migration from the current engine and moonshot: which loopers move first.

## Pin-rule follow-up (2026-10-22)

The looper image's `FROM ghcr.io/tacplus-dev/tacplus/build:20261021-ecddd39` is a patch pin, which
Jody's rule forbids (always the latest patch of a MAJOR.MINOR under canary, never a patch pin).
Fixed upstream: tacplus `images.yml` now tags the build image `X.Y.Z` / `X.Y` / `X` on every release
(main aa8140b, backported to release/2.0 #121 and release/2.1 #120). DONE 2026-10-23: v2.0.9 and v2.1.2
published `build:2.0` / `build:2.1`; the looper Dockerfile is on `build:2.1` (tools/looper 0dd5c64, image
rebuilt green). The looper's own image follows the rule.

## Moved to tacplus-dev, reshaped (Jody, 2026-10-26)

The looper is now **github.com/tacplus-dev/looper** (from `seabeam/tools/looper` 6879339; image
`ghcr.io/tacplus-dev/looper`, built by GitHub Actions on arm64). Decided in the huddle:

- **A dispatcher, no work of its own** (`looper-dispatch`), on the build host (this Mac under colima now, a
  small cloud VM or container later): every three minutes it starts a repository's container when that
  repository has a ready issue and none is running; nightly it starts each repository's scan, which only files
  issues. No looper runs when there is no ready issue.
- **One container per repository**, the repository's one looper: one checkout kept across issues, drains its
  ready issues oldest first and checks for new ones, may run sub-agents in the same checkout; only it commits
  and pushes, **always to main** (pre-production; a protected release line with patch-only review comes later,
  human or panel undecided).
- **`approved` is the gate.** Only someone with write access can label, so an issue nobody approved is never
  worked; the issues we file are labelled `approved` when filed. `looper:in-progress`, `looper:stuck`,
  `needs-decision` hold an issue.
- **Order across repositories**: `Blocked-by: tacplus-dev/<repo>#N` lines in the issue body.
- **No global cap for now** (fail on the side of getting things done; Fable runs out during the week, Opus does
  not, so loopers default to Opus). Later a repository asks Doppelganger for more resources, and new
  repositories are Doppelganger's to authorize; until then the hub may create them.
- **The image has no tacplus binary**: `tacplus` runs `ghcr.io/tacplus-dev/tacplus/tacplus:<MAJOR.MINOR>` for the
  repository's floor, always that line's latest patch, as a sibling container on the shared host socket.
  Therefore every repository declares its floor, and the unreleased compiler everything was built with
  (`326ef93`, registry-and-floor) is released as 2.4 first (tacplus-dev/tacplus#161).
- **Deploys**: their own huddle; until then loopers do not deploy and the hub does.

Superseded here: the GitLab CI planner (`looper-plan`, one job per issue) and the old labels (`fix-request`,
`feature-request`, `feature:approved`, `nightly:*`). The `looper: tacplus-dev/tacplus (24h)` GitLab schedule
was disabled (not deleted) on 2026-10-26 (schedule 4507454, `active: false`): it acted on the old labels.
