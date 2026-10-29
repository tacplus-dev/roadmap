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
