# Identity graph huddle (2026-10-21, Jody + Snoopy)

Status: HUDDLE. Nothing here is decided. It records the shape we talked through so the next
huddle starts from it, not from zero. Where it conflicts with D29
(`looper/schematic/decisions/d29-idp-uncoupled-from-tenants-design-2026-10-02.md`) or the
2026-09-08 tenant-owned-services decisions, this note is the newer thinking and the records
stay authoritative until a decision supersedes them.

Companion: the identity design doc "Federated Workload Identity: Satellites and Reflectors"
(Claude Docs) and its Biscuit companion cover workload identity; this huddle is about people
and tenants.

## Shape

1. **Users exist outside tenants.** `/users/<xid>`. XID is minted by us, immutable, never reused
   (deletion tombstones it). It is the `sub` of every user token and the user's SPIFFE path.
   Nothing in it names a cluster, tenant or IdP, so it is persistent across all three.
   Supersedes the D29 Person at `/tenants/people`.
2. **Mappings make the graph.** A mapping is keyed `(issuer, immutable subject claim)`, never
   email: Google `sub`, Apple `sub` (per developer team), Microsoft `oid`, per-tenant OIDC `sub`
   under its configured issuer. A key maps to at most one XID. A login whose key is already
   bound to a different XID is refused; XIDs are never merged silently.
3. **Personal mappings** (Google, Apple, Microsoft) are permanent and are what mints the XID.
   **Personal-first is a rule:** an org login with no existing XID is bounced to a personal
   sign-in and then bound. Reason (Jody): Scotty is the person's dog; without a personal anchor,
   termination would take Scotty away.
4. **Org mappings are the membership.** Google, Apple, Microsoft org accounts, or a per-tenant
   IdP (Microsoft, Google, Okta, any OIDC). There is no separate membership record; removing
   the mapping is leaving the org. Org mappings are soft state with a lifetime that only an
   org login refreshes, so an offboarded employee ages out without SCIM or any notice from the
   org. Nobody onboards employees: the org IdP is the proof.
5. **Device keys are mappings too**, issuer = the attestation root (Apple App Attest, Google
   hardware attestation; Samsung chains to Google and is not a separate signer) or ours for
   self-asserted keys (synced keychain, desktop). Device auth extends the same concept.
6. **Tenants get a TID**, minted at onboarding, immutable, never reused. Tenant names are
   mappings to the TID, several allowed, each name bound to one TID at a time. Paths, grants,
   the memberships claim and IdP records use the TID.
7. **Services stay as they are:** `/system/services/<name>` (global) or
   `/tenants/<tid>/services/<name>` (per tenant). Services are fixed; only users move. The
   2026-09-08 chroot rule for tenant-owned services is unchanged.
8. **One token per user.** `sub` = XID plus a `tenants` claim listing the TIDs the user is a
   member of. Policy is written against that claim. There are NO per-tenant user tokens
   (supersedes "one token per tenant membership" from 2026-09-08 and the D29 exchange).
   Membership latency equals token lifetime, so tokens are short-lived.
9. **Writes name their namespace.** A user-originated write says personal (`/users/<xid>`) or
   `/tenants/<tid>`; the kernel checks the TID is in the claim. Which one is Scotty's judgment,
   not mechanism (see Scotty notes).
10. **Tenant onboarding is self-service.** The first org login from an unbound issuer creates
    the tenant and binds the issuer (Google `hd`, Microsoft `tid`, OIDC issuer) to the TID.
    Minimal fuss is the product rule.
11. **The kernel owns the graph** (XIDs, TIDs, mappings, names) and mints tokens, behind a D24
    facade that does the OIDC dance (personalauth is the sketch). The kernel already operates
    above the cluster. Kernel state is the convergent, non-durable working copy; **Stratum is
    the durable, atomic record** and the owner loads on miss. Every write that carries an
    invariant (mint XID/TID, bind a mapping key, bind an issuer, claim a name) goes to Stratum
    first because only Stratum can make the uniqueness check plus write atomic.

## Added at the end of the huddle (Jody)

12. **Device attestation is supported**, for two reasons: it proves the user is a human, and an
    attested device is an **alternative anchor for minting the XID** instead of a personal
    account. It should work across the user's devices through the shared keychain.
13. **XIDs CAN merge.** A user may end up with two XIDs (different personal accounts, a device
    first, etc.). On merge, **keep the oldest XID and delete the others**; their mappings move
    to the survivor. (Replaces item 2's "never merged silently": the invariant is still one XID
    per mapping key, but a merge is an operation, not an accident.)
