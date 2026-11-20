Suggestion: The deprecated tacplus/tacplus image is still pushed by every 2.1 to 2.5 patch, as root, against VERSIONING.md: stop the lines pushing it
Type: Security
Summary: docs/VERSIONING.md says 'ghcr.io/tacplus-dev/tacplus/tacplus is deprecated: its tags up to the 2.5 line stay as they are, and it is never pushed again', and main's images.yml says the same; yet release/2.1 to 2.5's images.yml still carry the tacplus job (outputs name=${NS}/tacplus, push=true), so the 2026-11-15 patches pushed tacplus:2.1.25, 2.2.22, 2.3.24, 2.4.24 and 2.5.25, built from a root Dockerfile with no USER (tacplus#580 is main's fix; main has no root Dockerfile since #193), an image no report scans and no fleet pins. Backport #193's removal of the tacplus job and the root Dockerfile to the five lines, or write in VERSIONING.md that the lines keep pushing it and put it in the nightly sweep
Why: A public image under the platform's name that the docs say is retired keeps getting new tags on every patch, runs as root, and is in no security report and no fleet lock: beside the images bar A pins by digest, nothing should look current that nobody maintains. tacplus's PROPOSALS has asked since 2026-11-02 which it is to be. Also serves Documentation (VERSIONING.md says what the lines do) and Cost (five image builds a patch for nothing).

## Evidence
- https://github.com/orgs/tacplus-dev/packages/container/package/tacplus%2Ftacplus (versions 2.1.25, 2.5.25, 2.2.22, 2.3.24 and 2.4.24 created 2026-11-15T13:19Z to 13:24Z, with the line patches of that day; read 2026-11-20)
- https://github.com/tacplus-dev/tacplus/blob/main/docs/VERSIONING.md ('is deprecated: its tags up to the 2.5 line stay as they are, and it is never pushed again')
- https://github.com/tacplus-dev/tacplus/blob/release/2.5/.github/workflows/images.yml (line 293: outputs type=image,name=${{ env.NS }}/tacplus,push-by-digest=true,push=true; the same line on release/2.1; main's images.yml line 8: 'The old tacplus/tacplus tags stay, deprecated, and are never pushed again')
- https://github.com/tacplus-dev/tacplus/blob/main/PROPOSALS.md (2026-11-02: 'docs/VERSIONING.md on main says tacplus/tacplus is never pushed again ... but each patch tag runs its release branch's pre-#193 images.yml, which builds and pushes it ... Either say in VERSIONING.md that the 2.0-2.5 lines keep pushing it until they leave support, or stop pushing it from the lines')
- https://github.com/tacplus-dev/tacplus/issues/580 (approved, landing: 'the root Dockerfile, of the deprecated tacplus image, runs as root'; release/2.5's Dockerfile: FROM the base image, no USER)
- https://github.com/tacplus-dev/tacplus/issues/193 (closed 2026-10-29: no tacplus image from 2.6, build:X.Y per line; the lines' workflows were left as they were)

## Cost
Hours: one backport PR per line (2.1 to 2.5) that removes the tacplus job from images.yml and the root Dockerfile, merged through the hub's backport step (the 2026-11-18 suggestion asks the manager to do that); or, if the lines keep it, a VERSIONING.md paragraph, USER in each line's Dockerfile (#580's backports) and the image in tacplus's nightly sweep.

## Touches
decisions/tacplus-images-2026-10-27.md (base and build only; 'no tacplus image', #193), decisions/deterministic-builds-2026-10-27.md, PRODUCTION.md's supply-chain bar (no open HIGH or CRITICAL in shipped images), docs/VERSIONING.md (what each line publishes), tacplus#580 and #283 (the lines' images)

## Reuse considered
#193's change on main is the patch, and the backport flow (one PR per supported line, docs/VERSIONING.md) carries it; #283's line-rebuild job already runs each line's images.yml, so the removal takes effect at the next patch. Considered: keeping the job and only adding USER on each line (#580's backports), which keeps publishing an image the docs retired and nothing scans.

## Status
open
Rank: 4
Filed: 2026-11-20 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18215309726)
