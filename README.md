# opentelemetry-collector-config-for-eks-fargate

OpenTelemetry Collector configuration for monitoring **whole-cluster workloads on Amazon EKS Fargate** — without DaemonSets, host access, or kubelet reachability.

It runs as a single **deployment-mode** (gateway) collector that receives OTLP from your apps, scrapes per-container cAdvisor metrics through the Kubernetes API server proxy, pulls cluster state from the API server, collects MySQL metrics, applies tail sampling to traces, and exports everything to Grafana Cloud.

Visual reference (otelbin): https://www.otelbin.io/s/066285ca263d7b02df624e812ef384460e22a7f9

---

## Why this exists

EKS Fargate runs every pod in its own micro-VM, which removes the building blocks most Kubernetes monitoring stacks rely on:

- **No DaemonSets** — there is no shared node to place one agent per host on.
- **No host access** — no HostPath volumes, no privileged containers, no `/var/log/containers`.
- **No direct kubelet reachability** — a pod cannot dial the kubelet on its node.

The usual `kubeletMetrics` preset (the `kubeletstats` receiver) is therefore **disabled** here: it is node-local by design and expects `mode: daemonset`. Instead, this config collects node/container metrics centrally by scraping the kubelet's cAdvisor endpoint **through the API server proxy**, so one collector sees the entire cluster.

---

## Architecture

```
Sources (Fargate pods)                Collector (gateway)              Backend
────────────────────────              ───────────────────              ───────
App pods (OTLP SDK) ──push──┐
                            │
API server proxy ───────────┤
  /proxy/metrics/cadvisor   ├──▶  otel-collector  ──OTLP/HTTP──▶  Grafana Cloud
                            │     (deployment,                    (primary + backup
k8s_cluster receiver ───────┤      1 replica)                      region)
                            │
mysql receiver ─────────────┘

Logs: handled separately by Fargate's built-in Fluent Bit (not this collector).
```

---

## How cAdvisor scraping works (the Fargate workaround)

The `prometheus` receiver discovers nodes and rewrites each scrape target so the request goes to the API server instead of the (unreachable) kubelet:

1. **`kubernetes_sd_configs: role: node`** — discover every node as a scrape target.
2. **`labelmap`** — copy node labels onto the target (enrichment).
3. **Rewrite `__address__`** → `kubernetes.default.svc:443` (point every target at the API server).
4. **Rewrite `__metrics_path__`** → `/api/v1/nodes/$NODE/proxy/metrics/cadvisor`.

Each scrape becomes:

```
GET https://kubernetes.default.svc:443/api/v1/nodes/<node>/proxy/metrics/cadvisor
```

The API server's node-proxy subresource forwards the request to that node's kubelet and streams cAdvisor metrics back. Authentication uses the pod's mounted service-account token (`bearer_token_file` + `tls_config.ca_file`).

Because the raw output is Prometheus-named cAdvisor counters, the pipeline then:

- **`transform/cadvisor`** — renames `container_*` → `k8s.container.*`, extracts `k8s.pod.uid` / `k8s.container.id` from the cgroup `id` path, and promotes them to resource attributes.
- **`filter/cadvisor`** — keeps only container-level datapoints.
- **`filter/cadvisor_metrics_allowlist`** — keeps only the metrics the dashboards use.
- **`k8s_attributes`** — attaches deployment/owner metadata using the promoted resource attributes.

> **Note:** Not all cAdvisor metrics are populated on Fargate-backed nodes (e.g. host-level load average). None of those are in the allowlist, so this is expected.

---

## Versions

| Component | Version |
| --- | --- |
| Helm chart `open-telemetry/opentelemetry-collector` | `0.147.1` |
| Collector image `otel/opentelemetry-collector-contrib` | `0.150.1` |
| Mode | `deployment` (gateway), `replicaCount: 1` |

---

## Pipelines

Defined in `helm/values.yaml` under `config.service.pipelines`:

| Pipeline | Receivers | Exporter |
| --- | --- | --- |
| `traces` | `otlp` | Grafana Cloud (with `tail_sampling`) |
| `metrics/http` | `otlp` | Grafana Cloud (backup) |
| `metrics/business` | `otlp` | Grafana Cloud |
| `metrics/nodejs_runtime` | `otlp` | Grafana Cloud |
| `metrics/envoy` | `otlp` | Grafana Cloud |
| `metrics/kubernetes` | `k8s_cluster` | Grafana Cloud |
| `metrics/cadvisor` | `prometheus` | Grafana Cloud |
| `metrics/database` | `mysql` | Grafana Cloud |
| `logs` | `otlp`, `k8sobjects` | Grafana Cloud |

Each metric pipeline uses a strict allowlist (`filter/*_allowlist`) to control cardinality, plus `resource/add_cluster` and `k8s_attributes` for enrichment.

### Tail sampling policy (traces)

| Priority | Policy | Keeps |
| --- | --- | --- |
| 1 | `status_code` | All `ERROR` traces |
| 2 | `latency` | Traces slower than 1000 ms |
| 3 | `probabilistic` | 25% of baseline traffic |

---

## RBAC

The collector's `clusterRole` (in `values.yaml`) grants exactly what the two API-server-dependent features need:

- **cAdvisor proxy scraping:** `nodes/proxy`, `nodes/metrics` (`get`, `list`) and the `/metrics/cadvisor` nonResourceURL.
- **`k8s_attributes` + `k8s_cluster`:** `get`/`list`/`watch` on `nodes`, `namespaces`, `pods`, `apps` (deployments/replicasets/daemonsets/statefulsets), and `batch` (jobs/cronjobs).

> Without `nodes/proxy`, the API server returns `403 Forbidden (subresource=proxy)` and cAdvisor scraping fails.

---

## Scaling notes

This is a single-replica gateway. Two things prevent a naive `replicaCount` bump:

- **`tail_sampling`** requires all spans of a trace to land on the same instance.
- **`prometheus` / `k8s_cluster`** are single-instance scrapers; running N copies duplicates their data.

To scale, split into tiers: stateless OTLP **agent** collectors → a **`loadbalancing`-exporter** layer that routes by trace ID → a **sampling gateway** tier, while keeping the prometheus/k8s_cluster scrapers on a dedicated single-replica collector.
