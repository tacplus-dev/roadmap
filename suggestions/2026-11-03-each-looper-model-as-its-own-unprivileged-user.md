Suggestion: Each looper model as its own unprivileged user
Type: Security
Summary: Run every child's model as a non-root user of its own, with docker reached only through looper-docker as a broker, so models can't read the parent's credentials or signal the lander and other children
Why: Every model runs as root in the job's one PID namespace, beside the parent, the lander and the other children. It can read /proc/<parent>/environ (the GitLab credentials MODEL_WITHHELD keeps out of its environment) and kill any process. On 2026-11-03 this cut tacplus#222's land twice (looper#61). looper-pkill and prompt rule 4 are guards, and the README says they are not boundaries. Since models act on text written by people outside the org (looper#40), an injected instruction meets no boundary. Also serves Resiliency: no model can stop another child's land.

## Evidence
- https://github.com/tacplus-dev/looper/issues/61 (closed 2026-11-03: tacplus#256's and tacplus#283's models stopped tacplus#222's land under way by pattern; fixed by a guard, looper-pkill)
- https://github.com/tacplus-dev/looper/blob/main/README.md (the parent's plan: 'the job is one PID namespace and every model runs as root beside the lander ... It is a guard, not a boundary (kill <pid> reaches any process)')
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('The model reads its parent's environment', 2026-10-30, #25; 'The lander's processes out of the models' reach', 2026-11-03, #61; 'A child's memory as one cgroup', #41)

## Cost
Days. The hard part is docker: the socket is root-equivalent, so the model's user must not own it. looper-docker would become the only path to it (a sudo rule for that wrapper alone, refusing --privileged, host mounts outside the child's worktree, and --pid/--net=host). Each child also needs its worktree, caches and git credential owned by its user, and repo-looper's tests need updating. The per-child cgroup (#41's proposal) fits in the same change.

## Touches
looper#40 (prompt provenance), looper#25 (credentials withheld from the model), looper#41 (child limits), looper#61 (the lander out of reach), the issue-flow rule that broken work never lands

## Reuse considered
Considered first: (1) unshare --pid per model (PROPOSALS, #61) needs CAP_SYS_ADMIN in the job container, which widens the job's own privilege, and it doesn't hide the parent's environment from a root model by itself. (2) looper#28's think/commit split already keeps the identity out of the model's job, but only for kernel's pilot. (3) looper-docker and looper-pkill exist and are the natural broker and helper; the proposal makes looper-docker the boundary rather than a limit-setter. A separate user is the smallest step that makes both guards hold.

## Status
accepted
Rank: 2
Filed: 2026-11-03 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17692416595)
Decided: 2026-11-03 by Jody: Accept and next (Jody, 2026-11-03).
