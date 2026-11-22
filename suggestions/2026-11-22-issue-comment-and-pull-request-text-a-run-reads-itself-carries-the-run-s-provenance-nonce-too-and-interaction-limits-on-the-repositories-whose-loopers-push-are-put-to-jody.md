Suggestion: Issue, comment and pull-request text a run reads itself carries the run's provenance nonce too, and interaction limits on the repositories whose loopers push are put to Jody
Type: Security
Summary: looper#40 (2026-10-31) wraps the issue text the prompt carries in per-run nonce blocks, so only a member's text directs a run; but a comment posted during the run, a linked issue, a blocker or a pull request the model reads itself with gh issue view or gh pr view arrives unwrapped, with only the prompt's rule about authors between a stranger's text and a run that holds a push token (looper's PROPOSALS 2026-10-31; STATE.md's Known gaps), and no issue holds the follow-up. Every repository is public and a run reads other issues and pull requests as a matter of course (Blocked-by lines, backport PRs, the hub's mid-run comments it acts on). Give the model repo-looper --issue-text N and --pr-text N that print the same nonce blocks with each author's association from the run's .nonce, have the prompt send every later read through them, and put GitHub's interaction limits (collaborators only, renewable for six months) to Jody for the repositories whose loopers push
Why: The prompt-provenance rule is the one boundary between anyone on the internet and a model with write access to branches, issues and a model budget; today it holds for the first read only. The gates and the manager bound what a hijacked run can merge, not what it can push, open, comment or spend. No new evidence of an attack: the gap is in the record and unfiled for three weeks. Also serves Operability: a run's later reads carry the same association the prompt's rule needs, instead of each model looking it up

## Evidence
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('Wrapped issue text when the model reads it again', 2026-10-31, #40: 'a comment posted during the run, a linked issue or a pull request the model reads itself (gh issue view) comes unwrapped: there only the prompt's rule (its author's association) stands between it and the run'; 'Limit who may comment on the public repositories', 2026-10-31: 'it is Jody's call, maybe only on the repositories whose loopers push')
- https://github.com/tacplus-dev/looper/blob/main/STATE.md (Known gaps: text the model reads itself during a run (a newer comment, a pull request) is not wrapped: only the prompt's rule on authors guards it (#40, PROPOSALS.md))
- https://github.com/tacplus-dev/looper/issues/40 (closed 2026-10-31: prompt provenance, nonce blocks, only members' text directs a run; the follow-up for a run's own reads has no issue)
- https://github.com/tacplus-dev/cloud/issues/88 (2026-11-21: the run read the hub's comments posted during the run, 'Land it' at 12:31Z and the test fix at 14:31Z, and acted on them: the path exists and is used, and carries anyone's comment the same way)
- https://github.com/tacplus-dev/looper/blob/main/README.md (who directs a run, tacplus-dev/looper#40: an OWNER or MEMBER, or a COLLABORATOR, CONTRIBUTOR or NONE with write access; anyone else's text is data, which each model must establish itself for anything it reads after the prompt)

## Cost
A day in repo-looper: two subcommands that reuse the prompt's block writer, the run's .nonce and the author lookup it already makes (gh api collaborators permission), a line in the prompt's rules, and a test with a NONE author's comment coming out marked as data. Jody's decision on interaction limits is minutes in the organization's settings, renewed every six months; it also stops outside bug reports on those repositories, which is why it is a decision and not a default

## Touches
looper#40 (prompt provenance), decisions/looper-identity-panel-2026-10-29 (what a run's identity may do), the rule in every looper prompt that only members direct a run, accepted suggestions 2026-11-03 (each model as its own user) and 2026-11-04 (only the manager writes main: the bound on what a hijacked run reaches), decisions/throughput-first-2026-11-11.md (a hijacked run spends the budget the merge rate needs)

## Reuse considered
looper#40's nonce blocks and author lookup exist for the prompt; the subcommands print them for a later read, nothing new is designed. GitHub's interaction limits are a setting, no code. A webhook or App filter that rewrites comments would be more machinery than the gap needs. The gates (a model-written change lands only on green, by the manager) are why this is rank 2: they bound the merge, not the push, the comment or the spend

## Status
open
Rank: 2
Filed: 2026-11-22 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18256297468)
