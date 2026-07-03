# Skill: Cluster Node and Network Performance Analysis

## Goal

Analyze node-level resource pressure (CPU/memory/disk/network) and network-path performance (pod-to-pod, DNS, service mesh) using historical Prometheus data, to identify performance bottlenecks that a point-in-time `kubectl top`/metric-server snapshot would miss (e.g. periodic saturation, DNS latency spikes, node-level noisy-neighbor effects).

## Access Model

Uses the approved read-only Prometheus MCP server against the VictoriaMetrics backend described in `component-contexts/context.observability.md`. Never port-forward or connect directly to `vmselect`/`vminsert`/`vmalert`.

## Inputs

- `target_namespaces` — optional for this skill; node-level metrics are cluster-wide by nature, but per-pod network metrics should still be scoped to allowed application namespaces plus relevant infra (e.g. `istio-system` ingress gateways, `kube-system` CoreDNS) for context.
- `time_window` — from the runtime input contract in `component-contexts/context.observability.md`. Default to `last 6h`; use a longer window (confirm first) to catch periodic patterns (e.g. daily batch jobs).

## Read-Only Evidence Collection

Representative PromQL:

- Node CPU pressure: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
- Node memory pressure: `1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)`
- Node network throughput/errors: `rate(node_network_receive_bytes_total[5m])`, `rate(node_network_receive_errs_total[5m])`
- Pod-level network throughput (cAdvisor): `rate(container_network_receive_bytes_total{namespace=~"<ns>"}[5m])`
- CoreDNS latency: `histogram_quantile(0.99, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))`
- Istio mesh latency/connection churn (if scraped for target namespaces): `istio_tcp_connections_opened_total`, `istio_request_duration_milliseconds_bucket`
- Cross-reference with Kubernetes MCP: node `Ready`/pressure conditions, pod scheduling failures (`FailedScheduling` events) during the same window.

If mesh or DNS metrics are not scraped for the relevant scope, record a `data_gap` finding.

## Analysis Logic

- Flag `node_saturation` when CPU/memory pressure sustains above a high-utilization threshold (e.g. >85%) for a material portion of the window, especially if correlated with pod scheduling failures or throttling on that node.
- Flag `network_errors` when interface error/drop rates are non-trivial relative to throughput.
- Flag `dns_latency` when CoreDNS p99 latency is elevated, since this commonly manifests as intermittent app-level timeouts that are hard to attribute otherwise.
- Flag `mesh_saturation` when TCP connection churn or mesh request latency spikes correlate with app-level latency regressions (cross-reference `skills/slo-latency-error-analysis/SKILL.md` if run in the same session).
- Flag `data_gap` when a needed metric is unavailable for the scope requested.

## Output Format

- `network_performance_summary`
- `findings[]` with fields:
  - `scope` (`node:<name>` or `namespace:<ns>` or `service:<name>`)
  - `metric`
  - `observed_value`
  - `issue_type` (`node_saturation`, `network_errors`, `dns_latency`, `mesh_saturation`, `data_gap`)
  - `evidence` (per `skills/reporting.md` evidence format)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `recommendation`

## Guardrails

- No automatic node cordon/drain or network policy changes — this skill is analysis-only.
- Respect the 30-day/unscoped-query confirmation gate in `AGENTS.md`; prefer scoping to specific nodes/namespaces over unscoped cluster-wide range queries where possible.
