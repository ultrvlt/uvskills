---
name: ultraviolet-install
description: Install the Ultraviolet Observability core into a Kubernetes cluster — check prerequisites, size storage, verify the signed chart, create the admin credential (and the license, unless they start on the free tier), install, reach the portal over HTTPS, and turn on data sources. Use when someone wants to install, deploy, set up or try Ultraviolet Observability.
---

# Install the Ultraviolet Observability core

This installs version **0.1.152** (current stable) unless the user
was given another version. Reference: https://docs.uvobservability.com/install/installation/
(and /deploy/eks/ for AWS).

Work through the steps in order. Show each command before running anything
that changes the cluster, and ask first.

## 1. Confirm where you are installing

```bash
kubectl config current-context
kubectl version            # server must be 1.26+
helm version               # Helm 3; 3.14+ for --reset-then-reuse-values
kubectl get storageclass   # ClickHouse needs a default, or a named one
kubectl auth can-i '*' '*' --all-namespaces   # first install needs cluster-admin (CRDs)
```

Ask the user to confirm the context is the right cluster. If there is no
default StorageClass, note the name to set as `clickhouse.persistence.storageClass`.

## 2. Ask the questions that shape the install

- **Free tier or a license?** With no license the core runs the free tier: up
  to 50 nodes and 24 hours of data, 3 days once they register an email from the
  portal. It needs outbound HTTPS to `desk.uvobservability.com` for an hourly
  usage report (node counts, never telemetry). SSO and roles, more sites,
  air-gapped installs and automatic fixes need a license. Starting free and
  adding a license later keeps the data.
- **With a license: connected or offline?** Connected mode renews the license
  automatically and sends usage metadata only (license ID, version, node
  counts). Offline (air-gapped) means installing a new `.lic` before the current
  one expires.
