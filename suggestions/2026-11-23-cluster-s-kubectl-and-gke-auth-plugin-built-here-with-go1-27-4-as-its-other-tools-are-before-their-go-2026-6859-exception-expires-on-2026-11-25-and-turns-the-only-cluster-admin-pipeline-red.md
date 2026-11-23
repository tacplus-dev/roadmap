Suggestion: cluster's kubectl and GKE auth plugin built here with go1.27.4, as its other tools are, before their GO-2026-6859 exception expires on 2026-11-25 and turns the only cluster-admin pipeline red
Type: Security
Summary: cluster's security-exceptions.yaml accepts CVE-2026-97054 / GO-2026-6859 (net/http, HIGH) in kubectl v1.34.13 (go1.26.10) and gke-gcloud-auth-plugin 0.5.21 (go1.27.3) until 2026-11-25, two days from now, because 'no kubectl 1.34 or plugin release has the fix yet'; cluster's PROPOSALS (2026-11-13, #21) names the three ways out and none has been taken: 'a kubectl 1.34 patch and a Cloud SDK component built with go1.26.11+/go1.27.4, or kubectl built here from its tag with go1.27.4 (as Trivy is since #21), or a new dated decision'. ci/tools.sh already builds gitleaks, osv-scanner, crane and trivy from source with go1.27.4 and the fixed modules (cloud#24's policy) but fetches kubectl and the plugin as upstream binaries from dl.k8s.io and dl.google.com. Build both from their tags with go1.27.4 in ci/tools.sh (kubernetes cmd/kubectl at v1.34.13, cloud-provider-gcp's cmd/gke-gcloud-auth-plugin at its tag), drop the two entries, and let this be the first case the week-ahead expiry warning (the open suggestion of 2026-11-15) would have filed
Why: On Wednesday the pipeline that alone holds cluster-admin (plan, drift and the applies Jody plays) goes red on an expired exception, with seatrials' first deploy in flight (fleet-seatrials#65, cloud#94) and cluster#46 (the nightly runner entry, needs-decision) and cluster#41 waiting to be applied: a red cluster main blocks every runner and namespace change the bar needs, as cluster#48's red security job did this morning until the relock. Fixing the HIGH beats renewing the accept, and the pattern is in the same file. Also Operability

## Evidence
- https://github.com/tacplus-dev/cluster/blob/main/security-exceptions.yaml: two entries 'id: CVE-2026-97054', 'expires: 2026-11-25', 'where: kubectl' and 'where: gke-gcloud-auth-plugin', reason 'GO-2026-6859, an HTTP/1 server crash in net/http (gke-gcloud-auth-plugin 0.5.21, go1.27.3; kubectl v1.34.13, go1.26.10; fixed in go1.26.11 and go1.27.4) ... no kubectl 1.34 or plugin release has the fix yet'; its header: 'kubectl v1.34.13 GO-2026-5158 6603 6610 6611 6612 6617 6629 6860 6866, 6859 (go1.26.10, #21)'
- https://github.com/tacplus-dev/cluster/blob/main/PROPOSALS.md (2026-11-13, #21): 'GO-2026-6859 in kubectl and gke-gcloud-auth-plugin after 2026-11-25 ... it expires on 2026-11-25 as the hub dated it. Before then: a kubectl 1.34 patch and a Cloud SDK component built with go1.26.11+/go1.27.4, or kubectl built here from its tag with go1.27.4 (as Trivy is since #21), or a new dated decision'
- https://github.com/tacplus-dev/cluster/blob/main/ci/tools.sh: gitleaks, osv-scanner, crane and trivy built with GOTOOLCHAIN go1.27.4 and each fixed module ('A tool built from source is built with each fixed module it names, the policy of tacplus-dev/cloud#24'); kubectl_version=v1.34.13 fetched from dl.k8s.io by sha256, the plugin from dl.google.com
- suggestions/2026-11-15-a-week-before-a-security-exception-expires-the-nightly-scan-files-the-decision-it-needs.md (open, rank 5): this expiry is its first case, and nothing filed the decision; cluster's open issues tonight (#33, #36, #41, #46, #50) hold no kubectl build
- tacplus-dev/cluster#48 and tacplus-dev/cloud#97 (2026-11-23, closed with a relock): 'security job fails main's apply', the effect an expired entry has on the apply pipeline

## Cost
Hours to a day: two from-source builds in ci/tools.sh (a clone and go build of cmd/kubectl at v1.34.13 and of the plugin at its tag, GOTOOLCHAIN go1.27.4, their sha pins replaced by the module sums), the two exception entries dropped, one cluster land on the full gates, applied by Jody

## Touches
decisions/cluster-pipeline-2026-11-02.md (the only holder of cluster-admin; apply by Jody); decisions/deterministic-builds-2026-10-27.md (pinned, reproducible tools); the accepted suggestion of 2026-11-04 (the deterministic security checks in the pipelines that apply infrastructure); cloud#24's fixed-module policy; the open suggestion of 2026-11-15

## Reuse considered
ci/tools.sh's from-source path (go build with GOTOOLCHAIN and fix modules, cloud#24) is exactly what the two binaries need, and cluster#21 did it for trivy and for GitLab Runner's own images (ci/runner-images.sh, go1.27.4); the alternative, a renewed dated entry, is what the file's header already calls an interim

## Status
open
Rank: 3
Filed: 2026-11-23 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18274230893)
