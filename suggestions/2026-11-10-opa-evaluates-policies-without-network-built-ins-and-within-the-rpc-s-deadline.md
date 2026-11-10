Suggestion: opa evaluates policies without network built-ins and within the RPC's deadline
Type: Security
Summary: opa prepares every pushed bundle with OPA's default capabilities, so a policy can call http.send and the DNS built-ins from opa's /system pod, and both PrepareForEval and Eval run on context.Background(), with no deadline, cancellation or trace_id. Give rego a capabilities set without the network built-ins, and evaluate on the RPC's context
Why: opa decides entitlements and proxy policy, so it sits on kernel's sign-in and token-minting path. A policy that calls http.send can reach anything opa's pod can, the platform's internal endpoints among them: an SSRF from a policy edit. And a slow or hanging call holds the RPC indefinitely, since nothing cancels it. Policies are platform config, not tenants' code, so this is defense in depth against a bad push or a compromised pusher, not an open door. But nothing needs these built-ins, and removing them costs a list. opa's nightly scan proposed it on 2026-11-06, and its PROPOSALS also cite a security-audit Medium ('RPC context is discarded'). No open opa issue holds either (opa#2, #5, #6, #9, #10). Also serves Resiliency (no stuck evaluations) and trace propagation (decisions/trace-propagation-2026-11-03.md).

## Evidence
- https://github.com/tacplus-dev/opa/blob/main/internal/opa.go (line 44: rego.New(opts...).PrepareForEval(context.Background()) with no rego.Capabilities among the options; line 62: pe.query.Eval(context.Background(), rego.EvalInput(input)))
- https://github.com/tacplus-dev/opa/blob/main/PROPOSALS.md ('Restrict the Rego built-ins policies may call', 2026-11-06, nightly scan: a pushed policy can call http.send and the DNS built-ins, from opa's pod, on context.Background(), no deadline, no cancellation and no trace_id)
- https://github.com/tacplus-dev/opa/issues (open: #2 release, #5 gates, #6 alpine, #9 pin, #10 root image; none on built-ins or the evaluation context)

## Cost
Hours to a day in opa: NewPolicyEvaluator passes rego.Capabilities built from ast.CapabilitiesForThisVersion() minus http.send, net.lookup_ip_addr and the other network built-ins (a policy using one fails to load, refused at push like a wrong entry point); Evaluate takes the handler's ctx (rego's eval honours its deadline) and the handler passes the RPC's context. Tests: a bundle calling http.send is refused; an evaluation past the RPC's deadline returns DEADLINE_EXCEEDED.

## Touches
decisions/trace-propagation-2026-11-03.md (no ambient context: the request's context and trace_id carried through), kernel#41 (proxy policy output that can overwrite kernel-set claims: the same policy surface), the security floor (looper#49)

## Reuse considered
OPA's own rego.Capabilities option is the mechanism, so nothing new needs building. loadPolicy already refuses a bundle whose entry point doesn't match its slot, and an unknown built-in is refused at the same point. Considered: a NetworkPolicy on opa's egress, but that is ruled out by decisions/s2s-never-crosses-tenants-2026-10-31.md's 2026-11-07 addendum (no NetworkPolicies) and wouldn't bound the evaluation's time.

## Status
open
Rank: 2
Filed: 2026-11-10 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17872762621)