- **Can the cluster pull from `ghcr.io/ultrvlt`?** If not, see
  [Air-gapped](#air-gapped).
- **How will people reach the portal?** An ingress with TLS (required for
  sign-in to stick) — which ingress class, host name, and how TLS terminates.
- **Rough traffic and retention**, for sizing (next step).
- **Single sign-on now, or the local admin first?** Local admin is simplest;
  OIDC is in https://docs.uvobservability.com/install/configuration/#authentication.

## 3. Size it

Disk per day = records/s × bytes per record × 86,400, with ~85 B per span,
~raw size ÷ 10 per log line (~45 B for 500 B lines), ~5 B per metric point.
Multiply by retention (30 days default), then provision **1.5×**.

| Rate | Spans/day | Logs/day (500 B) | Metrics/day |
| --- | --- | --- | --- |
| 100/s | 0.7 GB | 0.4 GB | 0.04 GB |
| 1,000/s | 7.3 GB | 3.9 GB | 0.4 GB |
| 10,000/s | 73 GB | 39 GB | 3.9 GB |

Example: 1,000/s each of spans, logs and metrics for 30 days ≈ 520 GB. The
default volume is 50 GiB (`clickhouse.persistence.size`). Chart CPU/memory
defaults suit up to ~1,000 records/s in total; up to ~10,000/s, give
ClickHouse 4–8 vCPU and 16 GiB (`clickhouse.resources`). Beyond that, the user
should talk to Ultraviolet. Details: https://docs.uvobservability.com/install/sizing/

## 4. Verify the chart

```bash
export ULTRVLT_VERSION=0.1.152
curl -fsSLO https://docs.uvobservability.com/cosign.pub
cosign verify --key cosign.pub --insecure-ignore-tlog=true \
  ghcr.io/ultrvlt/charts/ultrvlt:$ULTRVLT_VERSION
```

Use cosign v2. `--insecure-ignore-tlog=true` is expected: signatures are kept
in the registry so they verify offline.

## 5. Create the license and call-home Secrets (skip on the free tier)

On the free tier, skip to step 6 and leave `queryApi.license` and
`queryApi.connected` out of the values file.

With a license, the user received a one-time link with a `.lic` file and (for connected mode)
a call-home token. **Don't ask for the token in the chat.** Have the user
paste it into a hidden prompt:

```bash
kubectl create namespace ultrvlt
kubectl -n ultrvlt create secret generic ultrvlt-license \
  --from-file=valid.lic=./path/to/license.lic

# connected mode only:
read -rs UV_TOKEN && kubectl -n ultrvlt create secret generic ultrvlt-callhome \
  --from-literal=token="$UV_TOKEN"; unset UV_TOKEN
```

A license can be added later: create the Secrets, add `queryApi.license` (and
`connected`) to the values file and upgrade. The data is kept.

## 6. Create the admin credential

`auth.mode: local` (the default) refuses to install without one. Prefer an
argon2id hash so no plaintext is stored. With the `argon2` CLI:

```bash
read -rs UV_PW && export ADMIN_HASH=$(printf %s "$UV_PW" | \
  argon2 "$(openssl rand -hex 16)" -id -t 3 -m 16 -p 2 -e); unset UV_PW
```

If the user runs more than one query-api replica, or renders the chart
without cluster access (Argo CD, `helm template`), also set a stable session
secret: `export SESSION_SECRET=$(openssl rand -hex 32)`.

## 7. Write a values file (no secrets in it)

Put everything that isn't secret in `values-ultraviolet.yaml`, so it can live
in git. For example:

```yaml
clickhouse:
  persistence:
    size: 200Gi
    # storageClass: gp3
queryApi:
  license:                      # leave out on the free tier
    existingSecret: ultrvlt-license
  connected:                    # leave out on the free tier and for offline installs
    enabled: true
    portalUrl: https://desk.uvobservability.com
    tokenSecret: ultrvlt-callhome
  auth:
    admin:
      hashed: true
portalUi:
  ingress:
    enabled: true
    # className: nginx
    host: ultraviolet.example.com
```

For production on AWS (ALB + ACM, gp3, 2 replicas each), start from
https://docs.uvobservability.com/deploy/eks/.

## 8. Preview, then install

```bash
helm template ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version "$ULTRVLT_VERSION" \
  -n ultrvlt -f values-ultraviolet.yaml \
  --set-string queryApi.auth.admin.passwordHash="$ADMIN_HASH" > /tmp/ultraviolet.yaml
```

Summarize what it creates (the ClickHouse operator and its CRDs, the database,
ingest gateway, query API, portal, the schema Job). After the user agrees:

```bash
helm install ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version "$ULTRVLT_VERSION" \
  -n ultrvlt -f values-ultraviolet.yaml \
  --set-string queryApi.auth.admin.passwordHash="$ADMIN_HASH"
  # add: --set queryApi.auth.sessionSecret="$SESSION_SECRET" if you made one
kubectl -n ultrvlt get pods -w
```

The schema Job runs on install and every upgrade.

## 9. Check it

```bash
kubectl -n ultrvlt port-forward deploy/ultrvlt-query-api 8080:8080 >/dev/null &
sleep 2
curl -s localhost:8080/api/v1/ping       # database reachable
curl -s localhost:8080/api/v1/license    # tier, and ui_locked false
kill %1
kubectl -n ultrvlt get ingress
```

The user signs in at `https://<host>` as `admin`. Before the ingress is set up
(or just to try it), `kubectl -n ultrvlt port-forward svc/ultrvlt-portal-ui
8080:8080` and http://localhost:8080 work too. Don't serve the portal to others
over plain HTTP: terminate TLS at the ingress, which sends
`X-Forwarded-Proto: https`, and the session cookie is marked `Secure`.

## 10. Turn on data sources

The pages are empty until telemetry arrives. Before the sensor, check a node's
kernel (BTF and 5.8+):

```bash
kubectl debug node/<node> -it --image=busybox -- sh -c 'uname -r; ls -l /host/sys/kernel/btf/vmlinux'
```

Then add to the values file and upgrade (`--reset-then-reuse-values`, after a
preview):

```yaml
sensor:
  enabled: true
  siteId: prod          # name this location
  clusterId: prod
clusterState:
  enabled: true         # Kubernetes object state for the Infrastructure pages
logCollector:
  enabled: true         # pod logs; set its siteId/clusterId to match the sensor's
```

On Bottlerocket, OpenShift or other SELinux-enforcing nodes, set
`sensor.privileged: true`. If sensors misbehave, use the
`ultraviolet-troubleshoot-sensor` skill.

Apps with an OpenTelemetry SDK can also send OTLP/gRPC to
`ultrvlt-ingest-gateway.ultrvlt:4317`.

## Air-gapped

Mirror the `ghcr.io/ultrvlt/*` images to the internal registry (verify them
first; see https://docs.uvobservability.com/security/verifying-artifacts/), then:

```bash
helm pull oci://ghcr.io/ultrvlt/charts/ultrvlt --version "$ULTRVLT_VERSION"
helm install ultrvlt ./ultrvlt-$ULTRVLT_VERSION.tgz -n ultrvlt -f values-ultraviolet.yaml \
  --set registry=registry.internal.example.com/ultrvlt \
  --set imagePullSecrets[0].name=my-mirror-pull-secret \
  --set-string queryApi.auth.admin.passwordHash="$ADMIN_HASH"
```

Air-gapped installs need a license (the free tier must reach the desk). Leave
connected mode off. Nothing in the core reaches the internet.

## When you're done

Tell the user what was installed, where the values file is (commit it), and
the next steps. On the free tier, start with registering: the **License** page
takes an email and raises 24 hours of data to 3 days. Then: single sign-on
(with a license), retention (**Retention** page), backups (`backup.enabled`),
and the Updates page (`lifecycleManager.enabled`) for verified upgrades.
