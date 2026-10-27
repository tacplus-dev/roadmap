# Multi-cluster: three regions, quorum, canary

- **Status:** held
- **Owner:** Jody
- **Opened:** 2026-10-22
- **Held on:** nothing to build until a second production cluster is close. Near term is one cluster
  per domain, two domains (dev, prod).
- **Gets more expensive while held:** nothing; the design folds into existing components (kernel,
  Stratum, gateway).
- **Records:** `schematic/roadmap/fleets-and-canary-huddle-2026-10-22.md`

## What

Why more than one cluster: geographic diversity. Shape when it comes: three GKE clusters in three
US regions, each with its kernel instance, gateway and one Stratum writer; `min_durable_copies = 2`;
search members per cluster; cross-cloud is the archive sink, asynchronous. No third Stratum node
type. No reflector: admission is a kernel operation (cloud-identity proof, quorum policy among
existing members, key install, pipeline-designated bootstrap). Releases go by canary and
stair-stepping, pod then cluster, with the Admiral deciding each step. Hazelcast stays.

## Before it starts

A looper verifies one stretched Hazelcast cluster across three regions with a region broken on
purpose. Fallback ladder if it fails: tuning, three zones in one region, Hazelcast's own
cross-region features.
