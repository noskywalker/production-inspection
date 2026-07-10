---
name: cluster-network-performance
description: "Analyze cluster-wide network performance including node-level resource pressure, DNS resolution latency, service mesh health, conntrack table usage, cross-zone traffic patterns, TCP retransmission, network policy effectiveness, and ingress/egress gateway performance using historical Prometheus data."
---

# Skill: Cluster Node & Network Performance Analysis

## Goal

Analyze node-level resource pressure (CPU/memory/disk/network) and network-path performance (DNS, service mesh, conntrack, cross-zone traffic, TCP reliability, network policies, gateway performance) using historical Prometheus data. Identify performance bottlenecks that a point-in-time `kubectl top`/metric-server snapshot would miss: periodic saturation, DNS latency spikes, node-level noisy-neighbor effects, conntrack exhaustion, mesh overhead, and cross-zone latency penalties.

## Access Model

Uses the approved read-only Prometheus MCP server against the VictoriaMetrics backend described in `component-contexts/context.observability.md`. Never port-forward or connect directly to `vmselect`/`vminsert`/`vmalert`. Cross-reference with Kubernetes MCP for node conditions and events.

## Inputs

- `target_namespaces` — optional for this skill; node-level metrics are cluster-wide by nature, but per-pod network metrics should be scoped to allowed application namespaces plus relevant infra (e.g. `istio-system` ingress gateways, `kube-system` CoreDNS).
- `time_window` — from the runtime input contract in `component-contexts/context.observability.md`. Default to `last 6h`; use a longer window (confirm first) to catch periodic patterns (e.g. daily batch jobs).
- Optional `focus_nodes` — specific node names to narrow analysis.

## Metric Verification (Pre-Check)

Before running analysis, verify metric availability:
- `label_values(node_cpu_seconds_total, instance)` — confirm node-exporter is running on all nodes
- `label_values(coredns_dns_request_duration_seconds_bucket, ...)` — confirm CoreDNS metrics are scraped
- `label_values(istio_requests_total, namespace)` — confirm mesh metrics availability
- `label_values(container_network_receive_bytes_total, namespace)` — confirm cAdvisor network metrics
- Record any metric unavailability as `data_gap` before proceeding.

## Inspection Check Categories

### 1. Node Resource Saturation

Collect via Prometheus MCP:
- Node CPU utilization: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
- Node memory pressure: `1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)`
- Node disk usage: `1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})`
- Node load average: `node_load1`, `node_load5`, `node_load15`
- Node CPU steal (if cloud): `avg by (instance) (rate(node_cpu_seconds_total{mode="steal"}[5m])) * 100`

Analysis:
- Flag `node_cpu_saturation` (S1) when CPU utilization sustains above 85% for >25% of the `time_window` — pods on that node will experience CPU throttling.
- Flag `node_memory_saturation` (S1) when memory usage sustains above 85% — risk of system OOM and kubelet eviction.
- Flag `node_disk_saturation` (S2) when root filesystem usage is above 80% — may impact kubelet operation, container logs, and ephemeral storage.
- Flag `cpu_steal_high` (S1) when CPU steal time is above 5% on cloud nodes — the hypervisor is oversubscribed, no amount of right-sizing will help; consider node migration.
- Flag `load_avg_anomaly` (S2) when `node_load5` exceeds CPU core count by >2x for a sustained period — system is heavily over-subscribed.
- Cross-reference with Kubernetes MCP: node `Ready`/pressure conditions, `FailedScheduling` events during the same window.

### 2. Node Network Throughput & Errors

Collect via Prometheus MCP:
- Network receive/transmit throughput: `rate(node_network_receive_bytes_total[5m])`, `rate(node_network_transmit_bytes_total[5m])`
- Network error rate: `rate(node_network_receive_errs_total[5m])`, `rate(node_network_transmit_errs_total[5m])`
- Network drop rate: `rate(node_network_receive_drop_total[5m])`, `rate(node_network_transmit_drop_total[5m])`
- Per-interface breakdown if multiple interfaces exist

Analysis:
- Flag `network_errors` (S2) when interface error rate is non-trivial relative to throughput (>0.01% of packets) — may indicate NIC issues, cabling, or switch problems.
- Flag `packet_drops` (S2) when packet drop rate is elevated — may indicate buffer exhaustion, rate limiting, or network policy enforcement overhead.
- Flag `bandwidth_saturation` (S2) when network throughput sustains above 80% of interface capacity (typically 1Gbps or 10Gbps) — may cause latency for network-intensive workloads.
- Flag `network_asymmetry` (S3) when receive vs. transmit throughput is significantly imbalanced — may indicate misconfigured traffic patterns.

