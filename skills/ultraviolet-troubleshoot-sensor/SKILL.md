---
name: ultraviolet-troubleshoot-sensor
description: Diagnose the Ultraviolet Observability eBPF sensor — pods restarting or OOMKilled, a node in degraded mode, no traces or encrypted (TLS) traffic missing, slow startup, or data not reaching the core. Use when sensors misbehave, a node or service is missing from Ultraviolet, or someone asks why the sensor uses CPU or memory.
---

# Troubleshoot the sensor

The sensor is a DaemonSet, one pod per node: `ultrvlt-sensor` in the core
chart, `ultrvlt-site-sensor` in a site's chart (prefixed by the release name if
it isn't the default). It's read-only on the cluster and needs no changes to
apps. References: https://docs.uvobservability.com/security/sensor-permissions/
and /install/sizing/#sensors-per-node.

Start by looking, not changing. Everything until "Fixes" is read-only.

## 1. Pod state

```bash
kubectl -n ultrvlt get ds | grep sensor
kubectl -n ultrvlt get pods -o wide | grep sensor
kubectl -n ultrvlt describe pod <sensor-pod> | sed -n '/Last State/,/Ready/p'
```

- **OOMKilled** → step 4.
- **CrashLoopBackOff** → read the previous logs:
  `kubectl -n ultrvlt logs <sensor-pod> --previous --tail=100`.
- **Pending on some nodes** → taints or resources; the default toleration is
  `Exists` (every node), so check `sensor.tolerations` wasn't narrowed.

## 2. Read the startup log

```bash
kubectl -n ultrvlt logs <sensor-pod> | grep -E 'degraded|capturer|discovery sweep|memory limit|attach_failed|denied' | head -40
```

What the lines mean:

| Line | Meaning |
| --- | --- |
| `eBPF unavailable on this node; running in degraded mode … err=…` | The kernel can't run the sensor's programs. The node still meters and reports Kubernetes data, but captures no traffic. The `err` says why (usually missing BTF). Step 3. |
| `capturer loaded capturer=… took_ms=…` | One per protocol, loaded **one after another** before capture starts. One large `took_ms` explains a slow start on its own. |
| `capturer init complete capturers=… took_ms=… go_heap_sys_mb=…` | Startup finished; capture is running. |
| `go memory limit set container_limit_mib=… go_limit_mib=…` | The sensor sized its heap to its container limit, leaving room for kernel memory. |
| `gotls discovery sweep` / `openssl discovery sweep` | Finding programs whose encrypted traffic to read. The first sweep does the most work; later ones are cheap. `attach_failed` with a `hint` means the node's security module refused: step 5. |

## 3. Degraded mode: check the kernel

The sensor needs a **BTF-enabled kernel, 5.8 or newer**:

```bash
kubectl debug node/<node> -it --image=busybox -- sh -c 'uname -r; ls -l /host/sys/kernel/btf/vmlinux'
```

No `/sys/kernel/btf/vmlinux` means the node image lacks BTF: use a node image
that ships it (Amazon Linux 2023, Bottlerocket, current Ubuntu, GKE's
Container-Optimized OS and Ubuntu images do). Nothing in Ultraviolet's
settings fixes a kernel without BTF. Delete the debug pod afterwards
(`kubectl get pods -A | grep node-debugger`).

## 4. Memory

Measured use is ~320 MiB per node, flat with traffic, against a **512 MiB**
default limit. The sensor's memory is its Go heap **plus kernel memory for its
eBPF buffers** (roughly 16 MiB per capturer), which the container is charged
for but `go_heap_sys_mb` doesn't show. So OOMKilled with a modest
`go_heap_sys_mb` is normal accounting, not a leak.

```bash
kubectl -n ultrvlt top pods | grep sensor
```

Nodes most likely to need more are those running hundreds of distinct
programs, which the sensor inspects at startup. Raise the limit (step 6) to
`1Gi` for those.

## 5. Encrypted traffic missing (`attach_failed`, `/proc access denied`)

By default the sensor runs **unprivileged**, with only the capabilities `BPF`,
`PERFMON`, `SYS_PTRACE`, `SYS_ADMIN`, `SYSLOG` and AppArmor unconfined. On
SELinux-enforcing hosts (Bottlerocket, OpenShift) the security module refuses
that set. Set `sensor.privileged: true` there. Policy engines (Kyverno,
Gatekeeper, Pod Security admission at `restricted`/`baseline`) can also block
the pod or strip capabilities: check the namespace's policies and events.

## 6. Nothing reaches the core

```bash
kubectl -n ultrvlt logs <sensor-pod> | grep -iE 'export|otlp|tls|connection' | tail -20
```

- **Core with enrollment on:** the ingest gateway only accepts enrolled
  client certificates, so a sensor on the core's own cluster must send through
  a co-located connector (`sensor.otlpEndpoint` → `ultrvlt-site-aggregator:4317`).
  See the `ultraviolet-add-site` skill, step 3.
- **At a site:** check the connector's logs
  (`kubectl -n ultrvlt logs deploy/ultrvlt-site-aggregator`) — it buffers when
  it can't reach the core, and says why.
- **One service missing, others fine:** the sensor sees TCP and parses HTTP/1,
  HTTP/2, gRPC, PostgreSQL, MySQL, Redis, MongoDB, Kafka and DNS. Other
  protocols still show as connections with bytes, not requests.

## Fixes

Each is a values change on the chart that runs the sensor. Preview, ask, then
upgrade with `--reset-then-reuse-values`:

```bash
helm upgrade ultrvlt oci://ghcr.io/ultrvlt/charts/ultrvlt --version <installed version> \
  -n ultrvlt --reset-then-reuse-values \
  --set sensor.resources.limits.memory=1Gi      # or: --set sensor.privileged=true
```

(For a site, the release is `ultrvlt-site` and the chart `ultrvlt-site`.)

## CPU

~25 millicores idle plus ~0.6 millicores per span/s on that node (100 spans/s
≈ 85 m). Continuous CPU profiling (`sensor.profiling.enabled`, on by default)
costs well under 1% of a CPU. High sensor CPU on a node usually means that node
serves a lot of requests; compare with its span rate on **Core status**.

## Still stuck

Collect, without secrets: the sensor's startup log lines above, `uname -r`,
the node image, the chart version, and `kubectl describe` of one failing pod.
Send them to Ultraviolet support.
