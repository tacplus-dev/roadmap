# Every build runs in a container; the VM is vanilla (Jody, 2026-11-03)

Jody: "I think all builds should always occur in containers with no exceptions. The VM should be very vanilla. The build
containers should have all of our stuff including the possibility of build caching to speed up builds. If the build will
also produce a container image then it will need to do so inside a container." Asked about the OS, the image builder and
the cache location: "Yes on all".

Part of the looper redesign huddle (one job per issue; `items/looper-job-per-issue.md` once written). It refines
`deterministic-builds-2026-10-27.md`.

## Decided

- **The VM is vanilla:** Google's **Container-Optimized OS** (Arm/T2A), pinned by image name (never a family). Nothing is
  installed at boot. The container runtime is the OS's own. This removes the baked runner image (cloud#4, looper#43,
  looper#67's bake) and the first-job Docker race.
- **Every build runs in a build container,** without exception. The container carries all our tooling: tacplus, compilers,
  verifiers, scanners.
- **Agents never build.** An agent job requests a build. The build runs as its own job in a build container, and the agent
  reads the verdict (as `deterministic-builds-2026-10-27.md` already says). So an agent container gets **no container
  runtime and no host Docker socket**.
- **Image builds happen inside the build container,** with **rootless BuildKit**: no daemon on the host, the same
  Dockerfiles, multi-arch. Fleet images built in Cloud Build already comply.
- **Caching lives outside the VM:** a **bucket and a registry in the `ci` project**, written only by build jobs, read by all
  of them:
  - BuildKit registry cache for image layers;
  - remote compiler caches (sccache for Rust, the Maven/Gradle remote cache, the Go build cache);
  - keyed by content, so a cache can't poison a build.
- **A cache only speeds a build up.** The verdict and the artifacts are identical with a cold or a warm cache, and a test
  proves it (`deterministic-deployments-2026-11-02.md`).

## What changes
- **The runner pools** (cloud#4) boot COS, pinned, with no bake.
- **looper#67** (bake in `tacplus-ci`) and the baked-image half of **looper#43** are superseded.
- **`looper-docker` and the host-socket gates** are replaced by build jobs, which also covers looper#68's Docker broker.
- The build image's Dockerfiles gain rootless BuildKit and cache configuration.
- Issues follow once the huddle's remaining questions are settled.