### 3. DNS Resolution Performance

Collect via Prometheus MCP:
- CoreDNS query latency p99: `histogram_quantile(0.99, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))`
- CoreDNS query latency per zone: `histogram_quantile(0.99, sum by (zone, le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))`
- CoreDNS error rate: `sum by (rcode) (rate(coredns_dns_responses_total[5m]))` — look for `NXDOMAIN`, `SERVFAIL`, `REFUSED`
- CoreDNS cache hit rate: `sum(rate(coredns_cache_hits_total[5m])) / sum(rate(coredns_cache_requests_total[5m]))`
- CoreDNS throughput: `sum(rate(coredns_dns_requests_total[5m]))`
- CoreDNS panic count: `sum(rate(coredns_panic_count_total[5m]))`

Analysis:
- Flag `dns_latency_high` (S2) when CoreDNS p99 latency exceeds 50ms — DNS resolution is a critical path for service discovery, high latency causes intermittent application timeouts.
- Flag `dns_nxdomain_high` (S3) when NXDOMAIN responses exceed 1% of total queries — applications may be querying non-existent service names (misconfiguration).
- Flag `dns_servfail` (S2) when SERVFAIL responses are non-zero — CoreDNS is unable to resolve queries, may indicate upstream resolver issues.
- Flag `dns_cache_hit_low` (S3) when cache hit rate is below 80% — CoreDNS is forwarding too many queries upstream, increasing latency.
- Flag `dns_throughput_saturation` (S2) when CoreDNS query rate is high and latency is increasing — may need more CoreDNS replicas (check `kube-system` CoreDNS Deployment).
- Flag `dns_panic` (S1) when CoreDNS panic count is non-zero — CoreDNS is crashing/restarting, check `skills/k8s-performance-inspection/SKILL.md` for restart patterns.
- Cross-reference: DNS latency spikes commonly manifest as intermittent app-level timeouts that are hard to attribute — correlate with `skills/slo-latency-error-analysis/SKILL.md` latency findings.

### 4. Service Mesh Health

Collect via Prometheus MCP and Kubernetes MCP:
- Istio control plane health (Kubernetes MCP): `kubectl_get pods -n istio-system -o wide` — istiod pod status
- Istio sidecar proxy resource usage: `container_cpu_usage_seconds_total{namespace=~"<ns>", container="istio-proxy"}`, `container_memory_working_set_bytes{namespace=~"<ns>", container="istio-proxy"}`
- Mesh request latency (if scraped for target namespaces): `istio_request_duration_milliseconds_bucket`
- TCP connection churn: `rate(istio_tcp_connections_opened_total[5m])`, `rate(istio_tcp_connections_closed_total[5m])`
- Pilot/envoy push rate: `pilot_xds_pushes`, `pilot_xds_rejected` (if scraped)
- Envoy proxy admin stats (if scraped): `envoy_cluster_upstream_rq_pending_overflow`, `envoy_cluster_circuit_breakers_default_cx_pool_open`

Analysis:
- Flag `mesh_control_plane_degraded` (S1) when istiod pods are not all `Ready` or have high restart counts — mesh configuration distribution will be impacted.
- Flag `sidecar_overhead_high` (S2) when istio-proxy sidecar CPU usage exceeds 10% of the application container's CPU — mesh overhead is significant relative to application work.
- Flag `sidecar_memory_leak` (S2) when istio-proxy memory usage is monotonically increasing over the `time_window` — known Envoy memory leak patterns.
- Flag `connection_churn` (S2) when TCP connection open/close rate is high and correlated with application latency spikes — connection pool exhaustion or misconfigured keepalive.
- Flag `circuit_breaker_open` (S2) when `envoy_cluster_circuit_breakers_default_cx_pool_open` is non-zero — upstream connections are exhausted, requests are being rejected.
- Flag `pilot_push_rejected` (S3) when `pilot_xds_rejected` is non-zero — Envoy is rejecting configuration pushes, may indicate invalid config.
- Flag `mesh_latency_overhead` (S3) when mesh-injected request latency is materially higher than direct pod-to-pod latency — sidecar processing overhead.

### 5. Connection Tracking (Conntrack) Analysis

Collect via Prometheus MCP:
- Conntrack entries: `node_nf_conntrack_entries` (if scraped by node-exporter)
- Conntrack max: `node_nf_conntrack_entries_limit` (if scraped)
- Conntrack usage ratio: `node_nf_conntrack_entries / node_nf_conntrack_entries_limit`

