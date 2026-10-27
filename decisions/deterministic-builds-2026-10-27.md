# Deterministic builds: the build has no agent; the looper has no build verdict (DECIDED 2026-10-27)

Jody, 2026-10-27: "builds always deterministic no matter who does this … builds that either pass or fail. So we will
know if builds succeed on their own or if the agent is doing something."

## Two separate images, two separate jobs

| | Looper image | Build container |
|---|---|---|
| Carries | an agent, git, gh; may pull tacplus to experiment and learn its commands | the build image `tacplus/build:X.Y` (toolchains, verifiers); **no agent** |
| Does | edits, commits and pushes to its branch, **starts a build**, reads the verdict, iterates; lands on `main` only after a green build | takes a commit **as it is**, builds it, runs the gates (tests, fuzzing, formal verification) and publishes the image |
| May | change the repository | **never** change the repository: read-only access to source, write access only to the image registry |

A looper's own runs are never the verdict. Only the build is.

## A build is fully determined by its commit

- The repository **pins** the exact tacplus version and the build image **digest** in a committed file. Moving to a newer
  patch or image is a **commit** (by a looper or a person), never something a build does by itself.
- The build fetches exactly the pinned tacplus binary and checks it against the release's `checksums.txt`. It never
  resolves "newest".
- `build:X.Y` remains a convenience tag for people; builds use the pinned digest.
- Same commit, same result, whoever or whatever starts the build.

This refines decisions/tacplus-images-2026-10-27.md: the image's `tacplus` script runs the **pinned** version (falling back to the
newest patch only outside a repository, e.g. for someone experimenting), and the loopers do **not** run in the build image.