14. **XIDs are secure random strings, effectively nonces.** Deleted XIDs are not kept; no
    tombstones. (Replaces item 1's tombstone.)

15. **Panel follow-up (Jody, 2026-10-21): item 8 is revised.** The panel (see
    `identity-graph-panel-2026-10-21/SYNTHESIS.md`) was unanimous that a single token carrying every
    TID lets one tenant's service act as the user in another tenant and leaks the tenant graph. Fix
    accepted: a **session token** (sub = XID, no tenant list, seen only by the kernel facade) and
    **per-tenant tokens** minted from it on demand (sub = XID, aud = the TID, short-lived, roles may
    ride in it), which is what tenant services receive. The membership list stays the kernel's record,
    not a travelling claim; live membership is checked at every mint. This is D29's exchange, so it is a
    return to a recorded decision. Jody: "as long as we have a fix I am good with it."

## Panel findings walked one at a time with Jody (2026-10-21, after the panel)

16. **Merge (accepted).** Requires fresh proof of control of every mapping being moved, personal or
    device proof only, never an org login. One atomic Stratum transaction with an audit entry. Survivor
    chosen by assurance first (personal IdP over device), then age, or by the human. A hash-only
    `old -> survivor` record is kept so references resolve and a restore can't resurrect the loser.
17. **Attested devices (accepted).** Every mapping carries an assurance level: attested device, synced
    key, personal IdP, org IdP. Attested mappings are per device; a synced-keychain key is a separate,
    lower-level mapping (that is how other devices find the XID). An attested device may mint an XID; a
    synced key alone may not. Jody: the app's attestation counts as proof of personhood for our purposes
    (a product bar, not a cryptographic one). A device-only XID dies with the device; the product says so.
18. **Self-service tenant creation (Jody overrode the panel).** First login creates and owns the tenant;
    no pending state, no DNS/admin-consent gate. Customers are not technical; the office manager or an
    intern sets it up, and when that person leaves the company calls us. Requires a platform-operator
    operation to reassign a tenant's admin. The consumer/shared-issuer deny-list stays as a cheap guard.
19. **Membership revocation (accepted).** Lifetime hangs on the IdP's `auth_time` from the last org
    login, never on our session refresh. A thin membership record beside the mapping: proved_at, optional
    per-tenant max proof age, revoked flag. Tenant admin and platform operator can kick immediately;
    per-tenant tokens are minted only against a live, unrevoked membership.
20. **XID records (Jody).** XIDs exist in kernel state AND in Stratum. The Stratum record carries a
    `last_used` stamp written lazily (at most about once a day) so year-idle XIDs are findable. What
    happens to idle ones is a later product rule.
21. **Stratum contract (accepted).** Transactions are ALREADY noted: "Transactions across maps (Jody,
    2026-10-18)" in `sonar/backend/decisions/mvp-plan-2026-09-28.md` Post-launch list (lock, several
    changes across maps, commit all or reject all; one redo entry or a commit marker so a restore never
    applies half). The identity graph adds to that note, not beside it: conditional writes with a unique
    index on mapping keys, a revision per record, and a change feed so kernels invalidate rather than
    wait for a miss. Fail closed for mint/bind/merge/revoke when Stratum is unreachable; minting for an
    already-cached identity may continue for a bounded window. The domain-level quorum question is
    answered before any of this is built.
22. **Tombstones (Jody).** Keep the deletion/merge records: cheap, and creating one requires having had
    an account, so there is no unauthenticated way to exhaust them. Backup-restore resurrection is noted
    and addressed later if ever needed. Replaces item 14's "no tombstones".
23. **Claim table (accepted).** Microsoft key = `(tid, oid)` (replaces item 2's `oid` and D29's `sub`;
    update D29). Google `sub` only; `hd`/email are for tenant binding and display. Apple `sub` plus our
    team id; never transfer the app between teams. Per-tenant OIDC: exact configured issuer plus `sub`,
    refuse address-shaped subjects, tenant attests `sub` is never reassigned.
24. **IdP records get an immutable id (Jody).** An IdP is registered once and minted an id like XID and
    TID; the issuer URL and JWKS are mutable attributes. Mapping keys are `(idp_id, subject)`, not
    `(issuer URL, subject)`. Google, Apple and Microsoft are pre-registered records with fixed ids.

## Side letters (acknowledged, not designed)

- Self-service founder: the first employee to sign in founds the tenant; shared-domain issuers
  (universities, hosting providers) become one tenant; how the first admin role is assigned.
- Businesses with no org IdP (a plumber on gmail): no issuer to bind, so membership can only
  come from an invite accepted by a personal account. Product question.
- Stratum: a domain-level Stratum above the clusters is required by this design and is not yet
  in the quorum equation. Noted in `platform/services/stratum/design.md`. Separate huddle.
- Scotty: personal vs business filing is Scotty's call per write, writes are traceable so a
  misfiled fact can be refiled, an agent may sweep for misfiles. Recorded in
  `sonar/backend/roadmap/SNOOPY-STATE.md`.

## Open for the next huddle

- The memberships claim shape (TIDs only, or TID plus role) and token lifetime.
- The per-tenant IdP record: what the tenant admin can configure after self-service creation.
- Device mappings in detail (assurance levels, synced keys).
- Migration from today's `/tenants/people`, `TenantMembership` and the per-tenant exchange.
