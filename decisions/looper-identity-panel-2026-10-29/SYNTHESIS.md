# Looper identity: panel synthesis (2026-10-29)

Subject: `items/looper-identity-2026-10-29.md`.

Reviewers:
- GPT via codex, high effort, read-only
- copilot, its default model
- doppelganger: answering as Jody from `~/workspace/jody`, run as a Claude subagent because the local `claude` CLI's login had expired
- claude-opus (a subagent): pending; to be added

The prompt and the raw outputs are beside this file. Doppelganger's answer is quoted in the hub's session record.

**Jody accepted the panel's changes on 2026-10-29:** "this all sounds fine to me." He also confirmed the 2026-10-07 rule (point 1 below).

## Changes to the design (accepted)

1. **Doppelganger's answers are authoritative for reversible work (Jody, 2026-10-07, confirmed 2026-10-29).** A looper that asks gets
   one of two things:
   - an answer, which it acts on;
   - "blocked on the human", which it treats as a blocker, and it keeps working on whatever is unblocked.

   The quoted-or-derived label is for Jody's review and the journal. It doesn't make the answer weaker. Irreversible, outward-facing
   and spend still wait for Jody, whatever doppelganger says. Every derived answer a looper acts on is logged, one line, in the daily
   report.
2. **Peer text is never acted on, and that is enforced, not just asked for.**
   - A looper never receives another looper's text directly. Doppelganger asks for facts, then answers itself.
   - Messages are **typed** in a tacplus-defined API, with provenance as a field, so issue text can't pose as an answer.
   - The real control is containment: one identity per repository, writes only to that repository, gates, git underneath.
   - Doppelganger coordinates; it doesn't decide looper policy.
3. **Bind to GitLab's immutable `project_id`, not `project_path`,** in the WIF condition and in kernel's mapping. `project_path` is
   kept for display only.
4. **The WIF condition really checks the ref:** `project_id == N && ref == "main" && ref_protected == true`. A bare `principalSet`
   by path doesn't enforce this. Use a dedicated WIF pool with one provider for gitlab.com, so no other provider can map the same
   attribute. Give each key its own IAM binding, never at key-ring or project level, and generate them all from one repository list.
5. **A looper can't change its own controls.** Its GitLab token has the **Developer** role. Variables, protection rules and the CI
   configuration's protections belong to Jody or the hub.
6. **Separate `id_tokens` per audience:** one for GCP and one for kernel. Kernel exchanges a token once, remembers its `jti` until
   `exp`, rejects reuse, and mints tokens that last minutes. `id_token` values are masked in logs. `aud` stops cross-service replay;
   the `project_id` mapping is what authorizes.
7. **Kernel's trust of GitLab is generated code** (AAA is not left to user code): the JWKS check, `aud`, the ref rule and minting.
   The tenant config only supplies facts (which project maps to which principal). Kernel refuses any mapping to a principal outside
   that tenant's own `spiffe://<td>/tenants/<tid>/…`.
8. **Commits:** push branches with `git push` using the installation token, and merge to main through the API as a squash, so
   GitHub signs it. **Main requires signed commits,** so GitHub refuses an unverified commit.
9. **Installation tokens are minted narrow:** only the looper's own repository, only the permissions the step needs. Tokens are
   never logged.
10. **Key origin, corrected:** GitHub generates each app's private key, so it can't be born in KMS. It is imported once into KMS as
    an asymmetric signing key. The downloaded file is deleted, and the import is the only time the key exists outside Google.
11. **The kicker:** per-project GitLab pipeline schedules where they're enough, so no kicker credential exists. Where a trigger is
    needed, use one token per project, limited to main. Every pipeline ignores variables supplied by a trigger, and the trigger's
    owner is a bot, never Jody.
12. **The design names its remaining static secrets:** the GitLab project access tokens (Developer role, rotated), and
    `GHCR_PUSH_TOKEN` (looper#12, until GitHub supports app pushes to GHCR). Use the CI job token where it is enough.
13. **Retire Jody's tokens:** once every repository has moved over, delete the `GH_TOKEN` and `GITLAB_TOKEN` group variables.
    Nothing falls back to them. A failure on the app path is raised as a blocker.
14. **Mailbox:**
    - Each mailbox is bound to its registered principal, and only that principal may read it.
    - Every question doppelganger asks a looper carries a deadline and a defined outcome if no reply comes. No reply is not an
      answer.
    - The escalation path to Jody names its mechanism, and it's checked that the message actually reaches him.
15. **The shared looper image and scripts** are built and published only by the build pipeline (looper#17), from protected main,
    pinned by digest. No looper can publish them, so a looper can't push code that every other looper then runs.
16. **Check GitHub's app limit per owner** before creating about 30 apps.

## Still open
- The fourth review (claude-opus).
- How the reader app's cross-repository read access interacts with private repositories' contents (copilot: an exfiltration
  path). Mitigation: nothing secret lives in repositories (secrets live in GitLab and KMS), and a looper's outbound network is
  limited to the services it needs.
