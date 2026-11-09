# Ultraviolet Observability — notes for agents

Use these when helping someone install or run Ultraviolet Observability. They
describe version **0.1.143** (the current stable release). For anything
not covered here, read the docs at https://docs.uvobservability.com.

## What it is

Self-hosted observability for Kubernetes: traces, service maps, logs, metrics,
profiles, alerts, SLOs and investigations. It runs entirely in the customer's
own environment and works air-gapped.

- **The core** is one Helm chart, `oci://ghcr.io/ultrvlt/charts/ultrvlt`: an
  ingest gateway (OTLP on `:4317`), a ClickHouse database (run by a bundled
  operator), the query API and the portal.
- **Data sources** are optional parts of the same chart: an eBPF **sensor** on
  every node (no code changes), a **log collector**, and a **cluster-state**
  collector. Any OpenTelemetry SDK or Collector can also send OTLP.
- **Remote sites** (other clusters or regions) run the small
  `oci://ghcr.io/ultrvlt/charts/ultrvlt-site` chart: a connector that buffers
  to disk and forwards to the core over mutual TLS, plus that site's sensor and
  collectors.
- The customer's license is a signed `.lic` file. **Connected mode** (optional)
  renews it automatically and sends usage metadata only, never telemetry.

## Rules when you act

1. **Confirm the cluster first.** Show `kubectl config current-context` and get
   a yes before changing anything.
2. **Show the change before making it.** `helm template`, `helm diff` (plugin)
   or `--dry-run`, then ask before `helm install`/`upgrade`.
3. **Never ask for secrets in the chat.** Licenses come from a file; tokens and
   passwords are read with `read -s` into a shell variable, or the user runs
   that command themselves. Don't print `helm get values` output that may hold
   them; read single keys instead.
4. **Upgrades use `--reset-then-reuse-values`** (Helm 3.14+), never plain
   `--reuse-values`, which skips settings that newer versions add.
5. **Verify signatures** before installing: releases are cosign-signed and
   verify offline with `https://docs.uvobservability.com/cosign.pub` and
   `--insecure-ignore-tlog=true` (expected; signatures live in the registry).
6. **Keep the portal private** (VPN, private network). Only the ingest
   endpoint is ever exposed to remote sites.

## Facts that trip people up

- The portal's session cookie is `Secure`: sign-in over plain HTTP or a bare
  `kubectl port-forward` appears to work and then bounces back to the login
  page. Serve it over HTTPS.
- The core boots without a license: it keeps ingesting, with the portal locked.
- `queryApi.auth.mode: local` is fail-closed: the install errors without an
  admin credential.
- Turning on remote-site enrollment switches the ingest gateway to mutual TLS,
  so the core's own sensor and collectors need a co-located connector (see the
  add-site skill) or they stop delivering.
- The sensor needs a BTF-enabled kernel 5.8+. Without one it keeps running in
  "degraded mode" (metering and Kubernetes data only) rather than crashing.

## Skills

- `skills/ultraviolet-install` — first install of the core.
- `skills/ultraviolet-add-site` — connect a remote cluster.
- `skills/ultraviolet-upgrade` — upgrade the core or a site.
- `skills/ultraviolet-troubleshoot-sensor` — sensor not capturing, restarting, or slow to start.
- `skills/ultraviolet-automatic-fixes` — let runbooks act on their own, scoped per site.

## Looking at live data

The core has a read-only MCP server for agents (`queryApi.mcp.enabled: true`,
then an API token from **Access → API tokens**; the address is the portal's
plus `/api/v1/mcp`). With it you can check health, traces, logs and deploys
while you work. See https://docs.uvobservability.com/guides/ai-agents/.
