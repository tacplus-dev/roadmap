# SeaBeam realign brief: adopting tacplus-dev's work

- **For:** SeaBeam's loopers and the people steering them.
- **From:** tacplus-dev, 2026-10-29, at Jody's request ("have seabeam restart its loopers and start work based on our new work and
  design").
- **Status:** phase 1 may start now. Phase 2 waits for the marker below.

## How to read this

**tacplus-dev is the source of truth** for everything listed here. SeaBeam adopts it; it does not re-design it. If something here
doesn't fit SeaBeam, open an issue on the tacplus-dev repository it belongs to, rather than diverging in SeaBeam.

Before you open any issue, read the decisions in `tacplus-dev/roadmap/decisions/` (`INDEX.md` first) and the target repository's
`ROADMAP.md`. That is the issue flow, `decisions/issue-flow-2026-10-27.md`.

## The one rule for every realign commit

Each SeaBeam commit that ports tacplus-dev work carries a trailer, one line per source commit:

```
Ported-from: tacplus-dev/<repo>@<sha>
```

tacplus-dev's hourly port scan, which copies SeaBeam changes into tacplus-dev, skips commits with this trailer (tacplus-dev/looper#23).
**Without it, your port is ported back and the two sides bounce changes forever.** Genuine SeaBeam-only changes carry no trailer
and are picked up as before.

**SeaBeam's loopers do not restart until looper#23 is on main.**

## Phase 1: adopt now (released or settled)

| Area | Adopt | Pin / source |
|---|---|---|
| Generator | tacplus **2.5** line (records write surface, route fields, redo contract). 2.4 for services not ready for 2.5. | `v2.5.0` (2.5.1 and 2.4.2 patches imminent); pin by MAJOR.MINOR, never by commit |
| Identity | XIDs (random 128-bit, never reused), devices on the XID (no `/people/<pid>` principals), the session credential and per-tenant exchange, retiring the `linked` claim, alg inference within a key's family and refusing `alg: none` | kernel `c630f29`, oauth `e53f7f7`; `decisions/identity-session-credential-2026-10-25.md` |
| Tenants | fixed-size tid per tenant; tenant-owned calls routed by the request's target-tenant field (tid), never by the caller's token; tenant-owned service SPIFFE ID `spiffe://<td>/tenants/<tid>/services/<svc>`; servers refuse cross-tenant calls whoever the caller is | `decisions/tenant-routing-2026-10-26.md` (all addenda), kernel#23 |
| Tenant deletion | delete all the tenant's data; no new AT/ST for it; XIDs kept and aged out after about a year without sign-in | `decisions/tenant-deletion-2026-10-27.md` |
| Process | loopers on GitLab runners only, never on a laptop; the four roadmap files (ROADMAP.md is the owner's, PROPOSALS.md the loopers', STATE.md current, HISTORY.md append-only); bug → fix plus patch releases, feature → design, approval, ROADMAP first | `decisions/ci-on-gitlab-2026-10-27.md`, `repo-roadmap-files-2026-10-27.md`, `issue-flow-2026-10-27.md`, `deterministic-builds-2026-10-27.md` |
| Language | Rust by default; Go or Java only with a written reason | (Jody, 2026-10-27) |

## Phase 2: hold until the marker

**Do not port these yet.** They changed on 2026-10-28/29, or haven't run in a fleet.

| Area | Why it waits | Track |
|---|---|---|
| Per-tenant Stratum, redo | landed hours ago; a cross-tenant write in the record plane client; Hazelcast member isolation unproven | stratum#2, redo#1, tacplus#207, stratum#17/#18 |
| Record writers on Stratum's write surface (no event-bus ingest) | waits on tacplus#207 | letsencrypt#2, doppelganger#7, coop#5, blobstore#6, mailbox#7, messaging#7 |
| flock and the people table | design only; nothing built | `items/flock-design-2026-10-28.md`, tacplus#206, people#1 |
| Build image (no tacplus image; `build:X.Y`) | in progress | tacplus#193 |
| Per-fleet trust domain | in progress | fleet-seatrials#16, operator#14 |
| Cross-cluster S2S clients | not designed | tacplus#204 |
| Kernel's Hazelcast "enterprise" override | being removed | kernel#24 (**do not copy** `hazelcast.internal.override.enterprise`) |

**Marker for phase 2:** tacplus-dev/fleet-seatrials#3 closes and the new Stratum runs on seatrials with the isolation tests passing.
tacplus-dev will update this brief then.

## Platforms (Jody, 2026-10-29)
- **SeaBeam is arm64 only, permanently.** Only we run it, and we run arm64, so SeaBeam needs **no amd64 builds at all**. Don't add an
  amd64 setting.
- tacplus.dev differs: its amd64 builds stay in place, disabled, to be re-enabled for open-source users once its change rate settles.

## What not to do

- Don't bring anything over from SeaBeam into tacplus-dev by hand. The port scan does that, by issue.
- Don't pin services by commit. Pin by MAJOR.MINOR, which resolves to the latest patch.
- Don't write gRPC clients, server wrappers, token handling or state clients by hand. Use what tacplus generates.
- Don't restart letsencrypt on any cluster to test anything. Every restart orders a new certificate, and the budget is nearly spent.
