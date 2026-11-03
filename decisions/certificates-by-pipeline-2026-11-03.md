# TLS certificates are obtained and renewed by the deployment's pipeline, once and done (Jody, 2026-11-03)

Jody, dictated:
- "We're going to run the deployment ... within the pipeline. That pipeline will check to see if it has a valid
  non-expired certificate, which could be a Let's Encrypt certificate, or it could be a signed certificate from somebody
  else. ... it's going to ... run the Let's Encrypt more as a once and done to get the certificate. Then it will install
  that certificate into the uniconfig and then reapply that uniconfig. And then the pipeline will run every once in a
  while to do the renewal ... it won't actually be stored in the cluster itself. And Let's Encrypt will not be running all
  the time ... it will still stay within our framework ... it won't have to do things like talk to kernel anymore, so it
  doesn't have state anymore."
- On the hub's three questions: "1 -> 2" (DNS-01, run in the pipeline job); "2 -> agreed and installed when it goes into
  the deployment"; "3 -> agree".

## Decided
- **The deployment's pipeline owns the certificate.** On each run it checks for a **valid, unexpired certificate** for the
  deployment's hostnames, and treats one inside the last third of its lifetime as due.
- **If there's none, or it's due:** the pipeline runs tacplus's **letsencrypt container once, inside the pipeline job**,
  with ACME **DNS-01** against Cloud DNS. Then:
  - no port 80, and no deployment needed first;
  - wildcard plus apex where wanted (`*.tacplus.dev`, `tacplus.dev`);
  - letsencrypt is **stateless:** no kernel, no resident service.
- **Stored as a secret version, never in the cluster:** the certificate and key go to Secret Manager as a new version. The
  uni-config **references that version**, and the certificate is **installed when the uni-config goes into the
  deployment** (operator#17's secret delivery). A fresh deployment reuses the current version and orders nothing.
- **Renewal:** a scheduled run of the same pipeline does the check and, when due, the once-and-done job. The new version
  means a new candidate uni-config, then an automatic deploy (in production, the patch canary).
- **Certificates signed by someone else** are supplied as a secret version the same way. The pipeline checks expiry, chain
  and hostname, and skips letsencrypt.
- **The challenge's permission, narrowed:** `_acme-challenge.<domain>` gets a Cloud DNS zone of its own, delegated from the
  main zone. The pipeline's identity may write records only there.
- **Retired once this lands:** the in-deployment letsencrypt pod, `apply-letsencrypt` with its letsencrypt guard, and the
  "never restart letsencrypt" rule.

## Issues
tacplus-dev/letsencrypt (the once-and-done DNS-01 job), tacplus-dev/fleet-seatrials (the pipeline steps),
tacplus-dev/cloud (the secret and the challenge zone; interim by hand in jody1-328023 until the cloud pipeline runs).

## Amended: no letsencrypt container at all (Jody, 2026-11-03)
Jody: "if we're utilizing DNS, I don't necessarily think that we have to have the ... Let's Encrypt container. We can do
this from the pipeline ... I don't know why we would even need to go to the cluster at all."

- **The pipeline's `cert` step uses a standard ACME client**, `lego`, pinned by version and checksum, with its Google Cloud
  DNS provider. It completes DNS-01 and writes the new Secret Manager version. Nothing runs in the cluster, and no tacplus
  service is involved.
- **The check** (valid, due, expired, wrong hostname) is a small pipeline script using `openssl`. It works for Let's
  Encrypt and third-party certificates alike.
- **tacplus-dev/letsencrypt#9 is closed as not needed.** Our deployments no longer use the letsencrypt service. Whether to
  keep the repository for others is left open.

## Renewal and distribution (Jody, 2026-11-03)
Jody: "our pipeline does need to run this every few days or whatever and check to see if it needs to ... renew the
certificate based on Let's Encrypt best practices and do so. And if it does renew, we simply update the secret ... and we
write the uniconfig and apply it to the cluster ... the kernel should basically be able to handle that. And ...
distributing it in state for the gateways." And on the letsencrypt service: "archive it, send it to the boneyard"
(archived 2026-11-03).

- **The schedule:** the pipeline's `cert` check runs **daily**, which is cheap, by Let's Encrypt's best practice.
- **Renew per Let's Encrypt's best practice:** use **ARI** (ACME Renewal Information, which lego supports) for the window
  Let's Encrypt suggests, falling back to renewing in the last third of the certificate's lifetime. This also adapts if
  Let's Encrypt shortens lifetimes.
- **On renewal:** a new Secret Manager version, a new uni-config referencing it, applied (automatically outside
  production; by canary in production).
- **Distribution:** the certificate is part of the gateway's config in the uni-config. Kernel's system-config refresh
  delivers it, and **every gateway hot-swaps it with no restart.** The gateway's `cert_chain` mode already has the
  hot-reload watcher. Its `letsencrypt` mode retires.
