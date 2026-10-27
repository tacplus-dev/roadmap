Do not treat the note as decided. Push back first on items 12 with 5, 13 with 14, and 10. Then 4, and the Stratum side letter.

(1) Inconsistencies, by item number:
- Items 12 and 5. Item 12 says device attestation proves the user is a human and can anchor an XID. Attestation proves a genuine hardware-backed key, not a person. Item 5 also lets self-asserted keys (synced keychain, desktop) be mappings under our own issuer. If those can mint an XID, the anchor proves nothing. Jody would push back: no XID is minted from a key the party generated alone.
- Items 13 and 14 together. Merge deletes XIDs with no tombstone and no restore path. The repo's definition of irreversible (confirmed 2026-10-04) is destroys data or state with no restore path. Under his bias to action, that is a human decision, not a call the agent makes. "Keep oldest" is also a policy choice that should be his.
- Item 4. Org membership ages out when the org stops logging in, with no signal. A bounded lifetime is fail-safe, and he endorses that: "Ethernet should have had a TTL. Because it should have been fail safe." But the note states no bound and gives no way to cut off a live token early. He would push back on removal depending on the absence of a refresh.
- Item 10. The first org login from an unbound issuer creates the tenant and binds the issuer, first-come. For shared-domain issuers, whoever arrives first owns that issuer for everyone after. A login at an issuer is not authorization to a tenant: "In networking reachability implies authorization was and is a logical fallacy." The kernel must verify the binding, not take the first claimant. "Minimal fuss" is a huddle line, not a decision.
- Side letter on Stratum. A domain-level Stratum is required but not yet in the quorum equation. "I believe in robust systems with no single point of failure." Pushback.
- Consistent with his views: items 1, 2 and 7. Identity bound to the thing (XID, immutable subject keys, never email) and services fixed match "abstractions must complete[ly] hide the layer below from the layer above" and his TCP note.
- Caveat: the note attributes items 12 to 14 to Jody. In the repo those are the huddle record, not his marked words. I cannot verify them as his words.

(2) Open items, with his likely answer and my confidence:
- Memberships claim: TIDs only. Roles become kernel-attested facts that policy reads, so a role change does not wait for token expiry. The repo says tenant facts are attested by generated code and a tenant may supply facts to the policy engine but never decide. Confidence about 55%. Token lifetime: short and bounded, which is his fail-safe principle. The number is not in the repo; confidence on any specific value is about 30%.
- Per-tenant IdP record: the tenant admin may tighten and never loosen. Examples: require device assurance, narrow domains, shorten lifetime. The admin cannot change the subject key, the issuer binding, or the platform's proof. The repo says the substrate owns the ceiling, the tenant owns the value and a tenant gate is stricter, never looser. Confidence about 70% on direction.
- Device mappings: an explicit assurance level on every device mapping. Hardware-attested keys can anchor an XID. Self-asserted keys are a lower tier and cannot anchor alone. He prefers explicit: "I am actually okay with us being verbose." Confidence about 60%.
- Migration: incremental and reversible where possible, with no deprecation window, and canary on the hand-written kernel code. "I believe in canary deployment models." "I believe in incremental changes with a solid fallback strategy." The repo says no window, and generated consumers regenerate. Confidence about 70%.

(3) What he would say is missing:
- Revocation of live tokens, and what happens when a mapping is removed mid-lifetime.
- Stratum failure behaviour. The kernel working copy and Stratum can diverge, and the note does not say what fails closed. Quorum is the missing piece.
- Tests. "I believe that testing must be done at all stages of development." Missing: a concurrent-login uniqueness test, and a merge test.
- An audit trail for identity writes and merges.
- A recovery path. If the personal anchor is lost, the XID is lost and Scotty goes with it. Personal-first was adopted for that reason.
- A status line saying which items are decided and which are huddle-only.
