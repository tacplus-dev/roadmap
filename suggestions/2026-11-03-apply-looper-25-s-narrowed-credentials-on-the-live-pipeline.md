Suggestion: Apply looper#25's narrowed credentials on the live pipeline
Type: Security
Summary: Replace the looper jobs' GITLAB_TOKEN (Jody's group-Owner looper-runtime) with the read_api bot token and revoke it, move the workers off build-box, and have Jody decide fleet-seatrials' re-key
Why: looper#25 was built and closed on 2026-10-30, but STATE.md (2026-11-03) still says the jobs hold Jody's api/Owner token, which can read fleet-seatrials' protected variables (FLEET_CRYPTO_YAML), and the workers run as build-box, which can overwrite Artifact Registry releases and any bucket's objects. The withholding from the model is not a boundary (/proc/<parent>/environ), and the models read issue text from outside the org (looper#40). So any prompt injection that gets through reaches group-Owner and project-wide GCP write. No open issue tracks applying it. Also serves Operability: the record stops saying one thing while the pipeline runs another.

## Evidence
- https://github.com/tacplus-dev/looper/issues/25 (closed 2026-10-30: the fix; README Credentials steps 1–5 are the hub's to apply)
- https://github.com/tacplus-dev/looper/blob/main/STATE.md (Known gaps, updated 2026-11-03: 'the jobs' GITLAB_TOKEN is still Jody's looper-runtime (api, Owner of the group: it reads fleet-seatrials' protected variables) and the workers still run as build-box (Artifact Registry uploads, every bucket's objects)')
- https://github.com/tacplus-dev/looper/blob/main/PROPOSALS.md ('The model reads its parent's environment', 2026-10-30, #25: the withheld credentials are still readable from /proc/<parent>/environ; 'Whether fleet-seatrials' key material counts as exposed': Jody's re-key decision, still open)
- https://github.com/tacplus-dev/looper/blob/main/README.md#credentials (build-box as of 2026-10-30, testIamPermissions: Artifact Registry upload, create/read/delete objects in every bucket of jody1-328023)

## Cost
Hours: the hub follows README Credentials steps 1–5 (bot membership, read_api token stored as GITLAB_TOKEN, the 404 check on fleet-seatrials, revoke looper-runtime) and switches the worker template's service account; Jody decides on the fleet-seatrials re-key (deploy/init.sh) and does it if needed. The window to apply is the kicker's last minutes of an hour (README).

## Touches
looper#25 (least-privilege credentials), looper#40 (prompt provenance: untrusted text reaches the model), decisions/looper-identity-panel-2026-10-29 (per-repository identity), fleet-seatrials#17 (secrets held by GitLab, nothing on a laptop)

## Reuse considered
The procedure is already written and tested (README Credentials, #25), so nothing new needs building. looper#28's per-repository app identity would replace the shared token in the end, but it is a kernel-only pilot still waiting on the hub's setup. It doesn't remove the Owner token from the other 31 repositories' jobs, so it is no substitute for applying #25 now.

## Status
open
Rank: 1
Filed: 2026-11-03 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/17692416595)
