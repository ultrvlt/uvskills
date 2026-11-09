---
name: ultraviolet-add-site
description: Connect another Kubernetes cluster, region or less-trusted network to an existing Ultraviolet Observability core as a remote site — prepare the core for enrollment, publish the ingest and enrollment endpoints, install the ultrvlt-site connector chart with a join token, and confirm data arrives. Use when someone wants to add, enroll or connect a site, cluster or region, or monitor a cluster from a central core.
---

# Connect a remote site

A **site** runs the `ultrvlt-site` chart: a connector that enrolls with the
core using a one-time join token, then forwards telemetry over mutual TLS,
buffering to disk through outages. The site only dials out; nothing connects
in to it. Reference: https://docs.uvobservability.com/deploy/remote-sites/

Two kube contexts are involved: the **core's** and the **site's**. Name which
one each command runs against, and confirm before changing either.

## 1. Is the core ready?

Check the core's values for enrollment (read single keys; don't dump all
values, which can contain secrets):

```bash
helm get values ultrvlt -n ultrvlt -o json | jq '{enrollment: .enrollment.enabled, wan: .ingestGateway.wan.enabled, enroll: .queryApi.enroll}'
```

If enrollment is already on and endpoints are published, go to step 4.

**Before turning enrollment on**, warn the user: it switches the ingest
gateway to mutual TLS, so it stops accepting plain OTLP — including from a
sensor, log collector or app on the **core's own cluster**. If anything sends
to the core in the clear, plan step 3 too.

## 2. Prepare the core

Agree the topology with the user:

- **A — over the internet / another account:** a dedicated, internet-facing
  network load balancer for the ingest port only, restricted to the sites'
  egress ranges. On AWS between accounts, prefer PrivateLink.
- **B — a less-trusted cluster on the same network:** an internal load
  balancer whose security group admits port 4317 only from that cluster, plus
  a NetworkPolicy so only the ingest gateway pods accept it.

The ingest endpoint must stay a **connection-level** load balancer (NLB, not
an ALB): the core terminates mutual TLS itself.

Core values (AWS example, topology A):

```yaml
enrollment:
  enabled: true
ingestGateway:
  wan:
    enabled: true
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: external
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    sourceRanges:
      - 203.0.113.0/24       # the sites' egress ranges, never 0.0.0.0/0
queryApi:
  enroll:
    publicUrl: https://enroll.example.com          # the enrollment endpoint, below
    corePublicEndpoint: ingest.example.com:4317     # the ingest load balancer
    coreServerName: ingest-gateway
```

Sites also need to reach the **enrollment service** (`ultrvlt-enrollment`,
port 8090) over HTTPS, scoped to the same ranges, to enroll and later renew.
An ALB or other HTTPS ingress is right for this one. Example in the reference
above.

Preview and apply on the core:

```bash
helm upgrade ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version <installed version> \
  -n ultrvlt --reset-then-reuse-values -f core-sites.yaml --dry-run
```

## 3. If the core collects its own telemetry

Install the same `ultrvlt-site` chart **on the core's cluster**, enrolled like
any site (its own join token from step 4), dialing in-cluster names:

```bash
read -rs JOIN_TOKEN
helm install ultrvlt-site oci://ghcr.io/ultrvlt/charts/ultrvlt-site --version <core version> \
  -n ultrvlt \
  --set aggregator.coreEndpoint=ultrvlt-ingest-gateway:4317 \
  --set aggregator.coreServerName=ingest-gateway \
  --set aggregator.enrollUrl=http://ultrvlt-enrollment:8090 \
  --set aggregator.joinToken.value="$JOIN_TOKEN"; unset JOIN_TOKEN
```

Then point the core chart's own producers at it — `sensor.otlpEndpoint`,
`clusterState.otlpEndpoint`, `logCollector.otlpEndpoint` (and
`prober.otlpEndpoint` / `prober.assignmentsEndpoint` if synthetics run) to
`ultrvlt-site-aggregator:4317` — and trust its CA:
`--set 'internalTLS.trustConfigMaps={ultrvlt-site-internal-ca}'`. Install or
upgrade this connector **before** the core whenever both change.

## 4. Mint a join token

In the portal, an admin chooses **Sites → Add site**, names the site and
cluster, and picks how long the token stays valid. The **site name** is
permanent: it's the identity in the site's certificate and on all its data.

The user keeps the token; don't have them paste it into the chat. It's read
with `read -rs` in the next step.

## 5. Install the connector at the site

Against the **site's** context. The wizard also offers a one-container
`docker run`; for Kubernetes, use the chart:

```bash
read -rs JOIN_TOKEN
helm install ultrvlt-site oci://ghcr.io/ultrvlt/charts/ultrvlt-site --version <core version> \
  -n ultrvlt --create-namespace \
  --set aggregator.coreEndpoint=ingest.example.com:4317 \
  --set aggregator.coreServerName=ingest-gateway \
  --set aggregator.enrollUrl=https://enroll.example.com \
  --set aggregator.joinToken.value="$JOIN_TOKEN" \
  --set sensor.enabled=true \
  --set clusterState.enabled=true; unset JOIN_TOKEN
```

- `coreServerName` is the name on the core's certificate (default
  `ingest-gateway`), **not** the ingest host name.
- `sensor`, `clusterState`, `logCollector` and `prober` are off by default and
  export to the connector automatically. Turn on what this cluster should send.
- The site's own apps can send OTLP/gRPC to `ultrvlt-site-aggregator:4317`.
- Put the non-secret settings in a values file for the site, and commit it.

## 6. Confirm

```bash
kubectl -n ultrvlt logs deploy/ultrvlt-site-aggregator --tail=50
kubectl -n ultrvlt get pods
```

The Add site wizard confirms when the first data arrives; the site then shows
on **Sites** with when it was last seen. If it doesn't arrive:

- **Enrollment fails:** the site can't reach `enrollUrl`, or the token
  expired or was already used (mint a new one).
- **Enrolled but nothing forwards:** the site can't reach `coreEndpoint`
  (source ranges, security group), or `coreServerName` doesn't match.
- **Sensor pods restart (OOMKilled):** raise `sensor.resources.limits.memory`
  (for example `1Gi`); see the `ultraviolet-troubleshoot-sensor` skill.

## Optional: keep data in the site's environment

For a site whose raw logs and traces must not leave it, the connector's
privacy policy (`aggregator.privacy.enabled`) masks logs into patterns,
allows attributes by name, templates paths, and hashes identifiers before
anything is sent. Read https://docs.uvobservability.com/deploy/remote-sites/#keeping-customer-data-in-its-environment
with the user before turning it on.

## Removing a site

**Sites → Enrolled sites → Decommission** (admin) revokes its certificates
within a minute; **Undo** reverses it. The connector keeps buffering meanwhile.
