# Suggestion types

Every suggestion in `suggestions/` names exactly one **Type** from this list. The list is the source of truth: the nightly
suggestion pass (tacplus-dev/looper#50) reads it and may use no other type. Changing it (adding, renaming or retiring a type) is
done by an agent at Jody's request, like ROADMAP.md. Started 2026-11-01. Jody's example type, *Resiliency*, plus a first set to
refine.

| Type | Covers | Example |
|---|---|---|
| **Resiliency** | surviving failure: faults, restarts, region or node loss, recovery time, degradation | a chaos agent like Chaos Monkey |
| **Security** | vulnerabilities, identity and access, secrets, supply chain, tenant isolation | deterministic SCA and SAST on every build |
| **Correctness** | the platform doing what its contracts and docs promise; data integrity; tests that prove it | servers end streams at token expiry, as API.md claims |
| **Performance** | latency, throughput, build and gate time, resource use per request | gates scaled to what a change touches |
| **Cost** | spend on cloud, runners, models and storage, and what drives it | right-size runner VMs; scale build pools to zero |
| **Operability** | running and changing the platform: deploys, config, observability, runbooks, automation | the uni-config as the single deployment definition |
| **Developer experience** | what building on tacplus is like: generated APIs, tooling, error messages, docs for users | clearer generation errors that name the proto line |
| **Documentation** | docs that are missing, wrong or out of date relative to the code and decisions | README describing a retired Mac kicker |

Pick the type for the suggestion's **main** benefit. When it serves several, name the others in its *Why*.
