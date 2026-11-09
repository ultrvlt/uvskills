---
name: ultraviolet-upgrade
description: Upgrade an Ultraviolet Observability core or remote site to a newer release safely — verify the signed chart, preview the change, keep existing settings with --reset-then-reuse-values, order connector and core upgrades, and check the result. Use when someone wants to upgrade, update or move Ultraviolet Observability to a new version, or change its Helm values.
---

# Upgrade Ultraviolet Observability

The current stable release is **0.1.143**. The portal's **Updates**
page (with `lifecycleManager.enabled`) shows the installed version against the
channel, with release notes, and can run verified upgrades itself. This skill
is for upgrading by hand with Helm, or changing values.

## 1. Where are we?

```bash
kubectl config current-context
helm list -A | grep -E 'ultrvlt'          # every Ultraviolet release and its chart version
helm version                              # 3.14+ needed for --reset-then-reuse-values
```

Confirm the context and the target version with the user. Read the release
notes for every version in between:
`https://github.com/ultrvlt/ultrvlt/releases/tag/v<version>` (also linked
from the Updates page).

## 2. Verify the new chart

```bash
export ULTRVLT_VERSION=0.1.143
curl -fsSLO https://docs.uvobservability.com/cosign.pub
cosign verify --key cosign.pub --insecure-ignore-tlog=true ghcr.io/ultrvlt/charts/ultrvlt:$ULTRVLT_VERSION
cosign verify --key cosign.pub --insecure-ignore-tlog=true ghcr.io/ultrvlt/charts/ultrvlt-site:$ULTRVLT_VERSION
```

## 3. Order

1. A **connector on the core's own cluster** (`ultrvlt-site` beside the core)
   first: the core's producers trust its CA and send through it.
2. The **core** (`ultrvlt` chart).
3. **Remote sites**, each in its owners' maintenance window. Their connectors
   buffer to disk, so a restart loses nothing. Keep sites on the core's
   version where you can.

## 4. Preview

Always keep existing settings with `--reset-then-reuse-values`: it reapplies
what the user set **and** picks up defaults the new version adds. Plain
`--reuse-values` skips those new defaults and can leave new features unset.

Preview with the `helm diff` plugin if it's installed:

```bash
helm diff upgrade ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version "$ULTRVLT_VERSION" \
  -n ultrvlt --reset-then-reuse-values
```

Otherwise render old and new and compare:

```bash
helm get manifest ultrvlt -n ultrvlt > /tmp/uv-before.yaml
helm upgrade ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version "$ULTRVLT_VERSION" \
  -n ultrvlt --reset-then-reuse-values --dry-run=server 2>/dev/null \
  | sed -n '/^---/,$p' > /tmp/uv-after.yaml
diff -u /tmp/uv-before.yaml /tmp/uv-after.yaml | less
```

Summarize the diff for the user — images, new resources, anything removed —
and get a yes. (A diff can include Secret contents; summarize, don't paste
them.)

**Changing or removing a setting:** add `--set key=value` or `-f values.yaml`
to the same command. To **remove** an override, don't use `--set key=null`;
export the current user values (`helm get values ultrvlt -n ultrvlt -o yaml >
values-current.yaml`, kept out of the chat and git if it holds secrets), delete
the key, and upgrade with `--reset-values -f values-current.yaml`.

## 5. Upgrade and watch

```bash
helm upgrade ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version "$ULTRVLT_VERSION" \
  -n ultrvlt --reset-then-reuse-values --wait --timeout 15m
kubectl -n ultrvlt get pods
kubectl -n ultrvlt get jobs        # the schema Job runs on every upgrade
```

While the database restarts, the ingest gateways buffer to disk and catch up
afterwards; the portal is unavailable for that time (unless replicated
storage is on).

For a site:

```bash
helm upgrade ultrvlt-site oci://ghcr.io/ultrvlt/charts/ultrvlt-site --version "$ULTRVLT_VERSION" \
  -n ultrvlt --reset-then-reuse-values --wait
```

## 6. Check it

```bash
kubectl -n ultrvlt exec deploy/ultrvlt-query-api -- wget -qO- localhost:8080/api/v1/ping
kubectl -n ultrvlt exec deploy/ultrvlt-query-api -- wget -qO- localhost:8080/api/v1/license
```

Then in the portal: **Sites** shows every site seen in the last minute or two,
**Core status** shows data arriving, and the Services page has fresh traffic.
If sensors restart after the upgrade, use the `ultraviolet-troubleshoot-sensor`
skill.

If something is wrong, `helm history ultrvlt -n ultrvlt` and
`helm rollback ultrvlt <revision> -n ultrvlt` put the previous workloads back;
tell the user to let Ultraviolet support know what happened.

## Notes

- A failed schema Job blocks the upgrade: read its logs
  (`kubectl -n ultrvlt logs job/<name>`) before retrying.
- The Updates page's one-click upgrades change workloads only; a release that
  changes the database operator or CRDs is refused there and must be applied
  with Helm as above.
