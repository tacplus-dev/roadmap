Suggestion: A Rust service's handler-provided streams end at their token's exp, as API.md promises of every stream
Type: Security
Summary: docs/API.md says the generated Go, Java and Rust servers end every streaming RPC at its opening token's exp. In Rust they do not all: StreamCredentialExpiryLayer is a tower layer that polices only the generated STREAMING_RPC_PATHS, so a stream on a handler's own additional_services route outlives its token (tacplus PROPOSALS, 2026-11-01, #260; STATE.md's known gap 'Rust does not yet cover handler additional_services streams'), while Go and Java learn the call type from gRPC's interceptors and end those too. No issue has held the Rust half since #260 closed on 2026-11-02. Read the streaming paths off the handler's routes (or a handler-declared list), end them at exp, and prove it in cross-test's stream-expiry leg for a handler route, in the 2.6 line too
Why: Rust is the default language (decision 2026-10-27) and the services on it (gateway, oauth, llmproxy, mailbox, messaging) are what a tenant's clients hold streams to; a token's exp is the one end the platform promises a live stream (tenant deletion: 'tokens already issued are not chased: ATs and STs are short-lived'), and a handler route is where a service's own long streams live. A contract line that holds in two languages of three is TYPES.md's own Correctness example; the main benefit is that an expired, or a deleted tenant's, credential stops at exp on every route. Also serves Correctness

## Evidence
- https://github.com/tacplus-dev/tacplus/blob/main/docs/API.md ('Servers terminate streams carrying expired credentials. The generated Go, Java and Rust servers end every streaming RPC' at the opening token's exp)
- https://github.com/tacplus-dev/tacplus/blob/main/PROPOSALS.md ('End Rust additional_services streams at their token's exp', 2026-11-01, #260: 'The Rust StreamCredentialExpiryLayer is a tower layer that cannot, so it polices only the generated STREAMING_RPC_PATHS, and a stream on a handler's own additional_services route still outlives its token. Reading the streaming paths off the handler's routes (or a handler-declared list) would close it'; no issue)
- https://github.com/tacplus-dev/tacplus/blob/main/STATE.md (Known gaps, 2026-11-18: 'Rust does not yet cover handler additional_services streams (#260)')
- https://github.com/tacplus-dev/tacplus/issues/260 (closed 2026-11-02: 'Verify servers end a stream at its token's exp (API.md claims it); fix any language that only checks at open': the generated paths fixed in all three languages, the handler routes left open in Rust)
- https://github.com/tacplus-dev/roadmap/blob/main/decisions/tenant-deletion-2026-10-27.md ('Tokens already issued are not chased: ATs and STs are short-lived': the exp is the end of a deleted tenant's live streams); https://github.com/tacplus-dev/roadmap/blob/main/decisions/rust-default-language-2026-10-27.md

## Cost
Days in tacplus: the Rust chassis template feeds the handler's additional_services routes (or a handler-declared streaming list) into the expiry layer's path set; a cross-test stream-expiry case for a Rust handler route; a 2.6 patch, which the fleets' pins take

## Touches
decisions/rust-default-language-2026-10-27.md, decisions/tenant-deletion-2026-10-27.md (tokens end at exp), decisions/identity-session-credential-2026-10-25.md, items/tacplus-security-sweep.md, PRODUCTION.md's security review (the contract it relies on), the Correctness type's own example in suggestions/TYPES.md

## Reuse considered
The Go and Java servers already do it through their interceptors, and the Rust layer already ends the generated paths: the change is the path set, not a new mechanism. cross-test's stream-expiry leg (#260) is the proof to extend. Considered: documenting the gap in API.md instead (the contract is what consumers and the go-live review rely on); leaving it to each service (a handler route is generated for every Rust service, so the fix belongs in the generator once)

## Status
accepted (Jody, 2026-11-20): tacplus-dev/tacplus#931
Rank: 2
Filed: 2026-11-19 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18176428236)
