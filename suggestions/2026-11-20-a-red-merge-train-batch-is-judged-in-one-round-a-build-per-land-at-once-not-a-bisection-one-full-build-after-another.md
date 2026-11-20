Suggestion: A red merge-train batch is judged in one round: a build per land at once, not a bisection one full build after another
Type: Performance
Summary: tacplus's manager merged 0 of 23 queued lands tonight: its one batch of 20 was red, bisected, and after 4 full builds (p50 55.9 min, one queued 34.6 min) its job died mid-bisection and the backlog stood at 23, over its threshold a 7th night. The reds come from the pod's OOM (cloud#82) and timing flakes (#778, #539, #627, #870), so a bisection on prefixes (about 1 + 2*log2(20), 9 builds, one after another) is the wrong search while the spot pool was full for 0 of 1385 pods. On a red batch, have the manager build each land's candidate (rebased on main; the pre-land set when gates.yaml names one, else full) at once, send back the red ones with their own failure and stage the greens as one batch; and approve tacplus#892, whose pre-land set stops most unit-test reds before a batch
Why: Jody: 'much of this can be done in parallel'. The train's economy, one build per green batch, is right for green and wrong for red: a serial bisection on an hour-long full build costs the night, and tonight it cost all 23. The parallel round spends build slots the pool had (pool_full 0, pods_pending 0) and ends with every land judged. Also serves Operability: a red land is told its own failure, not the batch's.

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/18215309726 (recap 2026-11-20, managers.tacplus: batches 1, batch_size 20, bisections 1, builds 4 at p50 55.9 min (max 58.7), build queue max 34.6 min, merged 0, lands 0; concurrency.repos.tacplus.backlog p90 23, max 23; manager/tacplus/backlog over its threshold of 3 on 7 of 7 nights; capacity.pool_full: full 0 of 1385 pods, pending 0)
- https://gitlab.com/tacplus-dev/looper/-/jobs/18209250868 (tacplus's manager job of the night: failed no_updates_running at 09:32Z, mid-bisection, its lands left queued)
- https://github.com/tacplus-dev/cloud/issues/82 (2026-11-20, approved, priority:high: 'three tacplus merge-train builds with different lands failed the same two ways ... Each red batch is then bisected or sent back, so innocent lands pay ~1 h each'; the 12Gi request merged, waiting on cloud's apply)
- https://github.com/tacplus-dev/tacplus/issues/778 (c2d-server: four legs on one socket path, a Java self-test hanging with no deadline; three red full builds of its fix in a row, 5209 to 6381 s each, 'The verdict does not name the failing gate')
- https://github.com/tacplus-dev/tacplus/issues/892 (2026-11-19, bug, not approved: gates.yaml names no pre-land set; 'three tacplus lands ... turned two of three batch builds red in test alone, each costing a ~60-min build plus a bisect')
- https://github.com/tacplus-dev/looper/blob/main/README.md (the merge train: 'Red: the batch is bisected on its prefixes ... about log2(n) builds', 'builds at most 1 + 2*log2(5) times'; the pre-land gate: the manager's checks are 'started all at once, a build each, then their verdicts')

## Cost
A day in repo-looper's manager: on a red batch of n > 1 lands, start a build per land at once (its candidate rebased onto main; the pre-land set when gates.yaml names one, else the set its diff calls for), collect the verdicts, send back the reds with their failure, stage the greens as one batch (one more build). Test: train-red-parallel (one bad of 5: 5 builds at once, 4 land in the next build). Build slots: n builds for an hour instead of about 2*log2(n) spread over the night; the pool's room bounds it, and the bisection stays for a pool with none (pool_full over 0 in the recap).

## Touches
decisions/merge-train-2026-11-04.md (one build per batch: 'changes and builds are different things'; its red path), decisions/throughput-first-2026-11-11.md, looper#86 (the train), looper#196 and tacplus#892 (the pre-land set), cloud#82 (the OOM reds), tacplus#812 (the 45-minute gate run), the 2026-11-19 suggestion 'Fewer full builds and more at once' (the pool's room)

## Reuse considered
The manager already fans out 'a build each, all at once' for the pre-land check of lands not green on its main (README, The pre-land gate): the red path reuses that fan-out with each land's candidate, and the verdict and send-back code the bisection uses. Considered: tacplus#892 alone (the pre-land set) stops the unit-test reds but not the OOM and timing ones; cloud#82 alone stops the OOM reds; neither shortens a red batch's search, and a 20-land batch with one flake still pays 9 serial builds.

## Advice
Recap: 2026-11-20 manager/tacplus/backlog
Cause: a red batch of 20 is bisected on its prefixes, one full build (about 56 min, plus queue) after another, while the build pool had room; tonight 4 builds, 0 merges, the manager job dead mid-bisection, backlog 23
Change: repo-looper --manage: on a red batch, a build per land at once (its candidate rebased on main; the pre-land set when gates.yaml names one, else the set its diff calls for), reds sent back with their failure, greens staged as one batch; tacplus's gates.yaml: approve #892's pre-land set
Gain: a red batch of 20 is resolved in about two builds' time (2 to 3 h with queue) instead of about 9 serial builds (the night): tonight's 23 lands would have had their verdicts by morning, about 20 land-hours a night at tacplus's current rate; with #892, most unit-test reds never reach a batch

## Status
accepted (Jody, 2026-11-20): tacplus-dev/looper#205
Rank: 3
Filed: 2026-11-20 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18215309726)
