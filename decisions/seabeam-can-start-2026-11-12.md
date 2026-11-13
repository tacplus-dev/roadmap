# The bar for "SeaBeam can start" (2026-11-12)

**Decision (Jody, 2026-11-12, to the hub: "That's fine. I want to go as fast as possible").** Until now nothing
said when tacplus is ready for SeaBeam to build on. This is the bar, and the loopers work toward it first:

- **A. Service release images and pinned fleets.** Every "Release:" issue (tacplus-dev/gateway#6 first, then
  kernel#7, oauth#5, opa#2, eventbus#2, stratum#3, redo#2, doppelganger#2, tenantecho#2, coop#2, pigeon#2), then the
  pins (tacplus-dev/fleet-seatrials#1, tacplus-dev/fleet-dev#1).
- **B. fleet-seatrials deployed on the `dev` cluster with sign-in working** (tacplus-dev/fleet-seatrials#4, and the
  cluster pipeline it needs).
- **C. A basic end-to-end test run on fleet-dev** (tacplus-dev/fleet-dev#3).

**Order.** Throughput first (`decisions/throughput-first-2026-11-11.md`): nothing in A to C lands while merges
queue for days. Then A, B, C. Stratum per tenant with redo (Next 1) comes after the bar unless SeaBeam's MVP turns out
to need it. No new feature outside the bar is started until it is met; bugs and security fixes go on as before.

**Why.** SeaBeam launches 2026-12-15 with its scope frozen, 303 approved issues are open, and about 12 close a day:
all of Next does not fit. The daily report tracks progress against A, B and C.
