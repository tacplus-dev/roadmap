# State: tacplus-dev (platform)

Overwritten to stay current (loopers and agents). Last updated: 2026-11-11 by an agent, at Jody's request (#3).

- **Version / line:** tacplus **v2.6.6** (2.6 line: v2.6.0 2026-11-05, v2.6.6 2026-11-10); the 2.0–2.4 lines still
  get patches (v2.4.16 and its siblings, 2026-11-10). The services declare the 2.6 floor. Those the fleets deploy
  have `v0.1.x` source tags (Jody, 2026-11-07), all `0.x` until 2026-12-01; only llmproxy and kmaker publish their
  images so far.
- **Works:** the Stratum path's first steps are done: the records write surface (tacplus-dev/tacplus#163), routing
  by target tenant (tacplus-dev/tacplus#170), per-tenant in-memory Stratum (tacplus-dev/stratum#2), certificates
  written to Stratum (tacplus-dev/letsencrypt#2), and redo's local step (tacplus-dev/redo#1, on main). Fleets pin
  by release line (tacplus-dev/tacplus#180, tacplus-dev/fleet-seatrials#8, tacplus-dev/fleet-dev#2). The operator
  renders `tac-<deployment>` and `tac-<deployment>-<tid>` namespaces through deploymentproxy
  (tacplus-dev/operator#3), and the cluster pipeline holds cluster-admin (tacplus-dev/cluster#1). Tenant deletion
  issues no new AT/ST, removes memberships and ages out idle XIDs (tacplus-dev/kernel#19, tacplus-dev/kernel#20).
  fleet-seatrials' deploy and test pipeline runs on GitLab (tacplus-dev/fleet-seatrials#17), aimed at
  tacplus-dev/cloud's `dev` cluster in seabeam.net (tacplus-dev/fleet-seatrials#39).
- **Released / deployed:** nothing is deployed. seatrials' old clusters are gone (`seatrials` 2026-10-29,
  `tacplus` 2026-11-05), and its first deploy on `dev` waits on the pins (tacplus-dev/fleet-seatrials#1) and the
  move to `tac-seatrials` (tacplus-dev/fleet-seatrials#4). When it was live, sign-in with Google (`alpha`),
  Microsoft (`beta`) and Apple worked.
- **Loopers:** pipeline gitlab.com/tacplus-dev/looper: one job per issue (a report on its own branch, up to each
  repository's cap) and a manager per repository that alone merges to main on green gates
  (`decisions/looper-job-per-issue-2026-11-03.md`). Jobs run on the interim runner VMs until the in-cluster runner
  takes over (tacplus-dev/looper#84). The roadmap scan files suggestions here daily (`suggestions/`).
- **Known gaps:** redo is decided a `/system` service (Jody, 2026-11-07, on redo#1), but `pinned_tenant` (restore)
  and `RPC_FLAG_ANY_TENANT` are still there until tacplus-dev/tacplus#258; its second-site copy is parked
  (tacplus-dev/tacplus#204). The operator does not render per-tenant Stratum and redo yet (tacplus-dev/operator#2,
  waiting on tacplus-dev/stratum#26 and tacplus-dev/kernel#34). The identity graph is still cluster-local in
  kernel (tacplus-dev/kernel#2, waiting on tacplus-dev/tacplus#206). Most services have no published image
  (tacplus-dev/gateway#6 first, then each "Release:" issue), so neither fleet can lock (tacplus-dev/fleet-seatrials#1,
  tacplus-dev/fleet-dev#1). Tenant providers still outlive a deleted tenant (tacplus-dev/oauth#11). The fleet
  client does not generate on main (tacplus-dev/fleet-seatrials#14). Open dependency advisories on most repos.
- **Blocked on (Jody):** `needs-decision`, checked 2026-11-11 against the open issues: tacplus-dev/tacplus#145,
  tacplus-dev/tacplus#1, tacplus-dev/cluster#18 (the hub's in-cluster runners and cluster's own pipeline),
  tacplus-dev/llmproxy#56 (new non-blocking security findings). Nothing on the Stratum path or the releases waits on
  a decision; they wait on the issues above.
