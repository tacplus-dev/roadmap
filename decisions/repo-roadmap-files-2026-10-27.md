# Every repo carries the same four planning files (DECIDED 2026-10-27)

Jody, 2026-10-27. An **agent** is an interactive session acting at Jody's request. A **looper** is the automated
per-repo worker. They have different rights.

| File | Owner | How it changes | Holds |
|---|---|---|---|
| `ROADMAP.md` | Jody, through an agent | Edited only at Jody's request; **loopers never change it** | Where the repo is going: current line, next (linked issues), later, not doing |
| `PROPOSALS.md` | Loopers | Loopers edit their own file | What loopers think should come next: follow-ups, debt, ideas, each with a reason and links. Jody promotes items into ROADMAP.md (by asking an agent) |
| `STATE.md` | Loopers and agents | Overwritten, always current | Version and floor, what works, released and deployed where, known gaps, open blockers |
| `HISTORY.md` | Loopers and agents | **Append-only** | One entry per closed issue, decision, **release and deploy**: date, who, one line, links. Not a copy of git log |

Templates: `templates/repo/` in this repository. Every repo starts from them and keeps their headings.

**Enforced now** (CI check in every repo): the four files exist, and `HISTORY.md` changes only by lines added at
its end. Headings and shape are not checked until the template settles.

**HISTORY entries for closed issues are written by `repo-looper` itself** (mechanically, same format everywhere),
not by the model.

**Decisions:** every new decision goes in this repository's `decisions/`. Existing per-repo `decisions/` folders
(e.g. kernel's) stay where they are, frozen, because code and docs cite them; `decisions/INDEX.md` here lists
them. A repo's HISTORY links the decision; it never copies it.

**Platform level:** this repository carries the same four files for things that span repos or don't exist yet.
`items/` holds the not-yet-started work; an item moves into a repo's ROADMAP.md when that repo exists.
