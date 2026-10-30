# Rust is the default language (Jody, 2026-10-27; recorded 2026-10-30)

- **Rust is the default** for anything new in tacplus-dev: services, clients, tools, and the build recipes the platform offers (for example, the fleet builder's client recipe, tacplus#195).
- **Go or Java only for a reason written into the issue or design.** Java where a service is built on a Java-only foundation is one example. Go where a service is already written in Go and the work extends it is another.
- "The existing code, toolchain or image already does Go" is **not** a reason on its own. When the reason is a real constraint (for example, cross-compiling to darwin), name the constraint and solve it, or ship that piece later. Do not switch language.
- coop and pigeon are Go. That is accepted as it stands, and they are not rewritten unless Jody asks.
- Why it is recorded now: the rule lived only in the hub's notes. tacplus#195's proposal made clients Go-only because the build image's Go toolchain was the easy path, and Jody asked why there was a Go-only proposal at all.
