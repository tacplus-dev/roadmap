# Decision index

New decisions live in this directory. Older per-repo decision folders, frozen and cited in place:

- tacplus-dev/kernel `decisions/` (identity, tenant user links, people tenant, security audit 2026-09-12)

- [deploymentproxy-2026-11-02.md](deploymentproxy-2026-11-02.md): nsproxy is renamed deploymentproxy, the one front door per cluster (custom resources only, requesting deployment only); chaos faults go through it, with node-level faults in a separate privileged node agent reachable only through the proxy (Jody).
- [skipper-and-admiral-2026-11-01.md](skipper-and-admiral-2026-11-01.md): Skipper (one per tenant, spans clusters) and Admiral (per deployment, /system) are tacplus components; tenants talk to their Skipper, Skippers to their Admiral (Jody).
- [s2s-never-crosses-tenants-2026-10-31.md](s2s-never-crosses-tenants-2026-10-31.md): S2S is build-time and never crosses tenants; cross-tenant traffic is C2S only (Jody).
- [rust-default-language-2026-10-27.md](rust-default-language-2026-10-27.md): Rust is the default language; Go or Java only with a written reason (Jody).
- [black-box-traffic-2026-10-29.md](black-box-traffic-2026-10-29.md): black-box traffic through the member port; typed gRPC across clusters (Jody). Amended 2026-10-31: may cross clusters over the gRPC member port (#259).
