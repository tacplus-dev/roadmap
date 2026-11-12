Suggestion: A secret finding keeps its identity when its line moves, and each repository's known fixtures sit in a reviewed .gitleaks.toml
Type: Security
Summary: Outside generated code (which #108 covers), the security floor fails kernel's lands on 8 gitleaks CRITICALs, oauth's on 18 and looper's on 5. All of them are PEM-assembly code or test-only keys, e.g. KeyRotator.java:667 builds '-----BEGIN PRIVATE KEY-----' around a key it has just generated. No repository but tacplus has a .gitleaks.toml, and an exception can't hold, because a secret's id carries its line. Key secret ids on gitleaks' fingerprint without the line, and give each repository a reviewed allowlist, as tacplus's own
Why: Tonight 0 of 466 to 526 findings anywhere were accepted by an exception, and every land in kernel, oauth and looper fails security-<repo> on fixtures (looper's 17 failed security jobs tonight, oauth's 4, kernel's 1). That costs the throughput-first plan its merges. Worse, a secret that really leaked would be one more CRITICAL among the same 25 that everyone has learned to skip, so the floor no longer tells a leak apart. The two pieces are already written up: the README says gitleaks reads the tree's .gitleaks.toml and .gitleaksignore, and looper's PROPOSALS 'A secret's identity that survives a moved line' (2026-11-12) shows #120's fixture coming back as a new CRITICAL at :2290 after #105 added lines above it, so any dated exception would stop matching. Today these are fixed one finding at a time: looper#117, #119 and #120 are three issues for one cause, and kernel and oauth have none. Also serves Performance (lands that pass the floor merge) and Developer experience

## Evidence
- https://gitlab.com/tacplus-dev/build/-/jobs/17932118588 (security-kernel, gate_failure: 8 blocking gitleaks findings, all hand-written: KeyRotator.java:667 and ServiceIntermediateCa.java:186 concatenate '-----BEGIN PRIVATE KEY-----' around keyPair.getPrivate().getEncoded(); IdentityServiceImpl.java:294-295, a KeyType switch; four test classes; 9 new since the previous run)
- https://gitlab.com/tacplus-dev/build/-/jobs/17932197537 (security-oauth, gate_failure: 25 gitleaks CRITICALs, 18 of them hand-written test keys: src/apple.rs:104 and src/auth.rs:1273 are #[cfg(test)] 'Fixture P-256 keypair ... Test-only' consts, plus tests/fixtures/ed25519-*.key, rogue.key and jwt-alg-cases.json)
- https://github.com/tacplus-dev/tacplus/blob/main/.gitleaks.toml (the pattern: useDefault plus an allowlist of PEM assembly and redaction fixtures, each named, with the reason; no other repository has one: kernel, oauth, looper, opa, operator, coop, llmproxy, mailbox, messaging and doppelganger all 404)
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('A secret's identity that survives a moved line', 2026-11-12, #120, #129: a gitleaks id is <path>:<rule>:<line>, for both new and an exception's id)
- https://github.com/tacplus-dev/looper/issues/120 (one of three per-finding issues for looper's fixtures, with #117 and #119)

## Cost
A day in looper-scan: a secret's id becomes path:rule:<gitleaks Fingerprint less its line, or a hash of the match>, still read the old ids for one release, and a test that a moved fixture stays the same finding and keeps its exception. Then an hour per repository: a .gitleaks.toml (useDefault, plus paths for tests/fixtures/ and the named PEM-assembly files) in kernel, oauth and looper, each entry with its reason, reviewed like an exception. Real keys stay caught everywhere else, and the floor's planted-secret smoke still runs

## Touches
looper#49 (the deterministic floor: a secret fails, unless an exception names it), decisions/throughput-first-2026-11-11.md (security findings and what blocks merging come first), looper#108 (the generated tree, its half), looper#129 (one issue per cause), looper#117/#119/#120 (folded into this one cause), kernel and oauth (no issue yet)

## Reuse considered
gitleaks already reads a tree's .gitleaks.toml and .gitleaksignore (looper README, Security checks), and tacplus's file is the template, so nothing new is needed for the allowlist. security-exceptions.yaml is the other route, but it is per finding, dated, and keyed by line, so it breaks on the next edit. An inline 'gitleaks:allow' comment works for code (KeyRotator) but not for key files such as tests/fixtures/*.key. #108 skips only the generated tree, so it doesn't reach hand-written fixtures. The fingerprint change is looper's own PROPOSALS entry, not yet an issue

## Status
open
Rank: 1
Filed: 2026-11-12 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17962167524)
