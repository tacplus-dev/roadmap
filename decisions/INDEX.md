# Decision index

New decisions live in this directory. Older per-repo decision folders, frozen and cited in place:

- tacplus-dev/kernel `decisions/` (identity, tenant user links, people tenant, security audit 2026-09-12)

- [certificates-by-pipeline-2026-11-03.md](certificates-by-pipeline-2026-11-03.md): the deployment pipeline checks for a valid certificate; if none, it runs letsencrypt once in the pipeline job by DNS-01; stored as a Secret Manager version the uni-config references; renewed on a schedule; nothing in the cluster (Jody).
- [trace-propagation-2026-11-03.md](trace-propagation-2026-11-03.md): trace_id on every message; user code copies it (no ambient context); tacplus copies it across projections and transformations; cleared on every store (Jody).
- [deterministic-deployments-2026-11-02.md](deterministic-deployments-2026-11-02.md): every deployment is deployed and managed only through deterministic, reproducible pipelines; people and AI may do anything, but lasting changes go through the pipeline; a known-good uni-config is recorded and deployed as it is (Jody).
- [cloud-pipeline-2026-11-02.md](cloud-pipeline-2026-11-02.md): every cloud and CI change comes from a pipeline and people read only; new GCP projects for tacplus-dev (jody1-328023 removed after), GitLab settings too, production planned in from the start (Jody). Design: roadmap#2.
- [cluster-pipeline-2026-11-02.md](cluster-pipeline-2026-11-02.md): tacplus-dev/cluster is the only holder of cluster-admin; plan on every change, apply only by Jody, explicit deletes (Jody, roadmap#1).
- [deploymentproxy-2026-11-02.md](deploymentproxy-2026-11-02.md): nsproxy is renamed deploymentproxy, the one front door per cluster (custom resources only, requesting deployment only); chaos faults go through it, with node-level faults in a separate privileged node agent reachable only through the proxy (Jody).
- [skipper-and-admiral-2026-11-01.md](skipper-and-admiral-2026-11-01.md): Skipper (one per tenant, spans clusters) and Admiral (per deployment, /system) are tacplus components; tenants talk to their Skipper, Skippers to their Admiral (Jody).
- [s2s-never-crosses-tenants-2026-10-31.md](s2s-never-crosses-tenants-2026-10-31.md): S2S is build-time and never crosses tenants; cross-tenant traffic is C2S only (Jody).
- [rust-default-language-2026-10-27.md](rust-default-language-2026-10-27.md): Rust is the default language; Go or Java only with a written reason (Jody).
- [black-box-traffic-2026-10-29.md](black-box-traffic-2026-10-29.md): black-box traffic through the member port; typed gRPC across clusters (Jody). Amended 2026-10-31: may cross clusters over the gRPC member port (#259).
