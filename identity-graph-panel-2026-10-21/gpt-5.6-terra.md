## Findings

- **[severity: blocker] XID merge is an account-takeover primitive unless both identities are freshly and independently proven.** “Keep oldest” is not an ownership rule: an attacker who controls an old XID can gain the victim’s mappings, memberships, data, and active device authority through a mistaken or coerced merge. Require high-assurance reauthentication of each source, an explicit confirmation on both accounts, session/token revocation, an immutable merge audit trail, and an atomic transaction; huddle.md #13 does not specify any of these.

- **[severity: major] Device attestation neither proves a human nor survives a shared-keychain design as a hardware-attested key.** App/device attestation establishes a genuine app/key on a device, not the person operating it; a synchronizable key copied through a cloud keychain is a shared credential and must not retain a “hardware-bound attested” assurance level. Treat attested-device enrollment as a separately scoped authenticator with recovery and abuse controls, not as an equivalent personal anchor (huddle.md #5, #12; login-model-2026-09-08.md §2a).

- **[severity: blocker] First-login self-service tenant creation lets an ordinary employee squat or take administrative control of an organization’s issuer.** `hd`, `tid`, and an OIDC issuer identify an organization context, not an authorized tenant founder; shared issuers make the problem explicit. Require verified domain/IdP-admin proof before exclusive issuer binding and first-admin assignment; otherwise an unbound issuer must create an unclaimed pending tenant only (huddle.md #10 and side letter).

- **[severity: major] A single bearer token containing every TID weakens tenant isolation and makes authorization, roles, revocation, and token size harder.** It exposes a user’s tenant graph to every recipient and grants a compromised token authority across all listed tenants; TIDs alone also cannot express tenant roles. D29’s per-tenant exchange keeps raw provider claims out, binds a token to a live membership, and carries bounded mapped roles; abandoning that needs an equally strong selected-tenant/audience and authorization model (huddle.md #8; d29.md §2 I1–I3, §3.2.4).

- **[severity: major] “Short-lived tokens” leaves revocation replay valid for the entire token lifetime and does not address cached authorization.** An offboarded user retains all listed memberships until expiry, while soft-state membership may already have been stale for its refresh period. Define a maximum access window, immediate platform-admin revocation, session/device revocation, and either token introspection or a tenant-membership epoch checked at every sensitive request (huddle.md #4, #8; d29.md §6 R3).

- **[severity: blocker] Stratum-first writes are insufficient unless Stratum is globally serializable and stale kernel replicas are forbidden from making invariant or authorization decisions.** “Load on miss” does nothing for a stale hit: two kernels could bind one mapping to different XIDs, honor a deleted membership, or mint a token after a merge/revocation. Stratum must provide linearizable conditional writes and ordered change versions; kernels must apply them durably or fail closed for minting, merge, issuer binding, and revocation during uncertainty (huddle.md #11).

- **[severity: major] The mapping-key rules need provider-specific normalization and lifecycle contracts.** Google should use exact trusted `iss` plus `sub`, never email or `hd`; Apple `sub` is scoped to the developer team/client grouping, so that scope must be retained; Microsoft `oid` is only stable within an Entra tenant/object lifecycle and must be paired with a tenant-specific issuer/`tid`—D29 instead selects platform Microsoft `sub`. A generic OIDC mapping is safe only for an exact preconfigured issuer and an IdP contract that `sub` is non-reassignable; no arbitrary “immutable claim” should be accepted (huddle.md #2, #10; d29.md §3.1, §7.4).

- **[severity: major] Deletion conflicts with the stated non-reuse invariant and lacks a privacy, recovery, and forensic model.** huddle.md #1 requires deletion tombstones, while #14 removes them; secure randomness makes collision unlikely but cannot make “never reused” true after all history is erased. Define separable data erasure, minimal non-identifying anti-reuse/audit records, legal-hold handling, export, recovery, and deletion of linked credentials, tokens, and Stratum/kernel replicas.

- **[severity: major] The design lacks the operational protections needed for public identity and tenant creation.** It does not specify recovery for lost devices, merge dispute handling, security-event auditability, enrollment/tenant/name rate limits, bot and device-farm controls, reserved-name and dispute policy, or a supported path for businesses without an IdP. The latter is acknowledged but unresolved, despite the prior model already having invited linked-personal memberships (huddle.md side letters; login-model-2026-09-08.md §2b, §6.3–4).

- **[severity: major] The huddle improves global identity naming but contradicts critical prior isolation decisions without replacing their enforcement mechanics.** Global XIDs and opaque TIDs are better than tenant-named people and mutable names, but #8 replaces D29’s membership-gated exchange and tenant-owned-services’ one-token-per-tenant model; #9 also reintroduces tenant namespace as a user request field, contrary to tenant-decisions #2. The newer model should retain D29’s live-membership/fresh-proof/mapped-role invariants and specify how gateway, policy, and delegated calls select one tenant safely (huddle.md #1, #6, #8–9; d29.md §2; tenant-decisions-2026-09-08.md #2, #9–10).

## Recommendations

1. Define merge as a Stratum serializable transaction requiring fresh proof of every moved mapping plus confirmations from both XIDs; revoke all sessions/devices, preserve an append-only audit record, and support a cooling-off/dispute window.

2. Separate authenticator assurance levels: hardware-attested device, synced credential, self-asserted desktop key, and IdP login; never call a synced key hardware-attested or human proof.

3. Replace automatic issuer ownership with a pending tenant claim flow requiring DNS verification, IdP-admin consent, or a verified organization-admin entitlement; reserve issuer binding until that succeeds.

4. Keep a tenant-scoped access token or introduce a mandatory `active_tid`, per-tenant audience, roles/scopes, and proof/membership version; services must reject any request whose namespace differs from the authorized active tenant.

5. Specify revocation SLAs: immediate local admin revocation, bounded IdP-offboarding freshness, token TTL, cache TTL, and a membership-epoch/introspection rule for high-risk operations.

6. Make Stratum the linearizable source for uniqueness, merge, revocation, names, and issuer claims, with idempotency keys and monotonic graph versions; prohibit kernel-cache-only minting when its version is stale or unavailable.

7. Publish a provider-key registry defining exact issuer validation and mapping keys: Google `(iss, sub)`, Apple `(team/client-group scope, sub)`, Entra `(tenant issuer/tid, oid)` or the chosen platform `sub` scheme, and generic OIDC `(configured issuer, sub)` only.

8. Retain D29’s membership policy concepts—invites/auto-join, freshness, mapped roles, proof issuer, and tenant-admin policy authorization—even if the storage representation becomes mappings.

9. Write product flows for no-IdP businesses, device loss/recovery, account deletion/export, tenant-name disputes, and security-event audit/rate limiting before implementation.

## What is sound

- Global, opaque XIDs and TIDs decouple durable identity from clusters, tenant names, and IdP branding (huddle.md #1, #6).

- Enforcing one XID per verified issuer/subject mapping and never using email as the primary key is correct (huddle.md #2).

- Keeping tenant-owned service SPIFFE identities and the kernel chroot rule unchanged preserves workload isolation (huddle.md #7; tenant-decisions-2026-09-08.md #1–3).

- Making the kernel own authorization-relevant graph state, with durable atomic storage for uniqueness invariants, is the right direction (huddle.md #11).

- Treating an organization login as membership proof rather than the person’s sole identity supports portability and reduces employer lock-in (huddle.md #3–4; d29.md §2).