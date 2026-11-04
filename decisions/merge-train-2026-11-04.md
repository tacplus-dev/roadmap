# Managers land in batches: a merge train (Jody, 2026-11-04)

Amends decisions/looper-job-per-issue-2026-11-03.md (the manager alone merges, on green gates).

Jody, 2026-11-04: "I think changes and builds are different things. If 5 loopers are working in parallel then I think 5
builds is a waste. It's better to merge and the fix build failures." The hub proposed a merge train, which keeps main
green at about one build per batch; Jody: "Ok let's do merge train."

## Decided
- **A repository's manager lands its queued branches as a batch.** It merges every ready branch onto a candidate ref
  (`staging`, never main), in queue order, and runs **one build** (a build job on the spot pool) for the candidate.
- **Green:** main fast-forwards to the candidate; every change in the batch has landed, each with its verdict.
- **Red:** the manager bisects the batch (builds halves) to find the change that breaks it, sends that change back to
  its report (the issue gets the failure and stays approved), and lands the rest as a new batch.
- **main stays green:** nothing reaches main without a green build of exactly that commit. Releases are cut from
  green commits.
- **Reports don't wait for builds,** and don't run them: a report queues its branch and ends; its verdict comes from
  the train.
- **Node pools (Jody, same day):** normal 3 small on-demand Arm nodes (deployments and loopers, never builds); the
  build pool spot, 0 to 3 for now (to become 1, raised only on Jody's word), builds sharing nodes.
