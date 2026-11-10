Suggestion: Kernel's Hazelcast port to kernel's pods only, before operator#24 lands
Type: Security
Summary: Kernel binds Hazelcast on 0.0.0.0:5001, whose member protocol is unauthenticated and which also accepts Hazelcast clients. operator#24 (landing) removes every NetworkPolicy but kernel-raft, which leaves 5001 open to the whole deployment. Admit 5001 only from app: kernel in that policy, name it in the decision's exception beside Raft, and refuse the client protocol in kernel
Why: Jody's 2026-11-07 rule (decisions/s2s-never-crosses-tenants-2026-10-31.md) is that bearer tokens isolate and NetworkPolicies go, with one exception: a protocol with no authentication of its own, 'kernel's Raft port today'. Kernel's Hazelcast port is a second such protocol, and kernel's state (tokens, tenants, the identity graph) lives in that Hazelcast cluster. A pod that reaches 5001 can join as a member or connect as a client and read or write it, and drive the shaded jackson-core 3.1.2 with a HazelcastJsonValue (kernel PROPOSALS, #44). operator#24's report flagged it as 'a choice for you to check': it kept 5001 open deployment-wide and put the fix in PROPOSALS ('one line in the same policy'). No issue holds it, and #24 is looper:landing now, so the merge that removes the other policies will ship this gap with it. Stratum already moved its Hazelcast behind the authenticated member port (tacplus#212, stratum#17); kernel has not. Not checked: whether the deployment's tenant namespaces count as 'the deployment' in kernel-raft's rule.

## Evidence
- https://github.com/tacplus-dev/operator/issues/24 (bug, approved, priority:high, looper:landing: removes every NetworkPolicy but kernel-raft; the report: 'kernel's Hazelcast port (5001) keeps the deployment-wide reach it had ... as unauthenticated as Raft, and I found no sign that kernel has moved it behind the member port')
- https://github.com/tacplus-dev/operator/blob/main/PROPOSALS.md ('Confine kernel's Hazelcast member port to kernel's pods too', 2026-11-03, #20: open to the whole deployment, admitting it only from app: kernel is one line, held back until verified against a live kernel cluster)
- https://github.com/tacplus-dev/kernel/blob/main/PROPOSALS.md ('Refuse the Hazelcast client protocol on kernel's member port', 2026-11-04, #44: open-source Hazelcast accepts clients on 5001 beside members, so anything reaching the port can put a HazelcastJsonValue and drive the shaded jackson-core 3.1.2)
- https://github.com/tacplus-dev/operator/blob/main/pkg/bootstrap/deploy.go (line 512: kernel's HazelcastListener on 0.0.0.0, DefaultHazelcastListenerPort)
- https://github.com/tacplus-dev/roadmap/blob/main/decisions/s2s-never-crosses-tenants-2026-10-31.md (2026-11-07: no NetworkPolicies for isolation; the only exception a protocol with no authentication of its own, kernel's Raft port today; commit 99986a5)

## Cost
Hours in operator: kernel-raft's ingress rule for 5001 admits only app: kernel pods, and a chart test that another deployment's pod and a same-deployment non-kernel pod are both refused 5001 (the #20 verification on a live kernel cluster, which the hub can do on dev). A day in kernel: an advanced network config with a member endpoint and no client endpoint, and a test that a HazelcastClient is refused (#44's proposal). A line in the decision naming 5001 beside Raft. Later, kernel's member traffic behind the tacplus member port, like stratum, which removes the exception.

## Touches
decisions/s2s-never-crosses-tenants-2026-10-31.md (bearer tokens isolate; the unauthenticated-protocol exception), operator#24, kernel#42 (Raft behind the member port: the same move for Hazelcast), tacplus#212 and stratum#17 (stratum's Hazelcast behind the member port), tacplus#465 (the member port's expired-token test, red on main)

## Reuse considered
The kernel-raft NetworkPolicy already exists and selects kernel's pods; the change is its 5001 source. Stratum's HazelcastNetworkPolicy (removed by #24) was exactly this rule for stratum. kernel#42 moves Raft, not Hazelcast, behind the member port, so it doesn't cover 5001. The tacplus member port (tacplus#212) is the lasting fix, and it already carries stratum's members. Not proposed: any NetworkPolicy beyond the unauthenticated-protocol exception Jody kept.

## Status
open
Rank: 1
Filed: 2026-11-10 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17872762621)
