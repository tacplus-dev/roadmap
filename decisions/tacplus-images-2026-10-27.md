# tacplus: a binary, and one build image per release line (DECIDED 2026-10-27)

Jody, 2026-10-27. This replaces the earlier version of this record (a small compiler image plus a big build image).

- **There is no tacplus image.** tacplus is a **binary**, published with each release (`tacplus-X-Y-Z-<os>-<arch>` plus
  `checksums.txt`; the repo is public).
- **One build image per release line: `ghcr.io/tacplus-dev/tacplus/build:X.Y`.** It carries every toolchain (Go, Rust, Java and
  Maven, Python, Node, …) and the verifiers (Kani for Rust; JBMC and OpenJML for Java), and is rebuilt on every patch release of
  its line. Big is fine. tacplus's own CI and gates run in it. Loopers do NOT run in it (decisions/deterministic-builds-2026-10-27.md).
- **In the image, `tacplus` is a shell script** that runs the repository's **pinned** tacplus version (decisions/deterministic-builds-2026-10-27.md); only outside a repository does it fall back to the newest binary of the image's line:
  - It finds the newest `vX.Y.*` release, downloads the binary for the platform, **verifies it against `checksums.txt`**, caches it
    under `~/.cache/tacplus/<version>/`, and execs it with the arguments.
  - It checks for a newer patch at most about hourly, and otherwise uses the cache (so it also works offline when cached).
  - `TACPLUS_VERSION=X.Y.Z` pins an exact version, so fleet builds are reproducible (the lock records the version).
  - It refuses a repository whose `tacplus.yaml` floor names a different MAJOR.MINOR than the image, saying which image to use.
  - The same script works on a developer's Mac (darwin binaries are published).
- **Activities:** generation, compile, test, fuzz and formal verification all run in the build image.
- **No formal verification for Go:** Go's tooling is too weak. Rust (Kani) and Java (JBMC/OpenJML) are verified, which covers Go
  well enough because the generated code is the same logic. Go is fuzzed and tested.
- **Later:** the build image may be split by language (e.g. build-go, build-rust, build-java), each carrying one toolchain and its verifier (Jody, 2026-10-27).