Analysis:
- Flag `conntrack_near_limit` (S1) when conntrack table usage exceeds 80% of max — new connections will be dropped, causing intermittent connection failures.
- Flag `conntrack_exhaustion` (S0) when conntrack usage reaches 100% — all new connections are dropped, widespread network failures.
- Flag `conntrack_trend_up` (S2) when conntrack entries are monotonically increasing over the `time_window` — may indicate connection leak in an application.
- Cross-reference: conntrack exhaustion often manifests as random connection timeouts across multiple services — correlate with `skills/slo-latency-error-analysis/SKILL.md` timeout findings.

### 6. Cross-Zone Traffic Patterns

Collect via Prometheus MCP:
- Inter-zone traffic (if zone labels are available): `sum by (source_availability_zone, destination_availability_zone) (rate(istio_requests_total[5m]))` or `sum by (source_zone, destination_zone) (rate(container_network_receive_bytes_total[5m]))`
- Cross-zone latency: compare `istio_request_duration_milliseconds_bucket` for same-zone vs. cross-zone traffic (if zone labels are available)
- Node zone distribution: `kubectl_get nodes -L topology.kubernetes.io/zone` (Kubernetes MCP)

Analysis:
- Flag `cross_zone_traffic_high` (S2) when a significant portion (>50%) of pod-to-pod traffic crosses availability zones — incurs cloud egress charges and adds latency.
- Flag `topology_anti_pattern` (S3) when services that should be co-located (e.g. `pinjam` gateway calling `id-payment`) are consistently deployed in different zones — topology spread constraints may be missing or misconfigured.
- Flag `cross_zone_latency_penalty` (S3) when cross-zone p99 latency is >2x same-zone p99 latency — expected to some degree, but extreme differences may indicate underlying network issues.
- Reference zone topology from `context.kubernetes.md`: zones `ap-southeast-5a`, `ap-southeast-5b`, `ap-southeast-5c`.

### 7. TCP Retransmission & Connection Reset Analysis

Collect via Prometheus MCP:
- TCP retransmits: `rate(node_netstat_Tcp_RetransSegs[5m])`
- TCP resets received: `rate(node_netstat_TcpExt_TCPSynRetrans[5m])`
- TCP resets sent: `rate(node_netstat_Tcp_OutRsts[5m])`
- TCP connection failures: `rate(node_netstat_Tcp_ActiveOpens[5m])` vs. `rate(node_netstat_Tcp_AttemptFails[5m])`

Analysis:
- Flag `tcp_retransmission_high` (S2) when TCP retransmit rate is elevated relative to connection rate — indicates network packet loss, congestion, or MTU issues.
- Flag `tcp_syn_retransmit` (S2) when SYN retransmits are high — connection establishment is failing, may indicate backend unavailability, firewall rules, or conntrack exhaustion.
- Flag `tcp_reset_storm` (S2) when TCP reset rate is abnormally high — may indicate crashing backends, load balancer health check failures, or connection pool misconfiguration.
- Flag `connection_attempt_failures` (S2) when `AttemptFails / ActiveOpens` ratio is elevated — a significant portion of outbound connection attempts are failing.

### 8. Network Policy Effectiveness

Collect via Kubernetes MCP:
- `kubectl_get networkpolicy -n <ns> -o json` — all NetworkPolicies in scope
- `kubectl_get ns <ns> --show-labels` — namespace labels (used by network policy selectors)

Analysis:
- Flag `no_network_policy` (S3) for application namespaces with zero NetworkPolicies — all pods can communicate with all other pods, no micro-segmentation.
- Flag `default_deny_missing` (S3) when no default-deny-all NetworkPolicy exists — pods are reachable by any pod in the cluster.
- Flag `policy_too_permissive` (S3) when NetworkPolicies use broad selectors (e.g. `podSelector: {}` with no namespaceSelector) — effectively no restriction.
- Flag `policy_orphaned` (S3) when a NetworkPolicy selector matches no pods — stale policy from a removed workload.
- Note: NetworkPolicy enforcement requires a CNI plugin that supports it (e.g. Calico, Cilium). If the CNI does not enforce NetworkPolicies, policies are no-ops — check CNI plugin from `context.kubernetes.md` (observed: `terway-eniip` DaemonSet in `kube-system`).

### 9. Ingress & Egress Gateway Performance

