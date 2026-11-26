# Adversarial Design Review: `agentic-dev-flow-2026-11-25`

## 1. Bottlenecks (10 projects, 50 MRs/day on `tacplus`, 30–80 min gates)

**[F1] Unbounded child builds on a single small VM cause CPU/RAM collapse.**
`looper-vm-tacplus` is one `c4a-standard-8` (8 vCPUs, 32 GB RAM). Splitting an 80-minute serial gate into 19 parallel jobs still consumes $\ge 80$ vCPU-minutes per build (10+ minutes at 100% 8-core saturation, 30 minutes wall time). At 50 MRs/day and a conservative 2.5 local builds per MR (initial + fix + pre-MR rebase), `tacplus` runs 125 builds/day = **10,000 vCPU-minutes/day**, which is **87% of the entire 24-hour capacity** ($8 \times 1440 = 11,520$ vCPU-minutes) assuming uniform arrivals. In bursts (e.g., 6 ready issues at once $\times$ 19 jobs = 114 containers on 8 vCPUs and 32 GB RAM), Rust/Java linkers will OOM-kill each other, cache locks will convoy, and a 30-minute gate will stretch past 3 hours. The shared org VM (`c4a-standard-4`, 4 vCPUs, 16 GB RAM) running 9 projects at once will OOM with just 3 concurrent builds.

**[F2] Mandatory pre-MR rebase + serial Merger conflict builds create an $O(N^2)$ stall.**
50 MRs/day is one merge every 28.8 minutes on average (every 10–15 minutes during active bursts). Because a report's local gate takes 30–80 minutes, `main` moves **1 to 5 times while every report is building**.
- If Step 4 strictly requires gates to be green on *current* `main` before opening an MR, reports livelock in endless rebase-rebuild loops.
- If MRs open against stale `main`, conflicts hit the serial Merger. If 30% of 50 MRs (15/day) conflict and require a 30-minute local gate (plus 5 minutes of agent resolution), the Merger is blocked for **$15 \times 35\text{ min} = 8.75\text{ hours/day}$**—competing for the same VM's 8 vCPUs. While one conflict builds for 35 minutes, 2–3 new MRs queue behind it; as each merges, queued MRs fall further behind `main`, manufacturing more conflicts.

**[F3] Rolling check + bisect takes 2–5 hours per failure, stalling every active report.**
At 30–80 minutes per full gate, the rolling check runs at most 18–48 times/day. At 50 merges/day, each check batches $B = 2\text{ to }4$ merges on average, and $B = 6\text{ to }8$ during bursts. When a batch of 8 fails, binary search takes $\lceil\log_2 8\rceil = 3$ sequential full builds = **90 to 240 minutes**, plus the initial 30–80 minute red build (**2 to 5.3 hours total**). During that entire window, `main` is red, and every active report rebasing onto `main` fails its local gates on someone else's bug.

**[F4] 19-way parallel gates hit GitLab's 500-job ceiling.**
Running `tacplus` gates as 19 parallel jobs on GitLab runners (Migration Step 1, Rolling Check, Release Builds) across 10 projects consumes $19 \times \text{concurrent pipelines}$ jobs, immediately hitting the ~500-job GitLab Free cap that already caused 56% pipeline refusal rates on 2026-11-24.

**[F5] Spot preemption wipes all in-flight builds simultaneously.**
Because all reports and builds for a project share one spot VM, a single preemption kills every in-flight 30–80 minute build at once. Five reports preempted at minute 25 lose >2 hours of wall work and restart simultaneously on the new VM, triggering **[F1]**.

---

## 2. Correctness Risks & Ownerless Failures

**[F6] Semantic merge skew breaks `main` on clean merges.**
MR 1 changes a function signature or protobuf in `pkg/a`; MR 2 adds a caller in `pkg/b`. Both pass gates against `main@0` and merge minutes apart with no textual conflict and **zero build**. `main` no longer even compiles.

**[F7] Blind revert cascades and blames innocent commits.**
If Merge 1 adds a helper and Merge 3 (landed 20 minutes later) calls it without touching the same lines, `git revert <Merge 1>` applies **cleanly with no conflict**. The revert agent pushes it unbuilt, breaking Merge 3. On the next rolling check, the revert agent sees `main` is still red and wrongly reverts Merge 3.

**[F8] Multi-fault or flaky batches corrupt bisection.**
If two merges in an 8-merge window independently break `main`, or one gate flakes during bisect, binary search invariants fail: the revert agent either reverts an innocent commit or loops while `main` stays broken.

**[F9] Unreviewed LLM conflict resolution silently drops code.**
Until the panel exists, the Merger resolves conflicts alone and runs only the gates mapped to the conflicting files. An LLM resolving conflicts without issue context can delete half a feature or break downstream invariants outside the conflicting file's gate set.

**[F10] Root container socket allows evidence forgery and cache poisoning.**
Every report shares the VM's root container socket and credentials. A misbehaving or prompt-injected report can overwrite the shared build cache (`sccache`/Go cache) with broken artifacts, mutate sibling worktrees, or forge PR text claiming gates passed.

**[F11] Unverified `release/X.Y` commits can reach production.**
The Rolling Check only watches `main`. A hotfix cherry-picked onto `release/X.Y` triggers a Release Build immediately without a prior Rolling Check; if Release Builds only build/attest images rather than running full `just gates`, broken code ships to production.

