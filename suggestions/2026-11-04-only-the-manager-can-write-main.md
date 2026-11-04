Suggestion: Only the manager can write main
Type: Security
Summary: A ruleset on main in the 28 public repositories, with only a manager identity allowed to update it, and reports pushing branches with a token that is not Jody's, so 'nothing reaches main without green gates' holds against a model
Why: Since looper#70 (live 2026-11-04: the looper manager merged #71, #75 and #78) the decision is that the manager alone merges and nothing reaches main without green gates. But no repository's main is protected (branches API, 2026-11-04: protected false on tacplus, looper, kernel and cloud), and every report's model holds GH_TOKEN, which is Jody's and can write every repository. The only thing between a model and main is its clone's pre-push hook, which git push --no-verify or a contents API call skips. Models act on text from people outside the org (looper#40), so one injected instruction can push untested code straight to tacplus's main, and from there into every generated service. GitHub rulesets are free on public repositories (28 of the 30). Also serves Correctness: the manager's score in the recap (red main, reverts) means something only if it is the only writer.

## Evidence
- https://github.com/tacplus-dev/roadmap/blob/main/decisions/looper-job-per-issue-2026-11-03.md (decided: the manager is the only merger; the guardrail: nothing reaches main without green gates)
- https://api.github.com/repos/tacplus-dev/tacplus/branches/main (protected: false, read 2026-11-04; the same for looper, kernel and cloud; 28 of the org's 30 repositories are public, fleet-dev and fleet-seatrials private)
- https://github.com/tacplus-dev/looper/blob/main/README.md#credentials (a report's model's own environment holds GH_TOKEN; the job's user reaches everything; the identity pilot refuses 'a job that holds Jody's tokens', which every report job does) and the README's report section ('a push to main from its clone is refused (its pre-push hook)')
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('The landing guard is a hook', #32: --no-verify skips it, main's protection would make 'broken work never lands' hold; 'A verdict only the manager can give', #70: any model can post the manager's verdict comment; 'ROADMAP.md guard that a model cannot skip', #13)
- https://github.com/tacplus-dev/looper/issues/75 (2026-11-04: the manager now merges live, so the manager identity has a single place to go)

## Cost
A day or two. (1) A GitHub App (or bot user), e.g. tacplus-manager, with contents:write, used only by manager jobs, the scans' planning-file commits and looper-suggest's roadmap commit. (2) The reports' GH_TOKEN replaced with a token that may push branches, comment and label but is no bypass actor: a second App installation token or a bot's fine-grained PAT. (3) One org script that applies a ruleset to main on the 28 public repositories (restrict updates, block force-push and deletion), with the manager App and Jody's own account as bypass actors. Repository admins are not in the bypass list, so Jody's PAT in a model's hands doesn't bypass, and it is no longer in a model's hands anyway. fleet-dev and fleet-seatrials are private on the Free plan, where rulesets need GitHub Team: Jody's call, or keep reports from pushing there. Tests: a report's push to main and a contents-API write are refused, and the manager's merge passes.

## Touches
decisions/looper-job-per-issue-2026-11-03.md (the manager alone merges; green gates), decisions/looper-identity-panel-2026-10-29 (an identity per role), looper#28 (per-repository identity pilot, kernel only), looper#40 (prompt provenance), looper#68 (model users), looper#70, looper ROADMAP Later (a protected release line), the rule that broken work never lands

## Reuse considered
looper#28's think/commit split already keeps every identity out of the model's job, but only for kernel's pilot, and it is still waiting on the hub's setup. This is the org-wide floor until it rolls out, and it keeps working after. The pre-push hooks (#13, #70) stay as early, friendly refusals. A required status check (the build's tacplus-reader[bot] status, #71) could replace the push restriction once every repository builds in tacplus-dev/build, but only looper is pinned today (build.lock). GitLab-side tokens are already narrowed (#25, read_api bot), so this is GitHub's half of the same step.

## Status
open
Rank: 1
Filed: 2026-11-04 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17729413356)
