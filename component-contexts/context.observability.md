# Observability / Metrics Backend Context (MCP Snapshot)

## Snapshot Metadata

- Date (UTC): `2026-06-24`
- Namespace hosting the stack: `observability` (see `component-contexts/context.kubernetes.md` section 3.1)
- Backend: **VictoriaMetrics** (Prometheus remote-write/PromQL-compatible), not vanilla Prometheus.
  - Write path: `vminsert-o11y-victoriametrics`
  - Query path: `vmselect-o11y-victoriametrics` (this is the effective "Prometheus API" surface)
  - Storage: `vmstorage-o11y-victoriametrics` (StatefulSet)
  - Alert evaluation: `vmalert-o11y-victoriametrics`
- Tracing (context only, not queried by these skills): `o11y-jaeger-collector`, `o11y-jaeger-agent`
- Scrape/export agents feeding the backend: `o11y-kube-state-metrics`, `o11y-prometheus-node-exporter` (DaemonSet)

Treat "Prometheus" in skill files under `skills/` as shorthand for "the PromQL-compatible query API exposed by `vmselect-o11y-victoriametrics`," accessed exclusively through the approved Prometheus MCP server below.

## Access Model (Mandatory)

- Access is **only** through the approved, read-only Prometheus MCP server. This supersedes the earlier blanket prohibition that used to live in `skills/resource-utilization-inspection/SKILL.md`.
- No local `kubectl port-forward` to `vmselect`/`vminsert`/`vmalert` from an operator machine. No direct Service/Ingress access outside the MCP server.
- Permitted MCP operations: instant query (`query`), range query (`query_range`), label listing (`label_names`, `label_values`), metric/series metadata (`series`, `metadata`, `targets` read-only status).
- Forbidden regardless of transport: remote-write, admin API (`/api/v1/admin/*`, snapshot, delete-series, TSDB flush), alerting/recording rule create-update-delete, config reload.
- Query cost guardrails: prefer bounded time ranges and label-scoped selectors (namespace/workload) over unscoped cluster-wide queries; avoid `step` values that would return excessive samples.

## Runtime Input Contract (Mandatory)

Before running any Prometheus-backed skill, ask the user for:
- `target_namespaces` — reuse the same validated set from `component-contexts/context.kubernetes.md` (application namespaces only; do not run performance/SLO queries scoped to system/infra namespaces as the primary subject).
- `time_window` — required. If not provided, propose a default (`last 1h` for live diagnosis, `last 7d` for trend/capacity analysis) and ask the user to confirm before running.

Rules:
1. If `time_window` exceeds 30 days or the query is not scoped to specific namespaces/workloads, require explicit user confirmation before executing (same spirit as the comprehensive-inspection confirmation gate in `context.kubernetes.md`).
2. Never run Prometheus-backed analysis against system/infra namespaces as the inspection subject; infra-level node/network metrics are fine (they are cluster-wide by nature), but per-service SLO/latency analysis is restricted to allowed application namespaces.
3. If a query returns no data (metric absent, target down, or namespace unscraped), report it as a `data_gap` finding rather than assuming healthy state.

## Metric Families Available (Observed/Expected)

- **kube-state-metrics**: `kube_pod_container_resource_requests`, `kube_pod_container_resource_limits`, `kube_pod_status_phase`, `kube_deployment_status_replicas`, `kube_pod_container_status_restarts_total`.
- **node-exporter**: `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes`, `node_network_receive_bytes_total`, `node_network_transmit_bytes_total`, `node_network_receive_errs_total`.
- **cAdvisor/kubelet** (via VictoriaMetrics scrape): `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, `container_network_receive_bytes_total`.
- **Istio mesh** (if scraped from `istio-system` sidecars/gateways): `istio_requests_total`, `istio_request_duration_milliseconds_bucket`, `istio_tcp_connections_opened_total`.
- **CoreDNS**: `coredns_dns_request_duration_seconds_bucket`, `coredns_dns_responses_total`.
- **vmalert**: `ALERTS`, `ALERTS_FOR_STATE` (for reading currently firing/pending alert state, read-only).

## Open Items for Operator Confirmation

- Confirm the exact Prometheus MCP server binding/tool names available in the runtime (this file assumes a generic `query` / `query_range` / `label_values` tool surface).
- Confirm VictoriaMetrics retention window (bounds how far back `time_window` can meaningfully go).
- Confirm whether Istio mesh metrics are actually scraped for application namespaces (needed for `cluster-network-performance` skill) or only for `istio-system` itself.
- Confirm per-namespace access scoping, if any, enforced at the MCP layer vs. relying on agent-side discipline.
