# Verify the generator once, on a kitchensink; services don't re-prove generated code (DECIDED 2026-10-27)

Jody, 2026-10-27.

- **A kitchensink proto set in tacplus**, built deliberately to cover every feature the generator supports: every field type, every
  `spec` option and assertion, every auth mode, records, state, streaming, tenant routing, MCP carriers, and edge cases (empty
  messages, deep nesting, size-bound limits, keyword-like names). The existing `cross-test/proto-*` fixtures are its starting point.
- **Formal verification (Rust: Kani; Java: JBMC/OpenJML) and fuzzing run on the kitchensink in tacplus's own build pipeline, as a
  release gate.** A tacplus release is verified once; every service inherits it. Go is fuzzed, not formally verified
  (decisions/tacplus-images-2026-10-27.md).
- **Every service build runs a cheap coverage check:** tacplus records which features and combinations the kitchensink covers, and
  generating for a service reports anything it uses that isn't covered. That fails the build (or verifies just that part), and the gap
  is added to the kitchensink, so it grows from real use.
- **Services no longer run the generated proof harnesses themselves** (e.g. kernel's 66). They fuzz their own hand-written code.
