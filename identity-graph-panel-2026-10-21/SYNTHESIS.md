# Identity graph huddle: panel synthesis (2026-10-21)

Subject: `roadmap/identity-graph-huddle-2026-10-21.md` (huddle, nothing decided).
Reviewers: claude-opus-5 (claude CLI), gpt-5.6-terra high (codex), claude-sonnet-5.5 (copilot default;
grok and gemini were not available through copilot today). Plus doppelganger answering as Jody from the
context repo. Raw outputs are beside this file.

## Where all three reviewers converge (treat as real)

1. **Merge (item 13) is the account-takeover path as written.** No authorization rule, no proof
   requirement, not atomic. "Keep oldest" is attacker-selectable (pre-mint cheap device XIDs and wait).
   Required: fresh proof of control of EVERY mapping being moved, never an org-IdP proof alone, a single
   Stratum transaction, an audit entry, and a survivor chosen by assurance (or by the human) rather than
   age. Doppelganger adds: deleting XIDs with no restore path is irreversible by the repo's own definition,
   so it is Jody's decision, not an agent's.
2. **Item 12 is internally contradictory.** App Attest and Android hardware attestation keys are
   non-exportable and per device; a shared-keychain key is by construction self-asserted (item 5 already
   says so). Attestation proves a genuine app on genuine hardware, not a human; device farms produce
   unlimited genuine attestations. Fix: an explicit assurance level per mapping (attested device,
   synced key, personal IdP, org IdP), and XID minting only from named levels. Doppelganger: Jody would
   push back on minting an XID from a key the party generated alone.
3. **The single token with a `tenants` claim (item 8) reopens the isolation that 2026-09-08 §9c and the
   login model §3 closed.** Any tenant service that holds the token can act in the user's other tenants,
   and sees the user's tenant graph. All three recommend: keep ONE subject credential (sub = XID), but
   mint short-lived audience-scoped per-tenant tokens from it by exchange for every tenant-directed
   call. The `tenants` claim may remain as a routing hint, never as authority. This is the biggest
   departure from what Jody said in the huddle and should be re-huddled with the reasoning above.
4. **Self-service tenant creation from an org IdP (item 10) hands a whole issuer to whoever logs in
   first.** Google `hd` follows domain ownership; Microsoft `tid` covers B2B guests; consumer tenants and
   shared domains are one issuer. Minimal fuss can stay: create the tenant immediately but PENDING, with
   no cross-user membership or admin until domain control (DNS TXT) or IdP admin consent is proven;
   deny-list consumer issuers. Doppelganger: "reachability implies authorization" is the fallacy Jody
   names himself.
5. **Soft-state membership (item 4) has no revocation lever.** An offboarded or terminated admin keeps
   access for the full lifetime. Keep "org login is the proof", but keep a thin membership record with
   proved_at, a per-tenant max proof age, an immediate admin kick, and lifetime hung on the IdP's
   `auth_time` (not on our own session refresh, or it never ages). Roles need a home; "TIDs only" has
   none.
6. **Stratum's contract is undesigned and the whole design rests on it (item 11).** "Uniqueness check
   plus write" is a single-key CAS; merge is multi-key. A stale kernel hit can mint for a merged or
   revoked identity; a miss treated as absence mints a duplicate. Needed: linearizable multi-key
   transactions, unique secondary index on mapping keys, per-record revision, change watch, defined
   fail-closed behaviour for mint/merge/bind/revoke when Stratum is unreachable, and the quorum question
   answered first.
7. **No tombstones (item 14) breaks referential integrity, not just reuse.** Live tokens, grants, SPIFFE
   paths and audit rows reference the deleted XID; a backup restore resurrects it. Keep a minimal
   pseudonymous record (`xid → merged_into | deleted_at`, hash only) even after personal data is shredded.
8. **The immutable-claim table (item 2) is wrong for Microsoft and loose for generic OIDC.** Microsoft
   key = (tid, oid), and B2B guests legitimately yield one XID per tenant; D29 chose `sub`, so say which.
   Apple `sub` changes on developer-team transfer. Generic OIDC must be an exact preconfigured issuer with
   a declared non-reassignable `sub`; refuse address-shaped subjects.

## Raised by one or two reviewers (worth a look)

- An org-owned account (Workspace with `hd`, Entra, Managed Apple ID) must never become the permanent
  personal anchor, or termination takes Scotty away after all. Needs an account classifier at login.
- The personal-first bounce (item 3) is open to login CSRF unless both legs share one nonce and session.
- Item 9 (client names the namespace) vs 2026-09-08 #2 (tenant only from the SPIFFE ID): safe only if the
  kernel intersects the requested TID with live membership and the service's own tenant.
- Tenant name release needs a cooling-off period, reserved names, unicode normalization.
- The no-IdP business is the modal SMB customer; invite-based membership is a second membership kind,
  which contradicts "the mapping IS the membership". The login model §5 already had customer promotion.
- Missing chapters: audit log for every graph mutation, rate limits on XID/TID/name creation and merge,
  erasure/export under privacy law, device-loss recovery, last-admin protection, `tenants` claim size.

## Doppelganger, answering as Jody (confidence as it gave it)

- Memberships claim: TIDs only; roles are kernel-attested facts policy reads (55%). Lifetime short and
  bounded; no number in the repo (30%).
- Per-tenant IdP record: the tenant admin may tighten, never loosen (70%).
- Device mappings: explicit assurance level on every one; self-asserted keys cannot anchor alone (60%).
- Migration: incremental, reversible, canary on the hand-written kernel code, no deprecation window
  (70%).
- Missing per Jody's own rules: live-token revocation, Stratum failure behaviour (quorum), a concurrent-
  login uniqueness test and a merge test, an audit trail, a recovery path if the personal anchor is lost,
  and a status line saying what is decided vs huddle-only.

## What every reviewer called sound (do not reopen)

XID outside tenants, keyed by (issuer, immutable subject) and never email; one key to one XID with a
conflicting login refused; TID with names as rebindable mappings; services fixed and the chroot rule
untouched; Stratum-first for invariant-bearing writes with the kernel as a convergent cache; the
personal-first rule and its Scotty reason; writes name their namespace and the kernel checks it; org
login as membership proof rather than a second identity.
