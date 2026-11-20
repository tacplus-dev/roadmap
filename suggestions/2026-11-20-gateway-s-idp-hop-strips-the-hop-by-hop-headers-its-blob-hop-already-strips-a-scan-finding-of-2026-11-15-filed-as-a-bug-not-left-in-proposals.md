Suggestion: gateway's IdP hop strips the hop-by-hop headers its blob hop already strips: a scan finding of 2026-11-15 filed as a bug, not left in PROPOSALS
Type: Security
Summary: IdpProxy::proxy_request (src/idp_proxy.rs:128-134 and 156-158) copies every request and response header but Host onto the Keycloak hop, connection, keep-alive, te, trailer, transfer-encoding and upgrade included, while both bodies are re-framed (the request collected into a Full, the response streamed by hyper), so a forwarded transfer-encoding or connection describes a connection the next hop did not agree to; blob_proxy::forwardable (src/blob_proxy.rs:326) drops exactly those on its hop and says why. The nightly scan found it on 2026-11-15 and left it in PROPOSALS ('No failure observed'); no issue holds it. File it approved and reuse forwardable (pub(crate)) in both directions of idp_proxy, keeping Host set to the backend and the X-Forwarded-* headers
Why: The IdP hop is the sign-in path of every fleet (bar B is seatrials signing in on dev), and forwarding framing and connection headers across a re-framing proxy is the classic request-smuggling and keep-alive mismatch; the fix is bug-sized, in a repository whose lands are held only by the security floor (gateway#34, #35), and a scan finding of the security kind should become an issue, not a proposal nothing reads into work. Also serves Correctness: the two hops then forward the same way.

## Evidence
- https://github.com/tacplus-dev/gateway/blob/main/PROPOSALS.md ('idp_proxy forwards the hop-by-hop headers blob_proxy strips', 2026-11-15, nightly scan: 'a forwarded transfer-encoding or connection describes the wrong connection'; 'Fix, bug-sized: reuse forwardable (make it pub(crate)) for both directions'; 'No failure observed; found by reading for trace propagation')
- https://github.com/tacplus-dev/gateway/blob/main/src/idp_proxy.rs (lines 128-134: every request header but Host copied, then Host set to the backend endpoint; 156-158: every response header copied; the request body is a Full, the response a hyper stream; read 2026-11-20)
- https://github.com/tacplus-dev/gateway/blob/main/src/blob_proxy.rs (forwardable, line 326: drops connection, keep-alive, proxy-authenticate, proxy-authorization, te, trailer, transfer-encoding, upgrade and host: 'forwarding them is how a proxy ends up advertising a framing or a keep-alive the next hop did not agree to')
- https://github.com/tacplus-dev/gateway/issues (checked 2026-11-20: no open issue names idp_proxy's headers; the open bugs are the security floor's, #29 to #43, and #6, the release, is first in bar A)

## Cost
Hours: make forwardable pub(crate), use it for the request and the response headers in idp_proxy, and a unit test that a chunked, keep-alive request reaches the backend without transfer-encoding and connection and that a backend's connection header does not reach the client. Lands on gateway's unit set.

## Touches
decisions/gateway-per-fleet-2026-10-27.md (the gateway is the fleet's one edge), items/login-flow-design-2026-10-29.md (sign-in through the gateway's IdP hop), decisions/seabeam-can-start-2026-11-12.md (bar B: seatrials signs in on dev), looper#50 (a security finding is filed, not proposed)

## Reuse considered
forwardable exists in the same crate and is the fix: nothing new is built. Considered: hyper-util's proxy helpers or a tower layer for hop-by-hop stripping, a dependency for one function the crate already has. Considered: leaving it as the scan's proposal, which no run picks up (the scan files bugs, and this one it did not).

## Status
open
Rank: 2
Filed: 2026-11-20 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18215309726)
