# Rebuttal: `agentic-dev-flow-2026-11-25`

## 1. Findings of Others to Adopt

* **Codex: 1, 3–7, 9–20, and credential isolation in 8.** Per-VM admission queues (**1, 19**); two-build (60–160m) check detection (**3**); 190-job gate sizing (**4**); coalesced dev deploys (**5**); kicker/spot/quota/human queue SLAs (**6**); canary windows and migration rules (**7**); bounded unverified merge windows and red/stale freezes (**9, 10, 20**); fenced state-machine recovery (**11, 16**); origin-only trust on socket-exposed VMs (**12, 18**); codegen edges in `gates.yaml` (**13**); digest-pinned manifests (**14**); panel capacity budgets (**15**); `release/X.Y` gating and deploy-order locks (**17**).
* **Grok: 1–11, 14, 15, 17, 20–23, 25, 26, and credential separation in 12.** Rebase-to-tip ceiling ($rTe^{rT}=1$, **1, 25**); 19-group flake rate ($1-0.98^{19}=32\%$, **9**); waking VMs on open MRs and shared-VM fair-share scheduling (**5**); reverted-tree quarantine (**8, 26**); per-issue single-flight locks (**21**); non-blocking nightly security/image-pin exemptions (**22**); triage metrics and freeze alerts (**15, 23**).

## 2. Findings Disagreed With

* **Codex 2 & 8, Grok 12 & 16 (Merger resolves conflicts):** Even parked (Codex **2**) or rebuilt on the check VM (Grok **16**), merger conflict resolution lacks issue context and steals check/worker CPU. Bounce conflicts to reports.
* **Grok 13 & 24 ("No compile-on-merge" and "One land per check started"):** One merge per check caps throughput at 18–48 MRs/day (<50/day). A <2m incremental compile check is not an 80m gate; require it per merge within $K \le 4$ merges per check.
* **Grok 16 (Key on project VM):** The root container socket lets containers extract any VM key via host mounts. Remove the socket from reports or verify off-box.
* **Grok 18 (Consumer checks only at release):** Waiting for nightly releases leaves `tacplus` `main` broken for consumers up to 24h. Gate pre-merge via pinned manifests (Codex **14**).
* **Grok 19 (15m panel timeout, fail-open):** Failing open on timeout lets outages bypass security review. Use a 90s deadline (timeout = abstain, 2-of-3 approvals, Codex **15**).

## 3. What All Three Missed

1. **Git `revert-a-faulty-merge` drops code on retry (Step 6):** After `git revert -m 1 <merge>`, the branch's commits stay in `main`'s ancestry. When the next report adds a fix commit to the kept branch and rebases/merges onto `main`, Git sees the original commits as already merged and drops the feature except the fix diff. Reopened branches must rewrite SHAs (`git rebase --no-ff`) or revert the revert.
2. **Orphaned sibling containers leak across failed builds:** Host `containerd`—not the `looper` process tree—owns containers spawned via the host socket. Aborted reports leak build and test-DB containers, exhausting RAM, ports, and cache locks.
3. **GCE Persistent Disk single-writer race wipes warm caches:** A preempted VM takes >30s to detach its cache disk. Because cloud-init (`looper-build-vms-2026-11-24.md` §10b) falls back to a local directory without waiting, replacement VMs boot cold on the small COS root disk and overflow it.

## 4. Final Answers to the Nine Open Questions

1. **Q1:** Stateless non-LLM merger on an isolated control VM setting protected statuses; bounce conflicts to reports. (Unchanged)
2. **Q2:** Require a <2m merged-tree compile check and cap unverified merges at $K \le 4$ ahead of `main-green`. (Changed: Codex **9**)
3. **Q3:** Dedicated non-spot capacity per heavy project (plus a shared light-repo pool) sized for 19-way gates with isolated caches. (Unchanged)
4. **Q4:** Freeze merges on red, rerun once, and auto-revert only if one culprit isolates and compiles; else file `fix-main` and rewrite branch SHAs. (Changed: freeze/SHA rewrite)
5. **Q5:** Strip the root socket from reports so a host daemon signs `{tree_sha, base_sha, gate_policy, image_digest}`, backed by independent release builds. (Changed: Codex **12**)
6. **Q6:** Derive `gates.yaml` from build/codegen graphs, fail closed to full gates on unmapped paths or `gates.yaml` edits, and record misses. (Changed: Codex **13**/Grok **17**)
7. **Q7:** Pin upstream digests in a compatibility manifest and run pre-release consumer gates before promoting cross-repo changes. (Changed: Codex **14**)
8. **Q8:** Run 3 vendor models in parallel off-VM with a 90s deadline, 2-of-3 approvals (timeout = abstain), and quota caps. (Changed: Codex **15**)
9. **Q9:** Add VM build semaphores, `main-green` refs, single-flight locks, reverted-tree quarantine, container reaping, and P0 alerts on red releases. (Changed: Grok **21, 26**, Codex **16**, reaping)

## 5. Single Biggest Remaining Bottleneck

**The ~30-minute indivisible floor of the slowest gate group (`current-generation`) paired with coarse `pkg/**` fan-out.** Even with 19 parallel jobs, Amdahl's law locks full Rolling Checks at $\ge 30$m. With a bounded window ($K=4$), `main-green` lags `main` by 30–60m, and any real failure (detect + rerun + 2-step bisect + revert check) freezes merges for **90–120m** until `current-generation` tests are sharded and `pkg/**` is split.