**[F12] Ownerless failures.**
1. **Red Release Build:** The failure table explicitly says *"Who waits: Nobody."* If image publishing/signing fails on a green commit, no issue is opened and nobody is paged; deployments silently freeze.
2. **Reopened issues on a running VM:** The Kicker only acts when a VM is *off*. If the looper process only scans issues at startup or misses label transitions when the Revert Agent reopens an issue, the issue sits orphaned.
3. **Poisoned shared cache:** If a corrupt incremental cache entry causes deterministic build failures without filesystem corruption, reports will endlessly try to "fix" their code instead of wiping the cache.

---

## 3. Answers to the Nine Open Questions

**[Q1] Merger placement and identity:**
*Disagree with the author's (c)+(b) recommendation where the merger resolves conflicts.* Run a **stateless deterministic controller** (no LLM, no builds) on a tiny dedicated control VM/job that triggers GitHub fast-forward/clean merge. On **any** conflict, reject the MR back to its report (`needs-rebase`). *Reason:* Only the report has issue context, and building conflicts in the Merger creates an 8+ hour serial queue (**[F2]**, **[F9]**).

**[Q2] "No build on a clean merge":**
*Disagree with zero check.* Add a **mandatory <2-minute compile/typecheck gate** on the merged tree before updating `main`, deferring only slow integration/e2e tests to the Rolling Check. *Reason:* Catches cross-MR symbol/signature skew (**[F6]**) in 90 seconds instead of paying 2–5 hours of red `main` and bisection (**[F3]**).

**[Q3] Where the rolling check runs:**
Run it on **one dedicated non-spot VM per heavy project** (and one shared VM for light repos) with a warm cache disk—not on GitLab runners. *Reason:* Avoids GitLab's 500-job cap (**[F4]**), avoids spot preemption mid-bisect, and isolates verification from the untrusted looper VM.

**[Q4] Revert edge cases:**
Re-run only the failed gate group once to filter flakes; if bisect isolates commit $X$, test `git revert X` with a compile check before pushing. If the revert conflicts, fails to compile, or bisect is ambiguous, **abort auto-revert**, advance a `last-known-green` ref no further, and file a P0 `approved` "fix-main" issue. *Reason:* Prevents cascading bad reverts (**[F7]**, **[F8]**).

**[Q5] MR evidence:**
Require a **cryptographically signed GitHub Check Run / Commit Status** posted by a local privileged builder daemon (not the report agent), binding `{commit_sha, base_sha, gate_sets, log_hash}`. *Reason:* PR descriptions written by LLM reports are untrusted text (**[F10]**).

**[Q6] Gate scoping accuracy:**
Generate `gates.yaml` mechanically from build-system dependency graphs (`cargo metadata`, `go list`), and have every red Rolling Check automatically add any missing path-to-gate mapping before reopening the culprit issue. *Reason:* Manual path globs inevitably drift.

**[Q7] Multi-repository changes:**
Add `depends-on: repo#MR` metadata and have the Merger trigger an **immediate commit-tagged pre-release artifact** when an upstream MR is depended on, rather than waiting for nightly releases. *Reason:* Waiting for nightly builds adds an up-to-24-hour queue across repo boundaries.

**[Q8] Keeping the panel fast:**
Run 3 vendor models **in parallel with a hard 90-second timeout and 2-of-3 quorum** (timeout = abstain). *Reason:* Serial review or unanimity makes single-vendor latency spikes a global merge bottleneck.

**[Q9] Missing queues and ownerless failures:**
Add a **per-VM build concurrency semaphore**, a **`main-green` branch ref** for loopers, and **auto-filed P0 issues on red Release Builds**. *Reason:* Directly eliminates **[F1]**, **[F3]**, and **[F12]**.

---

## 4. What Else Is Missing or Wrong

**[F13] Reports branch from `main` instead of `main-green`.**
When `main` is red during a 2-hour bisect, every report rebases onto broken code and tries to debug a failure it didn't cause. Loopers must branch and rebase onto the last commit the Rolling Check passed (`main-green`).

**[F14] No per-issue iteration or spend circuit breaker.**
A report that misunderstands a failure can run local builds and model calls in an infinite loop, saturating the VM indefinitely. Cap each report at $N$ failed build cycles before marking `blocked`.

**[F15] 25-second preemption checkpointing will corrupt shared git stores.**
During a 30-second GCP spot preemption, 10 child worktrees sharing one git object store cannot simultaneously commit and push over one NAT while 19 build containers die without colliding on `.git` locks or timing out.

---

## 5. The Three Changes to Make First, in Order

1. **[C1] Bound concurrent builds per VM and strip conflict resolution from the Merger.** Enforce a strict build-slot semaphore per VM sized to its RAM/vCPUs (**[F1]**), add a <2-minute compile check to the Merger (**[Q2]**), and bounce all conflicts back to the report (**[F2]**, **[Q1]**).
2. **[C2] Decouple looper base branches from `main` head via a `main-green` ref, and harden the Revert Agent.** Have reports rebase onto `main-green` (updated only by green Rolling Checks) so a red `main` never poisons active loopers (**[F3]**, **[F13]**), and halt auto-revert when a revert fails compilation (**[F7]**, **[Q4]**).
3. **[C3] Remove the container socket from reports and isolate builds behind a local signing daemon.** Give reports a narrow local RPC to request gates; only the daemon runs containers and posts signed Check Runs to GitHub (**[F10]**, **[Q5]**).

