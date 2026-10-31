# Issue flow: bug or feature (DECIDED 2026-10-27)

Jody, 2026-10-27. Every issue is first classified:

- **Bug** (the code does not do what it already promises: a defect, a vulnerability, a broken gate): it is fixed, and
  a **new patch release** ships on every supported release line that has it (docs/VERSIONING.md: main plus a backport
  to each `release/MAJOR.MINOR` whose `.0` is under twelve months old). No design or roadmap step.
- **Feature** (new behaviour, new surface, a changed contract): it needs **design work and Jody's approval**, and it
  **goes into the repo's ROADMAP.md** (written by an agent at Jody's request) before it is built. It ships in a minor
  (or a major if it breaks something).

How this maps onto labels and files:
- Labels `bug` and `feature` say which. `approved` on a **bug** means "fix it". `approved` on a **feature** means the design is
  approved, **and** the feature is listed in the repo's ROADMAP.md (or the platform's, for cross-repo work).
- A looper that finds a feature it was not asked for writes it to PROPOSALS.md (and may open an issue labelled `feature`,
  never `approved`). A design question on an approved feature is a comment with a recommendation plus `needs-decision`.
- A looper never builds a feature that is not on a ROADMAP.md.

## Before opening an issue: consult the roadmap (Jody, 2026-10-27)

Anyone opening an issue (Jody, an agent, a looper, the nightly or port scan) first reads the repo's ROADMAP.md and
the platform's (tacplus-dev/roadmap ROADMAP.md and `items/`):
- **The feature is already on a roadmap:** don't open a new feature issue. Open one asking for **that roadmap entry to
  be prioritized**, linking it and saying why now.
- **It isn't:** open it as a `feature` (it then needs design, approval and a ROADMAP.md entry) or as a `bug`.

## Addendum 2026-10-31: a new mechanism must argue against reuse (Jody)

Prompted by tacplus#212. Its custom TLS framing was specified straight from a stated mechanism. No alternative was weighed,
and on 10-31 it was replaced by gRPC with make-before-break stream rotation (#259). Jody: "Yes that is a good specific
mitigation."

A feature design that adds a **new wire format, protocol, transport, or security mechanism** must, before approval:
- **Include a "reuse what exists" alternative.** Say how the platform's existing stack would do it (generated gRPC, its
  TLS and token handling, existing stream rules), and argue specifically why that isn't enough. "It's a black box" or
  "it's different" is not an argument.
- **Check it against the platform's written contracts**: docs/API.md (the Client Send Contract, stream auth), the
  decisions here, and VERSIONING. List any rule it bends.
- **Go through a round-table if it touches security or identity**: independent analyses, a cross-read, a summary
  (the login-flow and looper-identity panels are the model). The summary goes on the issue before `approved`.

Start from the requirements, not from a mechanism someone has already named: write the requirements first, then the
options.
