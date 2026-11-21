Suggestion: opa's proxy policy fails closed on a result field of the wrong type, as it does on token_use: a dropped append_aud or extra_claims is a token the policy did not mean
Type: Security
Summary: EvalProxyPolicy (internal/handler.go) reads a policy result's url, reason, append_aud, extra_claims and auth_type with a type assertion each, and a present field of the wrong type (a string for append_aud, a list for extra_claims) is skipped without an error: kernel then mints the proxied token without the audience or claims the policy wrote, while an unrecognised token_use fails the call with codes.Internal since opa#13. opa's PROPOSALS has asked for the same rule on every field since 2026-11-12, needing Jody's approval because a policy that works today could be refused; no issue holds it. Approve it: a present field of the wrong type returns codes.Internal naming the field, with a test per field
Why: opa decides entitlements and proxy policy for every tenant, and a policy bug or a rego type slip must not silently change what kernel mints: the decider fails closed or it is not a decider. Security (fail closed on the entitlement seam); kernel#41, priority:high, is the other side of the same seam.

## Evidence
- https://github.com/tacplus-dev/opa/blob/main/PROPOSALS.md (A proxy-policy result's other fields fail closed too, 2026-11-12, #13: 'url, reason, append_aud, extra_claims and auth_type of a wrong type ... are still dropped silently (internal/handler.go, EvalProxyPolicy)'; 'opa decides entitlements, and a policy bug that silently drops append_aud or extra_claims mints a token the policy did not mean'; 'it needs Jody's approval')
- https://github.com/tacplus-dev/opa/blob/main/internal/handler.go (EvalProxyPolicy: if appendAudRaw, ok := obj["append_aud"]; ok { ... } and if m, ok := extraRaw.(map[string]interface{}); ok { ... } with no other branch; token_use's parseTokenUse alone returns codes.Internal, 'token_use %v (%T) is not a TokenUse name or value')
- https://github.com/tacplus-dev/opa/issues/13 (closed 2026-11-12: token_use fails closed; the other fields left as they were)
- https://github.com/tacplus-dev/kernel/issues/41 (open, priority:high: a proxy policy's extra_claims overwriting kernel's claims, the same seam from kernel's side)

## Cost
Hours in opa: five type checks returning codes.Internal with the field and its type named, a test per field, a README line on the result's schema; a policy whose result carries a wrong-typed field fails from then on, which is the point, and the seatrials bundle is checked against it before the land.

## Touches
decisions/identity-session-credential-2026-10-25.md and s2s-never-crosses-tenants-2026-10-31.md (what a proxied token carries, and for whom), the fail-closed rule opa#11 and #13 set (no network built-ins, token_use refused), kernel#41

## Reuse considered
opa#13's parseTokenUse is the pattern: the value and its type named in a codes.Internal, nothing new to build. A JSON schema of the result checked once at load (OPA's schema annotations on the entrypoint) would catch the same in one place, at the cost of a schema kept beside the proto; the five checks are smaller and land first.

## Status
open
Rank: 3
Filed: 2026-11-21 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18244259587)
