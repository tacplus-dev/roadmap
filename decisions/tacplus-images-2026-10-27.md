# tacplus images: a small compiler, a big build image (DECIDED 2026-10-27)

Jody, 2026-10-27.

- **`tacplus/tacplus` (the compiler) is small.** It holds `tacplus`, `protoc` and the protoc plugins, and nothing else:
  **no language toolchain** (no go, cargo, rustfmt, javac, mvn, node, …). Generation must work with no toolchain on PATH.
  Formatting happens in-process (e.g. `go/format`) or is left to the build.
- **`tacplus/build` (the build environment) is big, and that is fine.** It carries every toolchain needed to compile
  and test generated code in every language. The looper image is built from it, and gates run there.
- **Building tacplus itself is tested in the build image.** Its CI and `just gates` (unit tests, cross-tests that compile
  and run generated code in every language, determinism, current-generation) run in `tacplus/build`, which therefore
  carries every toolchain. Only the shipped compiler image is kept small.
- A smoke test enforces the split: generating a fixture in every language inside the compiler image with no toolchain
  on PATH must succeed (tacplus-dev/tacplus#189).

## By activity, not by language (amendment, Jody 2026-10-27)

| Activity | Runs in | Needs |
|---|---|---|
| Generate (`tacplus build`'s generation) | the compiler image | no toolchain; small |
| Compile, test, **fuzz**, **formally verify** (`tacplus verify`, fuzzing, gates) | the build image | every toolchain, JBMC/OpenJML (Java), Kani (Rust); big |

- The **build image carries the `tacplus` binary**, so `tacplus verify` and fuzzing run there directly (no compiler
  container). The looper image (built on it) runs verify and fuzzing natively.
- The compiler image keeps only what protoc plugins need to run (e.g. `node` for the TypeScript plugin, the Swift
  runtime). Those are plugin runtimes, not toolchains.
- **No formal verification for Go:** Go's formal-verification tooling is too weak to be worth it. Rust (Kani) and Java
  (JBMC/OpenJML) are verified, and because the generated code is the same logic in every language, that covers Go well
  enough. Go is still fuzzed and tested.
