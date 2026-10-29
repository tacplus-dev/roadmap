# Sign-in and linking: panel synthesis (2026-10-29)

Subject: `items/login-flow-design-2026-10-29.md`.

Reviewers:
- GPT via codex, high effort
- copilot, its default model
- claude-opus, as a subagent
- doppelganger, answering as Jody from `~/workspace/jody`, as a subagent

The prompt is in `PROMPT.md`, and the codex and copilot outputs are beside this file. The two subagents' outputs are in the hub's
session record.

The reviewers agree that the model is sound:
- access is a per-tenant token from a proof that tenant accepts;
- links are keyed by (IdP, subject), never by email;
- conflicting links are refused;
- personal data never falls back into a company tenant;
- OIDC only, no discovery.

What follows is where it breaks, with fixes.

## Where all four converge (treat as real)

1. **The first-sign-in binding must be one ceremony, not "whatever personal account is in the session".**
   - Kiosk or shared browsers, leftover sessions and login CSRF on the personal leg can bind a victim's company mapping to an attacker's
     XID. Then the victim's Scotty writes "personal" data into the attacker's people tenant.
   - **Fix:** both legs share one transaction id bound to the browser, in `state` and a cookie. Each leg has its own PKCE and nonce.
     The personal leg is **fresh** (`prompt=login`, `max_age=0`, `auth_time` checked). A proof made earlier in the session is
     **never** reused.
   - A **confirmation screen names both identities** ("bind jody@acme.com to j…@gmail.com?").
   - A session never mixes humans. A proof whose XID differs from the session's starts a new session.
   - Concurrent first sign-ins are serialized, so two can't both mint an XID.
2. **Kernel owns the XID value in every tenant mapping.**
   - A tenant may list, disable and delete entries for **its own IdP's subjects**, and set approval-required. It may **never set or
     change an XID**.
   - Only kernel writes the XID, and only as the result of the binding ceremony.
   - Otherwise a company admin points a subject it controls at a victim's XID: impersonation and correlation.
3. **A personal link is added only from a personal proof.**
   - A company session must not attach a new personal account to its XID. Otherwise an admin who controls the company IdP links its own
     fresh Google account to an employee's XID and takes their Scotty.
   - Rule: a personal link is added to an XID only from a session holding a fresh proof from an **existing personal link** of that XID.
   - A company-born XID (no personal link yet) never gains one. Mint a personal XID and run a proven merge instead.
4. **Org-owned accounts are never personal links. They're classified by issuer and tenant, not by email.**
   - Google with `hd`, Microsoft with `tid` other than the consumer tenant, and Managed Apple IDs are company accounts.
   - So the hub's clause that a suspended link "keeps working if the company authorized that same IdP" is wrong: it would make a
     company-controlled account the person's personal anchor.
5. **Registering a company needs proof that it controls the IdP, not just DNS.**
   - For Entra: admin consent from that `tid`. For Workspace: an admin install for that customer. For Custom: a signed discovery
     document at the verified domain.
   - **One (issuer, tid or customer) belongs to one tenant.**
   - Show the IdP's host before redirecting. Otherwise whoever controls DNS pins an IdP they run and phishes the company's staff.
   - **B2B guests** (Entra `acct`/`idp`) default to admin approval, never auto-membership.
6. **The domain refuse rule is hygiene, not the access control.** Say so. The access control is the per-tenant token from the
   authorized issuer and tid. Also:
   - **A missing verification claim means "unknown", not "personal".** Microsoft's `xms_edov` must be configured.
   - Apple relay addresses can't be enforced; document that.
   - **Subdomains** match by suffix at the registered label, checked against the Public Suffix List. Registering a public suffix is
     refused.
   - A verified email that changes at the IdP has a defined flow.
7. **Subjects must never be reassigned.** Custom IdPs declare a non-reassignable `sub`, and address-shaped subjects are refused.
   Microsoft is keyed by (`tid`, `oid`). A mapping stores `first_seen`. An admin can disable an entry, never reuse it.
