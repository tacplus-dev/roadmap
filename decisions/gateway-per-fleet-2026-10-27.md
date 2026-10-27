# The gateway is built per fleet; the fleet builder merges the webapp (DECIDED 2026-10-27)

Jody, 2026-10-27 (answering gateway#6).

- **The gateway image is built per fleet, by the fleet builder**, never deployed as one generic release image. Its route table
  is generated from the fleet's merged protos, and it serves the fleet's merged webapp.
- **Services contribute web pieces and clients from their own repositories**, at the version the fleet pins: e.g. a webapp module for
  the test ping utility, or a service's downloadable clients. The fleet builder gathers them, builds one **merged webapp**, then
  builds the gateway with it.
- The gateway repo still **tags releases** (its line and code), and fleets pin it by line like any service
  (decisions/deterministic-builds-2026-10-27.md, tacplus#180). The fleet's gateway image is built from that tag, in the pinned build
  image, by the build pipeline: deterministic, no agent.
- This is the one service whose deployed image is fleet-specific. Every other service deploys its published image by digest.
