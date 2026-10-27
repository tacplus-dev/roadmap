# tacplus-dev "stable and tested" gate — the move of the SeaBeam loopers

Jody, 2026-10-25: once tacplus-dev is stable and tested, the SeaBeam loopers stop and move
onto the new system, even if that delays the 2026-12-01 launch. This document is what
"stable and tested" means, so the call is made against a list and not a feeling. Each item
has a proof; an item without its proof is not done. Status is kept here.

Proof fleet: fleet-seatrials (GKE `seatrials`, https://seatrials.tacplus.dev), built from
the registry at pinned commits. "Proven" means proven there, by the command or check named.

## 1. Identity

- [ ] First login with a personal IdP (Google) mints an XID, its first link and the
      `people` membership; a token is issued. Proof: sign in at seatrials, oauth and kernel
      logs show the mint, no "no namespace" refusal.
- [ ] Session credential, Merge, Unmerge, per-tenant access tokens gated on a live proof
      (identity-session-credential-2026-10-25.md). Proof: kernel tests for every rule of the
      trust section; on seatrials an org-routed login yields the org's AT and `people`,
      a personal login yields `people` only, the step-up refusal is observed for the other
      tenant.
- [ ] Link, Unlink (refused for the signed-in proof), delete account (warned), sign out
      everywhere. Proof: kernel tests plus the webapp-login screens exercised once.
- [ ] Apple and Microsoft registrations created for tacplus-dev (none reused from seabeam)
      and a login through each proven. Microsoft's place in `people` decided.
- [x] Workload identity: pigeons minted under their dog by the coop as workload broker;
      proven 2026-10-25 (coop/COOP-STATE.md).

## 2. Platform defects fixed, not carded

- [x] letsencrypt runs under the ServiceManager (launcher no longer kills it) — 2026-10-25.
- [x] Record query plane: direct read (Get<T>Rows) and the result-set registry in a
      Hazelcast map — 2026-10-25; letsencrypt restores its certificate on restart without
      a new order (proven, one restart).
- [ ] Gateway keeps its Let's Encrypt certificate across a kernel restart (today it serves
      the cluster-CA cert until letsencrypt's hourly tick). Proof: restart kernel then the
      gateway; `openssl s_client` shows Let's Encrypt within a minute; no new order.
- [ ] Event principal: the broker stamps `spiffe_id` — done for the coop's thread
      2026-10-25; every topic type that carries a sender uses the convention.
- [ ] No pod in the fleet restarts on its own over 24 hours. Proof: `kubectl get pods`
      restart counts after a day.

## 3. The service set the loopers depend on

- [x] kernel, opa, gateway, letsencrypt, eventbus, stratum (3 nodes), oauth, kmaker,
      llmproxy, doppelganger, tenantecho, coop, pigeonhost — in seatrials.
- [ ] messaging and mailbox — not in seatrials; decide whether they are platform services
      and, if so, port and prove (a message sent, held, pulled).
- [ ] blobstore — not in seatrials; same decision.
- [ ] The runtime test harness (D27) runs against seatrials and passes its gate profile.
- [ ] Every ported service on its github.com/tacplus-dev module path (mailbox, messaging,
      letsencrypt, stratum, operator still carry gitlab.com import paths).

## 4. Reproducibility

- [ ] A fleet from scratch: `cluster/create.sh`, `tacplus fleet build`, operator build,
      every image, `deploy/apply.sh`, and the fleet comes up green with the proofs above
      passing, with no hand step. Several of 2026-10-25's fixes were found by hand; the
      rebuild is what proves none remain.
- [ ] Pins follow the rule: latest patch of a MAJOR.MINOR, never a patch pin; the tacplus
      pin moves off the registry-and-floor branch commit to a release.
- [ ] Builds run on Cloud Build (not GitLab compute); the tacplus build image is produced
      from the pinned source. (Already the case; keep it so.)

## 5. The loopers themselves

- [ ] Each looper's repo, state file and configs point at the tacplus-dev tree and the new
      fleet, not the sonar tree (loopers/STATE.md, PLAN.md ownership per
      snoopy-looper-control).
- [ ] The looper hierarchy (one looper per repo, project looper from roadmap epics) stands
      up against the new repos with local launchd, and "who merges" is decided.
- [ ] One looper runs one real task end to end on the new system before the rest move.

## When every box is checked

Stop the SeaBeam loopers, move them, and re-plan the launch date from what remains.
Until then, SeaBeam work continues on the old tree and the two are not mixed.
