# Exploration: protos live with their owners ("Types Cherry Picker")

- **Status:** exploring (a thought, not a plan; Jody 2026-10-22: "not to implement yet but to
  explore and determine if we should or should not do it")
- **Owner:** Jody
- **Opened:** 2026-10-22
- **Gets more expensive while held:** nothing; the 3-source build works.
- **Records:** `sonar/backend/build.sh` (the three-source build and its skew comments);
  `tacplus/docs/USER.md` (`src`/`sources`, git sources, dependencies.yaml merge);
  `tacplus/docs/VERSIONING.md` (the MAJOR.MINOR + floating patch policy)

## The idea (Jody)

Instead of one repo holding every `.proto`, each service, client or library keeps the protos it
defines. At build time they are gathered and built with tacplus's existing multi-source support;
cross-imports must still resolve. The kernel gets everything. A side benefit: a service or utility
would carry only the types it actually uses.

## What already exists

The SeaBeam backend build already merges three proto trees (platform, backend, docmanager) through
`sources`; tacplus supports git sources with tags, byte-identical dedup, merged `dependencies.yaml`
with conflicting pins as an error, and per-repo `registry.yaml` with an `id_min` floor so service
ids do not collide. The step is from 3 hand-listed sources to N resolved ones.

## What is new, and what the central repo was quietly doing

- **A resolver**: package -> owning repo + ref, transitive import closure, a lock. This is a proto
  package manager; the gathering is already done.
- **The atomic fleet snapshot** that one central commit gave: fleet-wide checks (state_import
  targets, eventbus wire contract, store quorum, grants catalog) ran on one agreed set.
- **Rebuild fan-out**: kernel, gateway and operator are compiled against the whole service set and
  rebuild on every change regardless; other importers need a reverse index.
- **Skew** is already visible at N=3 (build.sh: platform/proto main ahead of what the pinned
  generator parses).

## Direction agreed in the huddle (Jody, 2026-10-22)

- **Be smart at build time.** Any proto change rebuilds the kernel. Then determine whether it
  impacts another service.
- **Protos are pinned by MAJOR.MINOR with the patch floating**, the same policy as tacplus (not done
  yet: the backend builds with `PROTO_TAG=latest`). A patch flows to every importer with no repin;
  a minor is an explicit move; a major is rebuild-not-rollout.
- Build-time parts, all with inputs tacplus already has: (1) impact set from a fleet manifest plus
  the parsed import graph; (2) classify a change by diffing descriptors between tags, patch / minor /
  major, INCLUDING `spec` annotations (a changed `auth_services` is wire-compatible and still a
  breaking generated change), refusing a tag that claims a lower class than the diff shows;
  (3) rebuild only what the class requires.
- Snoopy's recommended shape: move the files, keep ONE small fleet manifest as the lock (which
  repo+ref provides each package). The manifest commit is the fleet contract version.

## Buf or roll our own (Snoopy, 2026-10-22)

- Buf's dependency resolution needs the Buf Schema Registry (hosted; private modules and
  self-hosting are paid). It does not resolve from arbitrary git repos. Adopting it for resolution
  adds a vendor in the build path for a problem tacplus already half-solves (git sources,
  multi-source).
- `buf breaking` is proven for WIRE compatibility classification and worth using as the first stage
  of the classifier. It cannot see `spec` semantics, so tacplus must add the annotation-aware stage.
- Recommendation: roll the resolver (small: fleet manifest -> fetch at tags -> tacplus sources), use
  `buf breaking` for the wire stage, add the annotation stage in tacplus.
- "Only the types it uses": independent of Buf. tacplus gathers the import closure anyway; a
  `--only-reachable` style option would generate just the closure of the service's own protos, so
  a utility gets its types and nothing else.

## Not before launch.

## Why it came back: distributing a service (Jody, 2026-10-22)

Open-source doppelganger beside tacplus. Someone deploying it in their own cluster, with their own
kernel implementation, should be able to "point your operator at it and have your kernel built to
handle it." That is the proto-with-the-service model as a distribution format: a service module
is its protos plus its handler code, and a fleet's kernel, gateway and operator are compiled against
the set of installed modules, like a distribution is built from packages. The fleet manifest IS the
module list.

What it forces, beyond the resolver:
- **Open services can only import open contracts.** doppelganger today imports only
  `sys/spec.proto` (tacplus's own), so it is already clean. The moment a module needs the kernel's
  token mint or config surface, that contract must exist as a versioned, public proto package, i.e.
  the "what the upstream must provide" boundary from the deck becomes a published package, not a
  description.
- **Service ids are fleet-assigned, not module-assigned.** Today each proto repo carries a
  `registry.yaml` with its own ids and an `id_min` floor. Two third-party modules cannot coordinate
  floors, so the fleet manifest allocates ids at resolve time and the module ships without them.
- **A module declares what it needs from the platform** (state store, token mint, eventbus, which
  traits) so the operator can refuse a fleet that lacks it, rather than failing at generation.

## Patch releases and labels (Jody, 2026-10-22)

A PATCH release may come from a bug fix or from a compatible proto change; that is not a problem.
Release labels say what the release is for. The label is the input to the impact set: `fix` means
only the service's own code changed and no importer rebuilds; `proto` means a patch-class contract
change and every importer on that line rebuilds to float onto it; both labels are treated as
`proto`. The classifier is what makes a `proto` label trustworthy (it must confirm patch class).

## The pin rule (Jody, 2026-10-22)

**We always run the latest PATCH of a MAJOR.MINOR, subject to canary release validation. We never
pin to a patch.** A new patch is not a decision anyone makes per consumer; it is a canary step the
fleet takes (see [multi-cluster](multi-cluster.md): canary and stair-stepping, the Admiral deciding
each step). A pin to a patch is a defect in the pipeline, not a configuration option. This is the
same rule tacplus's own consumers follow (`tacplus/docs/VERSIONING.md`), now stated for protos and
services alike.
