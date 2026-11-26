# Summation: panel review of the agentic development flow

Design: `../agentic-dev-flow-2026-11-25.md`. Reviewers: Codex (gpt-6-astra), Grok (grok-4.7) and Gemini (gemini-4-argon), each independently (`raw-*.md`), then each on the other two (`rebuttal-*.md`). Written by the hub, 2026-11-25. Only what all three hold after the rebuttals is listed as agreed; the rest is listed as a disagreement, with each side.

## Verdict

The direction stands: loopers build for themselves, a change reaches main without a full build in its way, main is checked continuously, and only a green release build is deployed. No reviewer argued for keeping the merge train. All three say the design as written would stall at the target load (50 merge requests a day on the busiest project), in the places below, and that its recovery path (the revert agent) was the least thought through.

## Agreed by all three

1. **The slowest gate is still the speed limit, and it is the first thing to fix.** After every other change, a full check of main is as long as its slowest group (about 30 minutes, `current-generation`), and `pkg/**` still maps to everything. Each named this as the largest remaining bottleneck. Shard `current-generation` and split `pkg/**` into per-language sets before anything else.
2. **Capacity is real at the target load.** One full gate run per merge request does not fit one 8-CPU VM at 50 a day (100 to 267 core-hours of builds against 192 available, before retries; nine other repositories share 4 CPUs). "Nothing is queued" only moves the queue into the CPU. Bound the builds running on a VM, show the wait, cap builds and spend per issue, and measure before sizing.
3. **Do not require a rebase onto main's tip.** Main moves faster than a gate runs, so reports would rebuild forever. A looper builds on the last commit the rolling check found green (a `main-green` ref), and its evidence records that base.
4. **The merger is not an agent and does not resolve conflicts.** One conflict would block the only merger for a gate's length, and an agent without the issue's context can drop one side and still pass file-scoped gates. A conflict goes back to the report that wrote the change; the merger keeps merging the others. The merger is a small deterministic controller on a machine that runs no project code.
5. **The looper's VM can hold neither the credential that writes main nor the one that says "accepted".** Otherwise a report accepts itself. Build containers on the VM can also reach its metadata server and so its service account, which reads the model token today: a signing daemon on the same VM is not a boundary until they cannot.
6. **A looper's own build is feedback, not proof.** What the merger trusts must be a status the report cannot mint. What a release trusts is an independent rebuild, bound to the digest it publishes.
7. **No full build on a clean merge, inside a limit.** All three accept unbuilt clean merges, and all three reject them without a bound: limit how many merges (or how old) may sit on main unchecked, and stop merging while the rolling check is red, down, bisecting or stale.
8. **The rolling check runs on its own reserved capacity**, not on a looper's VM, not on spot, and not in the GitLab build project that refused jobs on 2026-11-24. It needs more than 8 CPUs to run the gate groups side by side.
9. **Recovery is a fixed procedure with an owner, not an open task for an agent.** On red: stop merging; run it again to tell a flake from a break; revert only when one merge is isolated and the reverted tree is shown to build; otherwise open one "fix main" issue for one looper. A flaky gate is common, not rare (2% per group over 19 groups is a false red on about a third of runs). A red that no shipped change caused (a nightly image re-pin's findings) must not revert or freeze anything.
10. **A reverted change needs care to come back.** Its commits stay in main's history, so merging the same branch again brings back only the later fix, not the feature. It must return as a revert of the revert or as new commits, and the reverted tree must not merge again unchanged.
11. **Gate scoping fails closed.** A path no set names, and any change to the gate map itself, runs everything. Derive the map from the build's dependency graph, including generator-to-generated edges; record every miss the rolling check finds; never let a red check rewrite the map by itself.
12. **Releases publish exactly the checked commit.** A release job that resolves `main` when it starts can publish commits the check never built. Release branches need a green status of their own. An older build must not be able to deploy after a newer one. A red release gets an owner (an issue filed automatically), not "nobody".
13. **One run per issue, and every hand-off has a watchdog.** A preempted report and a kicker can start two runs of one issue; merges, reverts and deploys need claims that survive a restart; a VM that is up must notice an issue reopened after it started; an idle VM must wake for an open merge request, not only for a ready issue.
14. **Operations the design did not cover:** the cache disk's size, eviction and poisoning (a disk outlives its VM, and a detached disk is not attachable for 30 seconds or more); build containers left behind by a report that died; whether 25 seconds checkpoints ten checkouts on one git store.

## Disagreements left

| # | Question | Codex | Grok | Gemini | Hub's recommendation |
|---|---|---|---|---|---|
| D1 | A quick compile of the merged tree before a clean merge | Measure one in shadow before requiring it | No: it is not quick across crates, it serializes the VM, and it misses breaks that compile | Yes, required, under 2 minutes | Codex's: run it in shadow and decide on the numbers |
| D2 | How many unchecked merges may sit on main | A limit on count and age, measured | Only what one in-flight check covers | At most 4 | Start at 4 or one gate's age, whichever is first; tune from the data |
| D3 | What proves a revert before merging reopens | The full gates | The reverted tree's build | A compile | The failed group green on the reverted tree lands the revert; merging reopens when the next full check is green |
| D4 | When consumers of tacplus are checked | Before promotion, through a pinned compatibility manifest | At deploy only: never block tacplus main on a service | Before merge | Grok's: check consumers against the release's digest before deploy; do not put them in tacplus's merge path |
| D5 | The panel's time limit (later) | A timeout is not an approval; fall back and adjudicate | 15 minutes, then proceed; two rejections on security or scope block | 90 seconds, two of three approve, a timeout abstains | Decide when the panel is built; all three agree it runs in parallel, off the looper's VM, with a budget |

## What changes for Jody's decisions of 2026-11-25

- **"An agentic process does the merges, so it can deal with conflicts"** becomes: the report that wrote the change deals with its conflict (it is the agent with the context), and the merger itself is a small fixed program. All three reviewers.
- **A new reserved machine** for the rolling check, per heavy project, plus one for the rest. This is capacity, so it is Jody's to approve.
- **The order of work** starts with the gates (item 1), not with merge requests.
