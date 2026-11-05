# Every secret and key is created and rotated by a pipeline, never at a desktop (Jody, 2026-11-05)

Jody, 2026-11-05: "later I want to revisit all the places we have secrets and keys. If they are necessary they will
need to be managed by a pipeline. And this includes rotation. It cant be me at my desktop."

## Decided
- **First, is it necessary?** Prefer no secret at all: Workload Identity Federation, KMS-held keys that sign but are
  never read, short-lived tokens minted per job.
- **What remains is pipeline-managed end to end:** created, stored (Secret Manager by version, or KMS), granted, used
  and **rotated** by a pipeline on a schedule, with the old version disabled, then destroyed, by the same pipeline.
- **No person at a desktop** in the steady state: no pasting, no downloading, no one-time manual rotation. A bootstrap
  that cannot avoid a person is named as such, done once, and followed by the pipeline taking over.

## To do (later, Jody's word: "later")
An inventory and review of every secret and key across tacplus-dev and seabeam.net: what it is, whether it is
necessary, who creates and rotates it, and the gap to "by a pipeline". Known so far:
- GitHub App private keys (tacplus-reader, tacplus-manager, tacplus-looper): KMS-held. **GitHub offers no API to
  create an App private key**, so their creation and rotation is a person's step on github.com today; the review must
  settle that (fewer Apps, or a documented exception, or a different mechanism).
- GitLab bot token (`tacplus-cloud-bot`, `self_rotate`) and the looper bot's token: rotation by pipeline possible.
- The in-cluster runners' authentication tokens: re-registration by API possible.
- The Claude model token, the build trigger token, fleet secrets (`SEATRIALS_K8S_SECRET_*`), certificates
  (pipeline-owned since decisions/certificates-by-pipeline-2026-11-03.md).

## Addendum: GitHub stays the source of truth (Jody, 2026-11-05)
The hub explained why GitHub Apps exist (the loopers act on GitHub's code and issues; the Apps replace Jody's personal
token with scoped identities) and offered GitLab-as-source with a GitHub push mirror. Jody: "no I dont want a mirror".
GitHub keeps tacplus-dev's code and issues; the three Apps stay, and the review settles their key rotation.
