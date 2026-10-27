# Nightly security sweep follow-ups (tacplus)

- **Status:** active
- **Owner:** tacplus fix looper
- **Opened:** 2026-10-20
- **Records:** tacplus issues #62 (closed, the sweep), #90 (module dependency bumps), #91 (image
  findings); `.github/workflows/nightly-security.yml`

## What

The sweep runs nightly on main and every release line. Its first run reported 83 known module
vulnerabilities and HIGH image findings; both are fix requests the looper works, with backports to
release/2.0. Done when both jobs are green on main and release/2.0.
