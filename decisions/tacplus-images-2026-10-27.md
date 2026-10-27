# tacplus images: a small compiler, a big build image (DECIDED 2026-10-27)

Jody, 2026-10-27.

- **`tacplus/tacplus` (the compiler) is small.** It holds `tacplus`, `protoc` and the protoc plugins, and nothing else:
  **no language toolchain** (no go, cargo, rustfmt, javac, mvn, node, …). Generation must work with no toolchain on PATH.
  Formatting happens in-process (e.g. `go/format`) or is left to the build.
- **`tacplus/build` (the build environment) is big, and that is fine.** It carries every toolchain needed to compile
  and test generated code in every language. The looper image is built from it, and gates run there.
- A smoke test enforces the split: generating a fixture in every language inside the compiler image with no toolchain
  on PATH must succeed (tacplus-dev/tacplus#189).
