# Throughput first (Jody, 2026-11-11)

Jody: "Yes go with this", after the hub's account of why tacplus is slow.

**The numbers (2026-11-11):**
- About 300 approved open issues. 45–50 new a day against 12–15 closed.
- tacplus: 59 approved, 43 finished and waiting to merge, 6 code commits on main in 3 days.
- looper: 38 approved, 27 waiting to merge.
- Of 116 issues filed in 3 days, 88 came from the loopers' own scans (45 cargo-deny, 22 of them one "unlicensed" cause).

**For a few days, until the merge rate is up:**
1. **Intake freeze.** The hub approves only security findings and what blocks merging. The nightly scan keeps running (security first), but non-security findings wait unapproved until looper#129 (one issue per cause) lands.
2. **Merging fixes first, every one priority:high:**
   - tacplus#300 (gates in parallel: ~1.5 of 16 CPUs used today);
   - tacplus#266 (a gate map, so a change runs the gates it needs);
   - looper#121 (manager by priority), #128 (queued builds aren't red), #114 (planning-only moves keep a verdict), #86 (batching), #118 and #127 (build triggers), #113, #122;
   - the flaky gates tacplus#526, #531, #540, #562;
   - cluster#20 (in-cluster builds).
3. **The hub lands these by hand** once their gates are green, as with looper#100. They can't wait behind the queue they fix.
4. **No hub infrastructure changes** beyond cluster#20.

**Done when** tacplus merges about 10 or more a day and the approved backlog shrinks. Then intake reopens.