8. **Merges:**
   - They need a **fresh personal proof on both sides**, never a company proof.
   - They don't silently move company memberships. A tenant gets a "re-resolve subject S" event, not an event that reveals two XIDs
     are one person.
   - Tokens minted for the merged XID are revoked.
   - **After the redirect's 30 days, a stale mapping fails closed:** sign-in is refused with a re-bind path, and **never** mints a new
     XID. A minimal tombstone (`xid → merged_into`) outlives the redirect, so a restored backup or a missed sweep can't resurrect or
     re-mint.
9. **Offboarding "within minutes" holds only if company tokens are never re-minted from a cached proof.**
   - A per-tenant **maximum proof age**, based on the IdP's `auth_time`.
   - Refreshing at the IdP must succeed.
   - An **immediate admin kick** in kernel.
10. **DNS re-verification states:**
    - A failed lookup changes nothing and raises an alert.
    - An authoritative "record gone" moves the domain to *lapsed*: the admin is told, and after a grace period (in config) it's
      unregistered.
    - **Transfer policy:** a new owner of a lapsed domain can't suspend the old owner's users without a cooling-off period.
11. **Privacy:**
    - The remembered icon is opt-in ("remember on this device"), is forgotten on sign-out, and offers a "shared device" choice.
    - The domain lookup is rate-limited and answers only "continue" or "not registered". Writing down that registered domains can be
      enumerated is a decision.
12. **Tests for every attack above:** kiosk binding, login CSRF, an admin writing a foreign XID, a company session linking a
    personal account, a recycled subject, an expired redirect, the person with a suspended only-link, and DNS failure versus a record
    that's gone.

## For Jody
- **A. Per-tenant pairwise identifiers** (claude-opus, doppelganger, codex). A tenant would see a kernel-issued identifier per
  tenant (for example HMAC(tenant, XID)), never the raw XID, so two companies can't correlate one person and a leaked XID is useless.
  It changes "the tenant maps subject → XID" to "subject → pairwise id", which kernel resolves.
- **B. A person whose only account was `@company` before registration** is locked out of their Scotty when it's suspended. The
  panel proposes a recovery ceremony:
  - during a grace window, the suspended link may reach only a restricted page;
  - there the person signs in freshly with a new, classified personal account;
  - that account is linked to the same XID, and the old link is retired;
  - they're notified in advance at the verified address.

  Doppelganger frames the question: once the company controls that domain, does a link proven only through an `@company` account
  still prove the *person*?
- **C. A permanent minimal tombstone** (point 8): the 30-day automatic follow stays (Jody's), but a few bytes of `xid →
  merged_into` stay so nothing resurrects or re-mints.
- **D. Dropping "keeps working if the company authorized that same IdP"** (point 4). That clause was the hub's wording, not Jody's;
  he said suspension "is a good idea".

## Jody's answers
- **A. Pairwise per-tenant identifiers: accepted (Jody, 2026-10-29).** The concern is **correlation, not secrecy**: an XID is not a
  secret, but as a stable identifier shared across tenants it lets two companies intersect their member lists, joins leaked data, and
  makes merge events reveal links.
  - A company tenant sees a kernel-issued identifier for each person, for example HMAC(tenant, XID). The tenant maps
    subject → that identifier, never the raw XID.
  - **Tokens follow:** a company tenant's token carries that tenant's identifier as the user. The people tenant (Scotty's home) keeps
    the real XID.
  - Only kernel maps between them.
- **B. A person whose only account was `@company`: a grace window plus warning (Jody, 2026-10-29).**
  - When the company registers, everyone with a link from that domain is told, at the verified address, to link a personal account
    before suspension takes effect.
  - Then, for a grace window (30 days by default; in the proto config), the suspended link signs in **only** to a page that links a
    freshly signed-in personal account to the same XID and retires the company link.
  - Every link added this way is announced to every address the person has, so an admin who reset the account to hijack it is seen
    and can be disputed.
  - After the window, recovery needs a human review.
- **C. No tombstone (Jody, 2026-10-29; he is "very conflicted about forever").**
  - A merged XID's redirect lives 30 days, then nothing is kept.
  - In its place, a **rule**: any tenant mapping that points at an XID kernel doesn't know (merged, aged out, anything) **fails closed**
    to "this account needs re-linking", and never mints a new XID. That covers a restored backup and a missed sweep.
  - Random 128-bit XIDs make reuse impossible, so no record is needed for that.
