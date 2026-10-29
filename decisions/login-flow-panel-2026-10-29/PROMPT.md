You are reviewing an identity and sign-in design for a multi-tenant platform (personal accounts and company tenants, OIDC IdPs, one durable identity "XID" per human, a personal AI agent "Scotty" per person). Be adversarial and concrete.

Output (markdown, no preamble):
## Findings
Bullets "- **[severity: blocker|major|minor] one-line claim.** Why, with a concrete attack or failure scenario, citing the design's section."
Focus: account takeover (binding a company sign-in to the wrong XID, linking, merges, recycled IdP subjects/emails, the 30-day redirect), the registered-domain refuse rule (verified email claims, multi-tenant IdPs, guests, subdomains), what a malicious company admin can do through its tenant-held mapping, lockout cases, CSRF/session issues in the multi-sign-in flow, and privacy.
## Answers to the open questions
Numbered 1-6, a recommendation each, one short paragraph.
## What is good and should stay
Short.

===== DESIGN (items/login-flow-design-2026-10-29.md) =====
# Sign-in and linking: design

- **Status:** design for a panel review (2026-10-29). The points marked **(Jody)** are his decisions, from the huddle log in
  `items/login-flow-2026-10-29.md`. The rest is the hub's proposal. Nothing is built.
- **Owner:** Jody
- **Background:**
  - `decisions/identity-session-credential-2026-10-25.md` (the session credential and its proofs)
  - `decisions/identity-graph-huddle-2026-10-21.md` and its panel (issuer squatting; merge as account takeover)
  - `decisions/tenant-deletion-2026-10-27.md` (XIDs kept and aged out after about a year)
  - `items/flock-design-2026-10-28.md` (the people store; refuse a conflicting link)
  - `decisions/tenant-routing-2026-10-26.md` (per-tenant tokens; tid)

## Model

- **One XID per human (Jody).** An XID is random, 128 bits, and never reused.
- **Access is the token, not the XID (Jody).** You get into a tenant with a token minted for that tenant, from a proof that tenant
  accepts. An XID alone opens nothing.
- **The people store (flock) holds only *personal* links (Jody):** (IdP, subject) → XID, for personal accounts.
- **Each company tenant holds its own mapping (Jody):** its IdP subject → XID, as **tenant data**. The company controls it, and it's
  deleted with the tenant.
- **One Scotty per person, in the people tenant (Jody).** It writes personal data only with the person's **people-tenant** token.
- **Everything below is configuration defined in proto and generated (Jody):** the providers offered, a tenant's authorized IdP and its
  registered domains, how people join a tenant, and the durations.

## The sign-in page
- It starts from "not signed in" and offers **Apple, Google, Microsoft, Custom (Jody)**. Each of the first three may be a personal or a
  business account. **We never ask for an email or username (Jody):** the IdP does that.
- **Custom (Jody):** the person types a company domain, and we look it up in **our records**: a tenant that has *verified* the domain,
  and the OIDC IdP it authorized.
  - **Unregistered domain:** "your company isn't registered". There's no discovery and no ad-hoc sign-in (Jody).
  - **Remembered as an icon on that device** (Jody), in the browser's local storage. It's a convenience, not trust: every use repeats
    the lookup. It has a "forget".
- **OIDC only (Jody).** SAML was discontinued on purpose.

## Registered domains
- **A domain is registered only after DNS-verified ownership,** checked again periodically.
- **The registrant must be signed in with a personal account (Jody),** which tells us who is registering. They become the tenant's first
  admin, and they connect the tenant's OIDC IdP: the issuer, the client, and for multi-tenant IdPs (Entra, Google Workspace) the exact
  tenant id or customer.
- **A registered domain MUST sign in through its authorized IdP (Jody).** The domain we enforce on comes only from the IdP's **verified**
  email claim (Google `email_verified`, Microsoft `xms_edov`); an unverified email has no domain. If the verified domain is registered and
  the proof didn't come from that tenant's authorized IdP (issuer and tenant id), the sign-in is **refused**, with a pointer to the
  company sign-in. It is never silently treated as a personal account.
- **On registration, existing links from that domain are suspended, not deleted (Jody):** "we dont want to let business and personal
  get confused." They keep working if the company authorized that same IdP. The person is told why at their next sign-in.

## The first company sign-in (Jody)
1. The person signs in through the company's authorized IdP. If the tenant's mapping has their subject, they get a company-tenant token.
2. If it doesn't (a first sign-in), the tenant **creates the mapping, and so membership, by default** (a per-tenant setting; the
   alternative is admin approval). The person is **asked to also sign in with a personal account** ("link your personal account so you
   don't get locked out"):
   - if that personal account already has an XID, the tenant maps to **that XID**;
   - if they skip it, or it has no XID yet, a **new XID** is created and mapped.
3. The session credential holds every proof made in the session. **Each tenant's token comes from a proof that tenant accepts:**
   company tokens from the company IdP, people-tenant tokens from a personal link.

## Linking and Scotty
- **Linking a personal account** adds (IdP, subject) → XID in the people store, after a **fresh** sign-in at that IdP. It is **refused**
  if that account already belongs to another XID: the flock refuse rule. Merging two XIDs is a separate, proven action.
- Once linked, we don't keep asking; any linked personal account signs in as the same XID (Jody).
- **A company-only session:** Scotty holds no people-tenant token. When it has something that belongs on the personal side, it **says it
  won't store personal things with the company**, and asks the person to sign in with a personal account, or the thing is lost (Jody).
  Personal data never falls back into a company tenant.
- **Offboarding:** the company disables the person at its IdP. No new company token can be minted, the current one runs out within
  minutes, and the person's personal side and Scotty are untouched.

## Merges and stale mappings (Jody)
- A merged XID is **not deleted at once.** It becomes a dead XID with a **MERGED redirect** to the live one.
- **A merge publishes an event**; every tenant that maps to the merged XID watches for it and updates its mapping. A **daily sweep**
  catches misses. On sign-in, kernel follows the redirect, so access works before the tenant has updated.
- **The redirect lives 30 days (Jody)**, then goes.

## Open questions for the panel
1. **Binding the first company sign-in to an XID:** is "the personal account signed in during the same session" strong enough? Could an
   attacker get a victim's company mapping bound to the attacker's XID, or the reverse: a shared or kiosk browser, a session left signed
   in, a forced login (CSRF on the account chooser)?
2. **The refuse rule:** does verified-email-domain plus issuer/tenant-id matching close the hole, given Google accounts and Entra B2B
   guests with `@company` addresses, email changes at the IdP, IdPs that assert unverified emails, and subdomains?
3. **Takeover through linking and merges:** with links refused on conflict and merges a separate proven action, what paths to taking over
   an XID remain? Look especially at the company admin who controls the IdP, an account recycled at the IdP (a reassigned email or
   subject), and the 30-day redirect.
4. **Suspending existing links on registration:** what goes wrong for someone who used an `@company` account as their only account
   before registration? Are they locked out of their Scotty?
5. **The mapping as tenant data:** the company controls its subject → XID mapping. Can a malicious company admin point a mapping at
   *another* person's XID (to impersonate them in its tenant, or to learn about them)? What must kernel enforce?
6. **Privacy:** the remembered icon on shared devices; what the domain lookup reveals (which domains are registered); and what a tenant
   learns about a person's XID and other memberships.

===== EARLIER PANEL (identity graph, 2026-10-21) SYNTHESIS =====
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
