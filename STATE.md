# State: tacplus-dev (platform)

Overwritten to stay current (loopers and agents). Last updated: 2026-10-27 by an agent, at Jody's request.

- **Version / line:** tacplus **v2.4.1** (2.4 line; v2.4.0 and v2.4.1 released 2026-10-27). Services are moving
  to the tacplus floor 2.4; none has a versioned release yet (all `0.x` until 2026-12-01).
- **Works:** fleet-seatrials (GKE `seatrials`, https://seatrials.tacplus.dev) runs kernel, gateway, oauth, opa,
  eventbus, stratum, letsencrypt, kmaker, llmproxy, doppelganger, tenantecho, coop and pigeonhost. Sign-in with
  Google (routed to `alpha`), Microsoft (routed to `beta`) and Apple works. nsproxy is built (tenant namespaces,
  the reserved `restore` tenant `00000002`).
- **Released / deployed:** seatrials is deployed from commit pins in its `fleet.lock`, not release lines. Tenant
  routing runs on an interim kernel-config override until the operator fix (tacplus-dev/operator#5, pushed
  3801f16) is released and deployed.
- **Loopers:** pipeline gitlab.com/tacplus-dev/looper; jobs on the group runner *tacplus-dev arm64 autoscaler*
  (manager `gitlab-runner-mgr`), workers in the GCE managed instance group `runner-arm64-tacplus`
  (t2a-standard-8, up to 5, size 0 at rest). One looper per repository, on `approved` issues.
- **Known gaps:** per-tenant Stratum, the redo service and the people service do not exist yet; the identity
  graph is still in kernel state. Services cannot release (no release CI credential). Seatrials still runs in
  the old namespaces (tacplus-dev/fleet-seatrials#4). Open dependency advisories on most repos. This repository
  carries ROADMAP.md and STATE.md only so far (no PROPOSALS.md or HISTORY.md).
- **Blocked on (Jody):** tacplus-dev/tacplus#163 (decisions D1–D6; blocks the whole Stratum path);
  tacplus-dev/looper#12 (org secret `TACPLUS_REGISTRY_TOKEN`; blocks every service release);
  tacplus-dev/gateway#6 (how the fleet-shaped gateway image is released). Also `needs-decision`:
  tacplus-dev/tacplus#145, tacplus-dev/tacplus#1.
