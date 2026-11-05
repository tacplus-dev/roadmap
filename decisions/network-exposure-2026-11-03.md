# Only the load balancers are public; nothing else accepts internet traffic; COS, no SSH (Jody, 2026-11-03)

Jody: "why do nodes have external IPs ?????" ... "the only thing that should have a public IP is the LB. Thats it and we
will only have 2 now. one for tacplus.dev and one for seabeam when it deploys" ... "absolutely no internet traffic should
be ingressing these nodes. they only reach out. the LB is the only way in." ... "or whatever google provides that is only
for running containers" ... "and we should be running COS with NO SSH".

## Decided
- **Public IPs:** only the load balancers. Today that's tacplus.dev's (`seatrials-gateway`, 146.148.80.132); SeaBeam's
  comes when it deploys.
- **No ingress from the internet** to any node or VM. They only reach out, through Cloud NAT. The load balancer is the
  only way in, through the narrow rules Kubernetes creates for its service.
- **Nodes and VMs run Container-Optimized OS, with no SSH.** Inspect them through Google's APIs, logs and the serial
  console, never by logging in.

## Done by the hub, 2026-11-03
- **Found:** `any-ingress` (all traffic from 0.0.0.0/0 to every instance, from 2021); `default-allow-ssh` and
  `default-allow-rdp` from 0.0.0.0/0; `default-allow-http`, `-https` and `-icmp` from 0.0.0.0/0; and public IPs on every
  GKE node. Kernel's unauthenticated Raft port (kernel#42) was therefore reachable from the internet.
- **Deleted** all six rules, and the IAP SSH rule. What remains: internal ranges only (`default-allow-internal`, GKE's
  own), plus egress.
- **GKE node pools switched to private nodes** (rolling).
- **Runner templates without external IPs:** `runner-arm64-t2a8-v8-spot-private`, `runner-arm64-t2a16-v9-spot-private`,
  `runner-arm64-deploy-v3-private`. The runner manager had none already.

## Remaining
- **The runner VMs still run Ubuntu.** The GitLab autoscaler reaches them by SSH from inside the network only. They go
  away when loopers and builds move into the cluster (`cluster-node-pools-2026-11-03.md`); until then that internal SSH
  is the one exception.
- **The GKE control plane's public endpoint** (34.45.55.23) **stays public** (Jody: "keep it public"). It's Google's managed
  endpoint, authenticated by Google IAM, not a node or VM.
- **Cloud NAT's egress IP is outbound only,** and stays.
- **The cloud pipeline** (cloud#1, cloud#5) must encode all of this: org policy to restrict external IPs on VMs, and no
  firewall rule from 0.0.0.0/0 except to load balancers.

## Addendum: we terminate TLS, not the cloud (Jody, 2026-11-05)
Jody: "A. We terminate TLS we dont let the cloud do it." The public entry is Google's **global external proxy Network
Load Balancer (TCP proxy)** on the reserved global address `gateway` (tacplus-dev/cloud#13), passing the TLS connection
through unopened to the tacplus gateway pods (zonal NEG backends). TLS ends in our gateway, with the pipeline's
certificate (decisions/certificates-by-pipeline-2026-11-03.md) and hot swap (gateway#25). Nodes accept traffic only
from Google's proxy ranges, never from the internet. No L7 Gateway, no Google-managed or Certificate Manager
certificate for our traffic.