Collect via Prometheus MCP and Kubernetes MCP:
- Ingress gateway resource usage: `container_cpu_usage_seconds_total{namespace="istio-system", pod=~"istio-ingressgateway.*"}`
- Ingress gateway request rate: `sum by (destination_service) (rate(istio_requests_total{source_namespace="istio-system", destination_namespace=~"<ns>"}[5m]))`
- Ingress gateway latency: `histogram_quantile(0.99, sum by (le) (rate(istio_request_duration_milliseconds_bucket{source_namespace="istio-system"}[5m])))`
- Egress gateway throughput: `rate(istio_tcp_connections_opened_total{source_namespace=~"<ns>", destination_namespace="istio-system"}[5m])`
- Gateway pod status (Kubernetes MCP): `kubectl_get pods -n istio-system -l istio=ingressgateway -o wide`

Analysis:
- Flag `ingress_gateway_saturation` (S1) when ingress gateway CPU usage is above 80% — gateway is a single point of failure for all incoming traffic.
- Flag `ingress_gateway_latency` (S2) when gateway p99 latency exceeds 100ms — adds latency to every inbound request.
- Flag `gateway_single_replica` (S1) when the ingress gateway runs with only 1 replica — no redundancy for the cluster's front door.
- Flag `egress_gateway_bottleneck` (S2) when egress gateway connection rate is high and correlated with application timeout patterns.
- Flag `gateway_resource_unbounded` (S3) when gateway containers have no resource limits — gateway can be killed by OOM under traffic spikes.

### 10. MTU & Path MTU Issues

Collect via Prometheus MCP:
- Packet fragmentation rate: `rate(node_network_receive_frame_total[5m])` (frame errors may indicate MTU mismatch)
- Large packet loss on specific interfaces

Analysis:
- Flag `mtu_mismatch_suspected` (S3) when frame errors are elevated and correlate with cross-zone or cross-network traffic — path MTU discovery may be failing, causing packet fragmentation and retransmission.
- Note: This is an indirect signal; confirm with `exec`-based `ping -M do -s <size>` testing if approved (requires explicit approval per `AGENTS.md`).

## Output Format

- `network_performance_summary` — overall cluster network posture (node count, zones, CNI, mesh status, DNS health, conntrack usage)
- `findings[]` with fields:
  - `scope` (`node:<name>` or `namespace:<ns>` or `service:<name>` or `cluster-wide`)
  - `check_category` (1-10 above)
  - `metric`
  - `observed_value`
  - `baseline_comparison` (vs. historical if available)
  - `issue_type` (from the analysis flags above)
  - `evidence` (per `skills/reporting.md` evidence format)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `confidence` (`high`/`medium`/`low`)
  - `recommendation`
  - `owner_hint`
- `dns_health{}` — `{p99_latency, error_rate, cache_hit_rate, throughput, panic_count}`
- `mesh_health{}` — `{control_plane_status, sidecar_overhead_avg, connection_churn_rate, circuit_breaker_status}`
- `conntrack_status{}` — `{entries, max, usage_pct, trend}`
- `cross_zone_traffic{}` — `{same_zone_pct, cross_zone_pct, cross_zone_latency_penalty}`
- `gateway_health{}` — `{ingress_cpu, ingress_latency, ingress_replicas, egress_connections}`
- `data_gaps[]` — metrics that were unavailable

## Guardrails

- No automatic node cordon/drain, network policy changes, or mesh configuration changes — this skill is analysis-only.
- Respect the 30-day/unscoped-query confirmation gate in `AGENTS.md`; prefer scoping to specific nodes/namespaces over unscoped cluster-wide range queries.
- `exec`-based network testing (e.g. `ping`, `curl`, `tcpdump`) requires explicit human approval per `AGENTS.md`.
- Use coarse query steps for range queries (e.g. `1m` for 6h window, `5m` for 24h window) to keep result size manageable.

## Cross-Skill References

- Node saturation findings correlate with `skills/resource-utilization-inspection/SKILL.md` (node pressure) and `skills/k8s-performance-inspection/SKILL.md` (node conditions).
- DNS latency findings correlate with `skills/slo-latency-error-analysis/SKILL.md` (intermittent timeout correlation) and `skills/logging-auditing/SKILL.md` (timeout patterns in logs).
- Service mesh findings correlate with `skills/slo-latency-error-analysis/SKILL.md` (mesh latency overhead in request latency).
- Conntrack exhaustion correlates with `skills/capacity-trend-analysis/SKILL.md` (connection count trends).
- Network policy findings correlate with `skills/k8s-performance-inspection/SKILL.md` (security posture section).
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
