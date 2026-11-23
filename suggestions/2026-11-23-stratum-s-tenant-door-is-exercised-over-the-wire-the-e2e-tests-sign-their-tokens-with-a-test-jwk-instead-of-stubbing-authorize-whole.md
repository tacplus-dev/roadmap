Suggestion: Stratum's tenant door is exercised over the wire: the E2E tests sign their tokens with a test JWK instead of stubbing authorize whole, so another tenant's caller is refused in a test, not only in production
Type: Correctness
Summary: stratum's two E2E tests of the tenant-owned record plane, TenantQueryScopeE2ETest and RecordWriteSurfaceE2ETest, override Authorizer.authorize whole (an anonymous subclass returning the claims unverified), so, as the test's own comment says, 'The tenant door this instance's authorizer keeps (another tenant's caller refused at authorize) is not reached: authorize is stubbed whole'; only the route check and the handle check behind it run. stratum's PROPOSALS has asked since 2026-10-30 (#20) for the tests to sign their tokens with a test JWK source so the shipped authorize and setHostNamespace's refusal (tacplus#207) are on the path; no issue holds it. Per-tenant Stratum is the platform's highest-priority path (Next 1) and operator#2 is about to render one per tenant: add the signed-token E2E leg (a wrong-tenant caller refused PERMISSION_DENIED before any store is touched, the right one served), in stratum and as a cross-test fixture in tacplus so every generated Java server gets it
Why: Correctness, serving Security: the one check that keeps tenant A's caller out of tenant B's Stratum instance has no test over the wire, three weeks after the proposal, while the floor's Java SAST and SCA run on nothing for stratum (its report tonight: 'spotbugs: Java sources, but the gates compiled no target/classes', 0 findings), so nothing automated looks at this code at all. A test is cheap and is what the Correctness type exists for

## Evidence
- https://github.com/tacplus-dev/stratum/blob/main/PROPOSALS.md (2026-10-30, #20): 'E2E tests through the instance's tenant door: TenantQueryScopeE2ETest and RecordWriteSurfaceE2ETest stub Authorizer.authorize whole to skip the token signature check, so the door that refuses another tenant's caller on a tenant-owned instance (setHostNamespace, tacplus#207) is never exercised over the wire here; only the route check and the handle check behind it are. Signing the test tokens with a test JWK source instead would put the shipped authorize on the path'
- https://github.com/tacplus-dev/stratum/blob/main/server/src/test/java/com/github/tacplusdev/stratum/TenantQueryScopeE2ETest.java (lines 174-183: 'The tenant door this instance's authorizer keeps (another tenant's caller refused at authorize) is not reached: authorize is stubbed whole'; new Authorizer(sink -> { }) { public TokenWrapper authorize(...) }) and https://github.com/tacplus-dev/stratum/blob/main/server/src/test/java/com/github/tacplusdev/stratum/RecordWriteSurfaceE2ETest.java (lines 60, 84, 108-121: '.stub-signature', setHostIdentity(TENANT, ...))
- tacplus-dev/tacplus#207 (closed): setHostNamespace, the instance's tenant door; decisions/stratum-per-tenant-2026-10-25.md; tacplus-dev/operator#2 (open, the platform ROADMAP's Next 1.2): per-tenant Stratum rendered next
- tonight's stratum security report (the scan's context, security-stratum/summary.md, commit c20dad9c): 0 findings, 'spotbugs: source: Java sources, but the gates compiled no target/classes', osv-scanner's Maven resolution failed on stratum-server 1.2.0-SNAPSHOT (the open suggestion of 2026-11-15, rank 2); stratum's open issues tonight (#29, #36, #44) hold no test of the tenant door

## Cost
A day in stratum (a test JWK source, the two tests signing with it, one wrong-tenant leg each); a day more for a cross-test fixture in tacplus so the Java servers of every service get the leg

## Touches
decisions/stratum-per-tenant-2026-10-25.md; decisions/s2s-never-crosses-tenants-2026-10-31.md; decisions/tenant-routing-2026-10-26.md (one instance per tenant); decisions/kitchensink-verification-2026-10-27.md (tests that prove the generated contracts); the open suggestion of 2026-11-15 (a security run that analyzed none of a Java service's dependencies is not green)

## Reuse considered
the generated Java chassis's Authorizer already verifies against a JWK source, and kernel's tests sign test tokens; tacplus's cross-test already has stream-expiry and tenant legs for the Go, Java and Rust servers (the accepted suggestion of 2026-11-19 added a Rust leg the same way), so a wrong-tenant leg fits that harness; stratum's E2E tests already stand up the gRPC server and the record maps, only the authorizer changes

## Status
open
Rank: 5
Filed: 2026-11-23 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18274230893)
