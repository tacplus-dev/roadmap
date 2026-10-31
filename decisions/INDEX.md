# Decision index

New decisions live in this directory. Older per-repo decision folders, frozen and cited in place:

- tacplus-dev/kernel `decisions/` (identity, tenant user links, people tenant, security audit 2026-09-12)

- [s2s-never-crosses-tenants-2026-10-31.md](s2s-never-crosses-tenants-2026-10-31.md): S2S is build-time and never crosses tenants; cross-tenant traffic is C2S only (Jody).
- [rust-default-language-2026-10-27.md](rust-default-language-2026-10-27.md): Rust is the default language; Go or Java only with a written reason (Jody).
- [black-box-traffic-2026-10-29.md](black-box-traffic-2026-10-29.md): black-box traffic only inside a cluster, through the member port; typed gRPC across clusters (best practice, Jody).
