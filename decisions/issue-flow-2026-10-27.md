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
