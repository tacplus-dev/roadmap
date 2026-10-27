# Doppelganger as a fleet service

- **Status:** active (design agreed 2026-10-22; 2026-10-23 plan agreed and engine/registry steps built, see below)
- **Owner:** Jody (design), Snoopy (build)
- **Opened:** 2026-10-22
- **Gets more expensive while held:** nothing; today's local daemon keeps working for Mac-side agents.
- **Records:** github.com/tacplus-dev/doppelganger (private; local `~/workspace/github/tacplus-dev/doppelganger`,
  moved 2026-10-23 from gitlab/seabeam/tools); its `DOPPELGANGER-STATE.md` is the pick-up state; `~/workspace/jody/agents.md`
  ("An agent that represents him"); [looper-hierarchy](looper-hierarchy.md); [tacplus-registry-and-floor](tacplus-registry-and-floor.md)

## What (Jody, 2026-10-22)

Doppelganger runs in our cluster as a service. Workloads from GitLab (the loopers) authenticate and
are allowed in; the kernel maps the GitLab claims through to the platform token, and doppelganger
uses those claims to make decisions about what it answers and from which part of the context repo.

## Design

- Service surface: today's C2D contract becomes an in-fleet service surface; callers are workloads
  (`sub_type` pipeline/workload), authenticated through the cloud-auth facade, authorized by
  doppelganger's own policy, default deny.
- Decisions on claims: which namespace/project is asking (immutable ids), `ref_protected` (refuse
  unprotected refs), `pipeline_source` (refuse MR pipelines from outside), plus the caller's
  declared purpose in the session context. These select the disclosure class: a code looper gets
  `code.md`; business and life material never leave the repo for a pipeline caller.
- Context repo in cluster: doppelganger needs a current clone of `secondstar/jody`; pull on a
  schedule through the gitlab-connector, never a mounted secret.
- The model: the call goes through llmproxy (S2S, doppelganger's own service identity as the caller);
  llmproxy holds the federated identity to Anthropic. Doppelganger never holds a model credential.
  **2026-10-23 (Jody): doppelganger does NOT run locally.** It is a fleet service only; locally there is a
  client that signs in, gets a platform token and calls it. llmproxy moves to github.com/tacplus-dev/llmproxy
  (wholesale copy 2026-10-23 from sonar/backend d45d97b; divergence tracked as issues on the public repo; see its README).

## Open

- The exact claim-to-disclosure-class table (Jody's call, per class).
- Whether human callers (Jody from a desktop agent) reach the same service or stay on the local
  daemon.

## Plan (Jody, 2026-10-23: "Yes on all")

Doppelganger is the proving ground for the new design because it is free-standing from Seabeam;
nothing here may disturb the loopers or the 2026-12-01 launch.

1. Registry repo under github/tacplus-dev — DONE (seeded; doppelganger = 101).
2. tacplus branch — DONE locally and pushed (`registry-and-floor`): global registry + locks + version floor.
3. Doppelganger repo under github/tacplus-dev — DONE (private). Header entity (Seabeam LLC vs Jody Scott)
   must be settled before it goes public.
4. Build and test — engine `Provider` seam DONE on branch `engine-provider`; the fleet provider is the
   generated llmproxy S2S client (NOT a vendor SDK; the Anthropic adapter built earlier that day was removed).
   Needs: llmproxy gains an output-schema field on GenerateCompletionRequest and `doppelganger` in its
   auth_services; doppelganger's proto moves from C2D to a client-facing surface with a caller token.
   No API keys anywhere (federated identity).
5. Kernel rewrite in Rust, Hazelcast out of kernel (stays in Stratum) — its own track, merges after launch.
6. Merge back.

Test deployment: a small separate cluster built from the branch. A namespace on the launch cluster would
still require its kernel to learn doppelganger's types, which is the impact we are avoiding.
