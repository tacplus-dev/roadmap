Suggestion: The nightly scan files every blocking or HIGH finding no open issue names, not only the ones new since the previous run, which a land's build can make old
Type: Security
Summary: The nightly scan files only the findings its report marks new, and new is measured against whichever run published last, a land's or a --build's included, from which no scan files. So gateway's picomatch 4.0.7 ReDoS (GHSA-fc5q-39xc-vjpf, HIGH, webapp/package-lock.json) sat new: false in findings.json and off summary.md until the scan's own by-hand osv-scanner run found it (gateway#43, 2026-11-18), and chrono's RUSTSEC-2026-0543 went the same way in gateway (#31) and mailbox (#22), first seen by a land build of 2026-11-17. Have looper-scan count new against the last nightly of main, list the standing HIGH and CRITICAL findings neither baselined nor excepted in the summary, and have the scan file any blocking or HIGH/CRITICAL finding whose id no open issue of the repository names
Why: The floor's findings are only as good as what reaches an issue, and two repositories hit this gap independently on 2026-11-18 and found it by chance. The open suggestion of 2026-11-15 changes which report is latest (main's builds only), not the rule that a finding is filed once, when new: a batch's green build is a build of main's next commit, publishes first, and no scan files from it.

## Evidence
- https://github.com/tacplus-dev/gateway/blob/main/PROPOSALS.md (The security report should list HIGH and CRITICAL findings that are no longer new, 2026-11-18, #43: 'once it has been seen in one run it drops out of both lists and is visible only in findings.json. That is how osv-scanner's GHSA-fc5q-39xc-vjpf (picomatch 4.0.7, HIGH ...) went unfiled'; 'A finding first seen in a run that no scan files from (a land's or a --build's report) is already new: false in the next nightly')
- https://github.com/tacplus-dev/gateway/issues/43 (HIGH, 'Found by the nightly scan of 2026-11-18 ... No open or closed gateway issue holds it', found by the model's own osv-scanner run, not the report) and https://github.com/tacplus-dev/gateway/issues/31 (chrono, the same)
- https://github.com/tacplus-dev/mailbox/blob/main/PROPOSALS.md (A finding first seen in a land build should still be filed, 2026-11-18, #22: 'new is measured against the previous run of any kind, lands included ... Tonight's nightly then reported it new: false, so by the rule it went unfiled, and no mailbox issue held it until #22')
- https://github.com/tacplus-dev/mailbox/issues/22 ('They first appeared in a run after the advisory was published, probably the 2026-11-17 land build of #20. No scan files from a land build')
- https://github.com/tacplus-dev/looper/blob/main/STATE.md (Security checks: 'the new findings since the previous run (the build project's generic package security-<repo>), which the nightly agent scan reads'; since #180 each build's findings are published as its commit's too, so the last nightly's of main's head is there to compare with)

## Cost
A day in looper: looper-scan takes its base for new from the last nightly's package of main (security-<repo>/<main's head>, published since #180) rather than latest; a 'Standing HIGH/CRITICAL, not baselined, not excepted' section in summary.md; repo-looper's scan item 1 reads every blocking or HIGH/CRITICAL finding and files the ones no open issue names by id, one issue per cause (#129). Tests with the stub scanners: a finding first seen by a land build is filed by the next nightly.

## Touches
looper#49 (the deterministic floor), #129 (one issue per cause, blocked when the fix is upstream), #146 (the baseline keeps MEDIUM and below out), #180 (each build's findings as its commit's), PRODUCTION.md's security bar, the issue flow (the scans file bugs), suggestion 2026-11-15 (latest from main's builds only: complementary, not the same rule)

## Reuse considered
#180 already publishes each commit's findings, so the last nightly's package of main's head exists to compare with; the scan's prompt already searches the repository's issues by id before filing (gateway#43's 'No open or closed gateway issue holds it'), so 'no open issue names it' is that search applied to standing findings too; security-baseline.json keeps the list to HIGH, CRITICAL and blocking. looper#210 (filed tonight, a stale latest when a nightly publishes nothing) is the other way the same latest misleads the scan, and reading main's head's package fixes both.

## Status
accepted (the hub, on Jody's standing authority, 2026-11-21): tacplus-dev/looper#211
Rank: 2
Filed: 2026-11-21 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18244259587)
