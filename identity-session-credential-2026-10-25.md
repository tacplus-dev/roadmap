# Identity: session credential, per-tenant access tokens, merge/unmerge — DECIDED 2026-10-25

Huddle Jody + Snoopy, 2026-10-25 (morning after coop slice 2). Resolves the open items of
`identity-graph-huddle-2026-10-21.md` on token shape, memberships claim, lifetime, merge proofs and
migration from /tenants/people. Supersedes D29's per-tenant exchange *as the primary model*: the exchange
stays, but what it exchanges is the session credential below.

## The identity

- A human is `spiffe://<td>/users/<xid>`. The XID is minted at first login, random, immutable, never
  reused, and names no tenant, cluster or IdP. tacplus already carries the `/users/<xid>` root (v2.2.0).
- Links: `(idp_id, provider_subject) → xid`, kernel-held, Stratum durable. First login creates the XID
  and its first link. One provider identity belongs to exactly one XID.
- `people` is the default tenant: reserved, always present, cannot be deleted or left. Every XID is a
  member from first login. Membership in `people` grants managing the XID, its links, its account and
  its dog (Scotty); nothing in any org.
- Orgs are tenants. Membership in an org is proven by the org's IdP (claim matchers / tenant IdP), as
  today.

## The trust rule (Jody, verbatim in substance)

A session is a set of proofs. Each IdP assertion the user presents activates exactly the tenants that
IdP routes to. An org login activates that org AND `people` (the org vouches for the person). A personal
login activates `people` only. Joe with org A's proof has A and `people`, not B; B activates only when
B's IdP speaks; proofs accumulate (step-up), never replace. Relationships cross without a second login:
active in A, Joe acts toward B in A's capacity through the A–B relationship record, never through
membership in B.

## The tokens — three tiers, the token is the state

1. **Session credential** (held by the client, large is fine). A kernel-signed document: XID, a
   per-XID generation, and the set of proofs, each `{idp_id, provider_subject, auth_time, expiry,
   tenants vouched}`. Presented only to kernel (Mint, Merge, Unmerge, the per-tenant exchange). No
   session store: a login, renewal or rotation writes nothing on the backend.
2. **Access token (AT)**: one tenant, minutes long, `sub=/users/<xid>`, `namespace=/tenants/<tid>`.
   Minted from the credential only when it holds a live proof vouching for that tenant (or tid is
   `people`, which any proof activates); otherwise PERMISSION_DENIED with a step-up: sign in with that
   tenant's IdP. The personal token IS the `people` tenant's AT — every token is a tenant token, so the
   authorizer's single-namespace model is unchanged.
3. **Service token (ST)**: one audience, what is on the wire and what every service verifies today.
   Unchanged.

Mint checks, not lookups, except two reads: the link still exists and the membership still exists — the
only way an unlink or an org removing a member reaches a running session (at the next renewal).
Lifetimes follow the voucher: an org proof lives on the org IdP's policy; a `people` AT minted from it
shares that bound; the personal proof is the long-lived one (personal-first: termination must not take
Scotty).

## Kernel surface

- **Mint** (login): the broker's verified assertion → a credential with one proof, plus the first AT.
- **Merge(credentials…)**: all must name the same XID; union of proofs deduplicated by
  `(idp_id, subject)` keeping the fresher; re-sign. Stateless. A user's several personal IdP logins are
  proofs like any other and merge the same way.
- **Unmerge(credential, proof)**: drop a proof, re-sign. Stateless. Dropping the last proof is a
  sign-out (returns nothing). Unmerge changes only what this credential can mint; the link remains.
- **Link(credential A, credential B)**: different XIDs, both-sides-proven (both credentials presented).
  THE write: links move to the surviving XID, the other XID is tombstoned with a redirect record. Own
  warning path.
- **Unlink(credential, link)**: the write that removes an account from the XID. Refused for a link
  whose proof is in the presenting credential ("not the one you are signed in with").
- **Delete account**: warned, two-step; revokes links, bumps the generation, tombstones the Person.
- **Sign out everywhere**: bump the per-XID generation (one write); every outstanding credential fails
  at its next mint.

Accepted trade-offs of statelessness: no instant revocation (short ATs bound it), no single-use
rotation (an old copy of a credential is valid to its expiry; bearer semantics, TLS).

## Library orientation

Kernel's RPCs above are the contract. `gateway/webapp-login` grows the screens (step-up prompt, linked
accounts, delete account) over them; an upstream (SeaBeam) may replace the screens and keep the RPCs.
Gating (paying customers) is a tenant question upstream, not this layer's.

## Microsoft

Open: kernel's `people` provider set is closed to Apple and Google today. Whether a Microsoft account
can be a first login and a personal link, or stays tenant-only, is a follow-up decision. New Apple and
Microsoft registrations are to be created for tacplus-dev; nothing is reused from seabeam.

## Build order (seatrials)

1. Kernel: XID-keyed Person with links and the `people` membership at first login (replaces the
   provider-keyed row). Google first login mints and works.
2. Kernel: the session credential, proofs, Merge/Unmerge, and the gated per-tenant exchange with the
   step-up refusal.
3. Claim matchers routing Jody's accounts into alpha and beta, to prove the gate.
4. webapp-login: step-up prompt, linked accounts, delete account; Link/Unlink/Delete RPCs.
