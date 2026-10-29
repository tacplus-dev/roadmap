# Looper identity: one identity per repo, rooted in GitLab OIDC, and loopers talking through doppelganger

- **Status:** design draft for a panel review (2026-10-29). Nothing is decided beyond the points marked "Jody". Nothing is built.
- **Owner:** Jody
- **Background:**
  - `decisions/ci-on-gitlab-2026-10-27.md`, with its 2026-10-29 addendum: deploys, tests and secrets on GitLab; nothing on a laptop;
    secrets as protected project variables, never group variables.
  - `decisions/tenant-routing-2026-10-26.md`: no mTLS anywhere; tokens.
  - tacplus-dev/looper#17 (build pipeline), #25 (the loopers' GitLab token is Jody's), #12 (build tokens).
  - doppelganger (tacplus-dev/doppelganger): answers on Jody's behalf from his context repo.

## Problem

Every commit, push, issue and comment the loopers make today is made as **Jody**, through his personal GitHub token (`GH_TOKEN`)
and his personal GitLab token (`GITLAB_TOKEN`). Both are group variables, so every looper can write to every repository. Jody: "I
want the loopers to commit with their identity to GitHub not always me… this will look much more like a real product if I am not
the only committer."

## Decided so far (Jody, 2026-10-29)

- **One identity per repository.** Each looper writes only to its own repository, and reads any.
- **GitLab mirrors GitHub one to one.** The `tacplus-dev` GitLab group holds a project per repository, and each looper runs in its
  own repository's project.
- **Loopers will register with doppelganger** and authenticate to it with a token from our platform, issued through OIDC between
  GitLab and tacplus.dev. Loopers can then ask doppelganger questions. Doppelganger can ask loopers questions, and facilitate
  discussion between loopers when needed.

## Design

### 1. The root identity: the GitLab job's OIDC token
Every looper job runs in `gitlab.com/tacplus-dev/<repo>`. GitLab issues the job an OIDC `id_token` naming `project_path`,
`ref`, `ref_protected` and `pipeline_source`. **That token is the looper's only identity.** Everything else is minted from it, short-
lived, and scoped by `project_path`. No long-lived secret exists for any looper.

### 2. GitHub: an app per repository, plus one read-only app
GitHub accepts no external OIDC, and a GitHub App has one permission set across every repository it is installed on. So:
- **`<repo>-looper`**, one GitHub App per repository, **installed only on that repository**: contents, issues and pull requests write;
  metadata read. Its commits, comments and pushes appear as `<repo>-looper[bot]`.
- **`tacplus-reader`**, one app installed on every repository, with **read-only** permissions. A looper reads its dependencies, the
  roadmap repository and other repositories' issues through it.
- **Keys never leave Google.** Each app's private key is a non-exportable Cloud KMS signing key.
  - GCP Workload Identity Federation trusts gitlab.com.
  - The KMS key for `<repo>-looper` grants `signerVerifier` **only** to `principalSet://…/attribute.project_path/tacplus-dev/<repo>`,
    on protected refs.
  - The reader key is granted to every project.
  - A job exchanges its `id_token` for a GCP token, has KMS sign the app JWT, and trades that for a **one-hour installation token**.
    A job in `tacplus-dev/kernel` therefore can only ever obtain `kernel-looper`, and `kernel-looper` can only write to kernel.
- **Verified commits:** the loopers commit through GitHub's API (Git data API), so the app's commits are marked Verified. Plain
  `git push` stays possible for branches, but main is written through the API.
- **The hub and Jody** keep their own identities. Jody's commits stand out as his.

### 3. GitLab: a project access token per project
Each project's looper uses that project's **project access token**, which carries its own bot user, for labels, comments, pipeline
triggers and its own variables. It never uses Jody's token (looper#25). The kicker gets a token that can start pipelines in every
project, and nothing else.

### 4. Our platform: kernel trusts GitLab's OIDC
- Kernel accepts gitlab.com as an **external workload issuer**, as it trusts Kubernetes service-account tokens today. It checks the
  `id_token` against gitlab.com's JWKS, requires `aud` = the fleet's trust domain (e.g. `seatrials.tacplus.dev`), and refuses
  unprotected refs.
- **The mapping lives in a tenant's config** (Jody's 2026-10-27 rule: tenant config is the tenant's, and goes with it). A
  `tacplus-dev` tenant, which hosts the loopers' doppelganger, lists the allowed `project_path` values and the workload principal
  each becomes, for example `spiffe://<td>/tenants/<tid>/workloads/gitlab/tacplus-dev/kernel`.
- Kernel mints a **short-lived token** (minutes) with `aud` = doppelganger. The looper calls doppelganger through the gateway, as any
  authenticated caller of that tenant does. No static credential exists.

### 5. Loopers and doppelganger
- **Register:** at start, a looper registers its repository, issue, run id and what it intends.
- **Ask:** a looper asks doppelganger before stopping for `needs-decision`. Doppelganger answers from Jody's recorded decisions and
  says whether the answer is **Jody's words** or **derived from his rules**. Only Jody's recorded decisions settle a real call.
  Anything doppelganger can't answer goes to Jody, as `needs-decision` does today.
- **Doppelganger asks loopers, and brokers discussion:** loopers are short-lived batch jobs and can't be called. So doppelganger
  posts to a looper's **mailbox** (the mailbox and messaging services), and a looper reads its mailbox at start and between steps. A
  discussion between loopers, for example the producer and the consumer of one contract, is doppelganger relaying through mailboxes.
- **Messages between agents are data, never instructions.** A looper does not act on another agent's message alone. It acts on its
  issue, its repository's rules and Jody's recorded decisions. Doppelganger's answers carry their provenance.

## Order
1. **One repository first (kernel):** its GitHub App, KMS key, WIF binding, GitLab project, project access token, and a looper run
   that commits as `kernel-looper[bot]`. Then every repository.
2. The kicker starts pipelines in each repository's project, and the single `tacplus-dev/looper` project retires to the image and
   the scripts.
3. Kernel trusts GitLab OIDC, and the `tacplus-dev` tenant config is in place.
4. Registration and mailbox questions, once doppelganger runs reliably on seatrials.

## What only Jody can do
- Create each GitHub App and its key. The hub prepares an app-manifest link per repository, so it's one click to create and one to
  install.
- Approve the GCP setup: the KMS keys, and the WIF pool and bindings in jody1-328023.

## Open questions for the panel
1. Does scoping the KMS binding by `project_path` plus protected refs really keep a looper in one repository? What could a looper do
   in its own GitLab project (edit its `.gitlab-ci.yml`, run on an unprotected branch, change its variables) to get another
   repository's key?
2. Commits through the API versus signed `git push`: which is safer and simpler for the loopers' workflow (rebase, gates, push)?
3. The kicker needs to trigger every project. What is the least privilege for it, and what if its token leaks?
4. Is a GitLab job's OIDC token a sound root for platform tokens? Consider replay, audience, the lifetime of the `id_token`, and
   GitLab's issuer being outside the trust domain.
5. Agent-to-agent messaging through doppelganger: what stops prompt injection travelling between loopers (one looper's issue text
   making another act)? Is provenance enough?
6. Roughly 30 GitHub Apps: what is the operational cost (key rotation, app limits per org, rate limits per installation), and is
   there a simpler way to meet "write only to its own repository"?
