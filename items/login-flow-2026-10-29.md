# The sign-in flow

- **Status:** huddle in progress (2026-10-29). Changes nothing yet.
- **Owner:** Jody
- **Background:**
  - `decisions/identity-session-credential-2026-10-25.md`
  - `decisions/identity-graph-huddle-2026-10-21.md`, and the 10-21 panel: issuer squatting, merge as takeover
  - `items/flock-design-2026-10-28.md`: refuse a link that conflicts on a local write

## Jody, 2026-10-29
- **Start from "not signed in".** The page offers **Apple, Google, Microsoft, Custom**. Apple, Google and Microsoft may be a business or
  a personal account. **We never ask for an email or login:** the IdP does that.
- **Custom:** we ask for the **company domain** and find its IdP. The browser remembers it, so next time it's a quick icon.
- **A domain registered with us MUST sign in through its authorized IdP.**
- **Once signed in, we encourage linking personal accounts,** so the person isn't locked out of their Scotty if their employment ends.
- **An unregistered domain under Custom:** "your company isn't registered". There's no generic discovery and no ad-hoc sign-in.
- **Registering a company requires being signed in with a personal account.** We want to know who is registering it.

## How it maps (hub's notes, not decided)
- **Finding the IdP from a domain** is a lookup in our records: a tenant that verified the domain, and the IdP it authorized. It isn't
  discovered from the domain.
- **The remembered icon** lives in the browser's local storage, on that device only. It's a convenience, not trust: it goes through
  the same lookup and IdP every time. It has a "forget".
- **The domain we enforce on** comes only from the IdP's **verified** email claim (Microsoft `xms_edov`, Google `email_verified`).
  - If the verified domain is registered and the sign-in didn't come from that tenant's authorized IdP (issuer or tenant id), it's
    **refused**, with a pointer to the company sign-in. It never becomes a personal account silently.
  - An unverified email gives no domain.
- **A domain counts as registered only after DNS-verified ownership** (no squatting, per the 10-21 panel).
- **Linking a personal account** adds a link to the same XID, with a fresh sign-in at the personal IdP. It's refused if that account
  already belongs to another XID (the flock refuse rule). When employment ends, the company removes its membership; the XID and the
  personal tenant carry on.
- **The registrant** is a signed-in XID with at least one personal link. It verifies the domain in DNS, connects the IdP, and becomes the
  tenant's first admin.
