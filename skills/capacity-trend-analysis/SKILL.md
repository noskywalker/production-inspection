---
name: capacity-trend-analysis
description: "Analyze multi-day/multi-week Prometheus trends in CPU, memory, traffic, storage, and replica count per namespace/workload. Flag services heading toward resource exhaustion, evaluate HPA effectiveness, detect seasonal patterns, and forecast capacity breach dates using growth-rate extrapolation."
---

# Skill: Capacity & Scaling Trend Analysis

## Goal

Analyze multi-day/multi-week Prometheus trends to flag services heading toward resource exhaustion or needing HPA/scale-up attention before they cause an incident. This is a longer-horizon complement to `skills/resource-utilization-inspection/SKILL.md` (which is point-in-time via metric-server).

Specifically:
1. CPU/memory usage trends and growth-rate forecasting
2. Traffic volume trends and demand projection
3. HPA effectiveness evaluation (is autoscaling compensating?)
4. Resource headroom calculation (available capacity vs. projected demand)
5. Node-level capacity planning (cluster autoscaler triggers, node pool sizing)
6. Storage capacity trends (PVC growth, disk usage)
7. Seasonal pattern detection (daily/weekly/monthly cycles)
8. Cost projection based on resource trends

## Access Model

Uses the approved read-only Prometheus MCP server against the VictoriaMetrics backend described in `component-contexts/context.observability.md`. Never port-forward or connect directly to `vmselect`/`vminsert`/`vmalert`.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md` (application namespaces only).
- `time_window` — from the runtime input contract in `component-contexts/context.observability.md`. This skill defaults to a longer window than the others (`last 7d`, or `last 30d` if the user wants a growth-trend view); confirm with the user per the 30-day confirmation gate in `AGENTS.md`.
- Optional `forecast_horizon_days` — how far ahead to project trends (default: 14 days, max: 90 days).
- Optional `growth_model` — `linear` (default) or `compound` for exponential growth patterns.

## Metric Verification (Pre-Check)

Before running analysis, verify metric availability and retention coverage:
- `label_values(container_cpu_usage_seconds_total, namespace)` — confirm cAdvisor CPU metrics for target namespaces
- `label_values(container_memory_working_set_bytes, namespace)` — confirm cAdvisor memory metrics
- `label_values(kube_pod_container_resource_requests, namespace)` — confirm kube-state-metrics resource data
- Check VictoriaMetrics retention (see open items in `context.observability.md`) — if retention < `time_window`, record `data_gap` noting the actual available range.

## Inspection Check Categories

### 1. Workload CPU Usage Trend & Forecasting

Collect via Prometheus MCP (`query_range` with coarse step, e.g. `1h`):
- CPU usage trend: `sum by (namespace, workload) (rate(container_cpu_usage_seconds_total{namespace=~"<ns>"}[1h]))`
- CPU request trend (kube-state-metrics): `sum by (namespace, workload) (kube_pod_container_resource_requests{resource="cpu", namespace=~"<ns>"})`
- CPU limit trend: `sum by (namespace, workload) (kube_pod_container_resource_limits{resource="cpu", namespace=~"<ns>"})`

Forecasting methodology:
- **Linear model**: Fit a linear regression `y = mx + b` to the time series. Project `y` at `now + forecast_horizon_days`. Compute growth rate as `m` (units/sec per day).
- **Compound model**: Compute week-over-week growth rate `r = (latest_7d_avg / previous_7d_avg) - 1`. Project: `projected = latest_avg * (1 + r)^(forecast_horizon_days / 7)`.
- Use linear as default; switch to compound if the R² of linear fit is below 0.7 and the compound model fits better.

Analysis:
- Flag `capacity_risk` (S1) when projected CPU usage will exceed the configured limit within `forecast_horizon_days` — service will hit CPU throttling.
- Flag `request_saturation` (S2) when projected CPU usage will exceed the configured request within `forecast_horizon_days` — scheduler may not have enough headroom, and Burstable pods may throttle.
- Flag `growth_acceleration` (S2) when the growth rate is itself increasing (second derivative is positive) — non-linear growth, may exhaust capacity faster than linear projection suggests.
- Flag `cpu_usage_declining` (S4) as informational when CPU usage is trending downward — may indicate traffic loss or an optimization.
- Do not flag `capacity_risk` from a single short-lived spike; require a sustained multi-sample trend (at least 3 data points over the window showing consistent growth).

### 2. Workload Memory Usage Trend & Forecasting

Collect via Prometheus MCP (`query_range` with coarse step, e.g. `1h`):
- Memory usage trend: `max by (namespace, workload) (container_memory_working_set_bytes{namespace=~"<ns>"})`
- Memory request trend: `sum by (namespace, workload) (kube_pod_container_resource_requests{resource="memory", namespace=~"<ns>"})`
- Memory limit trend: `sum by (namespace, workload) (kube_pod_container_resource_limits{resource="memory", namespace=~"<ns>"})`

Analysis:
- Flag `memory_capacity_risk` (S1) when projected memory usage will exceed the configured limit within `forecast_horizon_days` — OOMKill is imminent.
- Flag `memory_leak_suspected` (S1) when memory usage shows a monotonically increasing trend without a corresponding traffic increase — classic memory leak pattern, especially in JVM workloads.
- Flag `memory_leak_with_gc` (S2) when memory usage is increasing but with periodic drops (GC cycles) — may still be a slow leak if the post-GC baseline is increasing.
- Flag `memory_request_saturation` (S2) when projected memory usage will exceed the configured request — node scheduling pressure.
- Report growth rate per workload: `{current_usage, projected_usage, growth_rate_per_day, days_to_limit}`.

### 3. Traffic Volume Trend & Demand Projection

Collect via Prometheus MCP (`query_range` with coarse step, e.g. `1h`):
- Traffic trend (if mesh metrics available): `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[1h]))`
- Traffic distribution shift: `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[1h]))` — check if traffic share is shifting between services

Analysis:
- Flag `traffic_growth_outpacing_capacity` (S1) when traffic growth rate exceeds the resource growth rate (either from manual scaling or HPA) — demand is outpacing supply.
- Flag `traffic_declining` (S4) as informational when traffic is trending downward — may indicate user churn, feature deprecation, or upstream routing changes.
- Flag `traffic_concentration_increasing` (S2) when one workload's share of namespace traffic is increasing over time — capacity concentration risk.
- Flag `traffic_burst_pattern` (S3) when traffic shows periodic spikes (e.g. daily at 2am batch processing, end-of-month reporting) — ensure HPA can react to burst patterns.

### 4. HPA Effectiveness Evaluation

Collect via Prometheus MCP and Kubernetes MCP:
- Replica count trend: `kube_deployment_status_replicas{namespace=~"<ns>"}` via `query_range`
- HPA target utilization vs. actual: `kube_hpa_status_current_metrics_average_utilization` (if scraped) vs. `kube_hpa_spec_target_metric_average_utilization`
- HPA scaling events: rate of change in `kube_deployment_status_replicas` over time
- HPA config (Kubernetes MCP): `kubectl_get hpa -n <ns> -o json`

Analysis:
- Flag `hpa_not_scaling` (S2) when a workload has an HPA configured but replica count is flat despite traffic/usage fluctuations — HPA may be misconfigured (wrong metric, wrong target, or min=max).
- Flag `hpa_thrashing` (S2) when replica count fluctuates frequently (>5 scale events per hour) without a corresponding stable traffic pattern — HPA thresholds are mistuned, causing oscillation.
- Flag `hpa_max_reached` (S2) when replica count is at `maxReplicas` for a sustained period — HPA has hit its ceiling, service needs a higher max or resource optimization.
- Flag `hpa_min_reached` (S3) when replica count is at `minReplicas` for a sustained period during low traffic — HPA is working as expected, but verify `minReplicas` provides adequate redundancy.
- Flag `hpa_scale_up_slow` (S2) when HPA takes too long to scale up relative to traffic spikes — check HPA stabilization window and CPU metric aggregation interval.
- Flag `hpa_metric_misconfigured` (S3) when HPA uses CPU utilization but the workload is I/O-bound or memory-bound — CPU-based HPA will not react to the right signal.

### 5. Resource Headroom Calculation

For each workload with sufficient trend data:

Compute:
- `cpu_headroom_pct` = `(cpu_limit - current_cpu_usage) / cpu_limit * 100`
- `memory_headroom_pct` = `(memory_limit - current_memory_usage) / memory_limit * 100`
- `cpu_headroom_days` = days until projected CPU usage reaches limit (based on growth rate)
- `memory_headroom_days` = days until projected memory usage reaches limit

Analysis:
- Flag `low_headroom` (S2) when any headroom metric is below 20% — limited buffer for traffic spikes.
- Flag `critical_headroom` (S1) when any headroom metric is below 10% or `headroom_days` < 7 days — immediate attention needed.
- Flag `excessive_headroom` (S4) as informational when headroom is above 80% consistently — resources are over-provisioned, cost optimization opportunity.
- Report headroom per workload: `{cpu_headroom_pct, memory_headroom_pct, cpu_headroom_days, memory_headroom_days}`.

### 6. Node-Level Capacity Planning

Collect via Prometheus MCP and Kubernetes MCP:
- Node allocatable total: `sum(kube_node_status_allocatable_cpu_cores)`, `sum(kube_node_status_allocatable_memory_bytes)`
- Node requested total: `sum(kube_pod_container_resource_requests{resource="cpu"})`, `sum(kube_pod_container_resource_requests{resource="memory"})`
- Node usage trend: `sum(rate(node_cpu_seconds_total{mode!="idle"}[1h]))` via `query_range`
- Pending pods (Kubernetes MCP): `kubectl_get pods --all-namespaces --field-selector=status.phase=Pending`
- Cluster autoscaler status (if available): `kubectl_get deploy cluster-autoscaler -n kube-system` or check for CA-related events

Analysis:
- Flag `cluster_capacity_risk` (S1) when total pod requests are above 85% of total node allocatable — cluster is running hot, new pods may fail to schedule.
- Flag `cluster_capacity_exhaustion` (S0) when total pod requests exceed total node allocatable — pods are already failing to schedule (check for `FailedScheduling` events).
- Flag `pending_pods_capacity` (S1) when pods are pending due to `Insufficient cpu` or `Insufficient memory` — cluster needs more nodes or workloads need right-sizing.
- Flag `node_pool_imbalance` (S3) when one node pool is heavily utilized while another has spare capacity — may indicate workload placement issues or taint misconfiguration.
- Report cluster capacity summary: `{total_cpu_allocatable, total_cpu_requested, request_pct, total_memory_allocatable, total_memory_requested, request_pct}`.

### 7. Storage Capacity Trend Analysis

Collect via Prometheus MCP and Kubernetes MCP:
- PVC usage (if scraped): `kubelet_volume_stats_used_bytes`, `kubelet_volume_stats_capacity_bytes`
- PVC usage ratio: `kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes`
- Node disk usage trend: `1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})` via `query_range`
- PVC inventory (Kubernetes MCP): `kubectl_get pvc -n <ns> -o wide`

Analysis:
- Flag `pvc_capacity_risk` (S2) when a PVC's usage trend projects it will reach capacity within `forecast_horizon_days` — application will fail to write.
- Flag `pvc_near_full` (S1) when any PVC is above 85% capacity — immediate risk of write failures.
- Flag `node_disk_growth` (S2) when node root filesystem usage is trending upward and projected to exceed 80% within `forecast_horizon_days` — may impact kubelet, container logs, and ephemeral storage.
- Flag `log_storage_growth` (S3) when the log storage PVC (e.g. in `barito-worker` namespace) is growing faster than expected — may need to adjust retention policy (correlate with `skills/logging-auditing/SKILL.md`).
- Report storage trends per PVC: `{pvc_name, namespace, capacity, used, usage_pct, growth_rate_per_day, days_to_full}`.

### 8. Seasonal Pattern Detection

Analyze the time series data for recurring patterns:

Methodology:
- **Daily cycle**: Compare hourly averages across multiple days. Compute the coefficient of variation for each hour. If certain hours consistently show higher usage, a daily pattern exists.
- **Weekly cycle**: Compare day-of-week averages. If weekend traffic is significantly different from weekday, a weekly pattern exists.
- **Monthly cycle**: Check for end-of-month or beginning-of-month spikes (common in billing/reporting workloads).

Analysis:
- Flag `daily_peak_pattern` (S4) as informational — note peak hours and the magnitude of the peak vs. off-peak. Helps schedule maintenance windows and right-size for peak vs. average.
- Flag `weekly_pattern` (S4) as informational — note weekday vs. weekend traffic ratio.
- Flag `seasonal_capacity_risk` (S2) when seasonal peaks are approaching and current capacity may not handle them — e.g. if daily peak usage is at 85% of limit, the next peak may breach.
- Report pattern summary: `{daily_peak_hour, daily_peak_amplification, weekly_weekend_ratio, monthly_spike_day}`.

### 9. Cost Projection (Indicative)

Based on resource trends, provide an indicative cost projection:

Compute:
- Current resource allocation cost: `sum(cpu_request * cpu_unit_cost + memory_request * memory_unit_cost)` per namespace
- Projected resource allocation cost: same formula with projected resource needs
- Cost delta: `projected_cost - current_cost`
- Over-provisioning waste: `sum((request - actual_usage) * unit_cost)` for workloads with >50% headroom

Analysis:
- Flag `cost_growth_concern` (S4) as informational — projected cost increase over `forecast_horizon_days`.
- Flag `over_provisioning_waste` (S3) when total over-provisioning waste exceeds 30% of total allocation cost — significant right-sizing opportunity.
- Note: Cost calculations are indicative and based on standard cloud pricing. Actual costs depend on the cloud provider, reserved instances, and pricing model. Use `kubecost-system` if available (observed in `context.kubernetes.md` section 3.1) for more accurate cost data.

## Output Format

- `capacity_trend_summary` — overall capacity posture (cluster utilization, services at risk, HPA coverage, days to capacity exhaustion for worst-case workload)
- `findings[]`:
  - `namespace`
  - `workload`
  - `metric` (`cpu`, `memory`, `replica_count`, `traffic_volume`, `pvc_storage`, `node_disk`)
  - `current_value`
  - `trend_description` (e.g. "steady +8%/week over 4 weeks")
  - `growth_rate` (per day or per week)
  - `projected_value` (at `forecast_horizon_days`)
  - `projected_breach_date` (if applicable — date when usage crosses limit/request)
  - `days_to_breach` (if applicable)
  - `issue_type` (`capacity_risk`, `memory_leak_suspected`, `traffic_growth_outpacing_capacity`, `hpa_not_scaling`, `hpa_thrashing`, `hpa_max_reached`, `low_headroom`, `critical_headroom`, `cluster_capacity_risk`, `pvc_capacity_risk`, `seasonal_capacity_risk`, `over_provisioning_waste`, `data_gap`)
  - `evidence` (per `skills/reporting.md` evidence format)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `confidence` (`high` if >7 days of trend data, `medium` if 3-7 days, `low` if <3 days)
  - `recommendation`
  - `owner_hint`
- `hpa_effectiveness{}` — per workload with HPA: `{configured, current_replicas, min_replicas, max_replicas, scale_events_per_hour, target_metric, effectiveness_rating}`
- `headroom_summary{}` — per workload: `{cpu_headroom_pct, memory_headroom_pct, cpu_headroom_days, memory_headroom_days}`
- `cluster_capacity{}` — `{total_cpu_allocatable, total_cpu_requested, cpu_request_pct, total_memory_allocatable, total_memory_requested, memory_request_pct, pending_pods_count}`
- `storage_trends[]` — per PVC: `{pvc_name, namespace, capacity, used, usage_pct, growth_rate_per_day, days_to_full}`
- `seasonal_patterns{}` — `{daily_peak_hour, daily_peak_amplification, weekly_weekend_ratio, monthly_spike_day}`
- `cost_projection{}` — `{current_allocation_cost, projected_cost, cost_delta_pct, over_provisioning_waste_pct}`
- `data_gaps[]` — trend data that was incomplete or unavailable

## Guardrails

- No automatic HPA/resource patching, node scaling, or PVC resizing — this skill is analysis-only, recommendations require operator action.
- Use coarse query steps (`1h` for 7d window, `6h` for 30d window) and namespace/workload label scoping to avoid expensive unscoped long-range queries, per the guardrails in `component-contexts/context.observability.md`.
- Forecasting is based on historical extrapolation and assumes current growth patterns continue — note this assumption in the output. Unexpected events (viral traffic, incidents, new feature launches) will invalidate projections.
- Do not present cost projections as authoritative — they are indicative for planning purposes only.

## Cross-Skill References

- CPU/memory capacity risks should be validated against `skills/resource-utilization-inspection/SKILL.md` point-in-time data — trend analysis catches what a snapshot misses.
- HPA effectiveness findings correlate with `skills/slo-latency-error-analysis/SKILL.md` (latency during traffic spikes) and `skills/k8s-performance-inspection/SKILL.md` (pending pods from HPA scale-up blocked by capacity).
- Cluster capacity findings correlate with `skills/cluster-network-performance/SKILL.md` (node saturation) and `skills/k8s-performance-inspection/SKILL.md` (node pressure, scheduling failures).
- Storage trend findings correlate with `skills/logging-auditing/SKILL.md` (log storage retention) and `skills/k8s-performance-inspection/SKILL.md` (PVC health).
- Memory leak findings correlate with `skills/logging-auditing/SKILL.md` (`oom_in_logs` detection) and `skills/resource-utilization-inspection/SKILL.md` (OOMKill patterns).
# Skill: Capacity and Scaling Trend Analysis

## Goal

Analyze multi-day/multi-week Prometheus trends in CPU, memory, and traffic per namespace/workload to flag services heading toward resource exhaustion or that need HPA/scale-up attention before they cause an incident. This is a longer-horizon complement to `skills/resource-utilization-inspection/SKILL.md` (which is point-in-time via metric-server).

## Access Model

Uses the approved read-only Prometheus MCP server against the VictoriaMetrics backend described in `component-contexts/context.observability.md`. Never port-forward or connect directly to `vmselect`/`vminsert`/`vmalert`.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md` (application namespaces only).
- `time_window` — from the runtime input contract in `component-contexts/context.observability.md`. This skill defaults to a longer window than the others (`last 7d`, or `last 30d` if the user wants a growth-trend view); confirm with the user per the 30-day confirmation gate in `AGENTS.md`.

## Read-Only Evidence Collection

Representative PromQL (using `query_range` with a coarse step, e.g. `1h`, to keep result size manageable):

- Workload CPU usage trend: `sum by (namespace, workload) (rate(container_cpu_usage_seconds_total{namespace=~"<ns>"}[1h]))`
- Workload memory usage trend: `max by (namespace, workload) (container_memory_working_set_bytes{namespace=~"<ns>"})`
- Configured requests/limits for comparison (Kubernetes MCP, not Prometheus): `kube_pod_container_resource_requests`, `kube_pod_container_resource_limits` if exposed via kube-state-metrics, otherwise via direct deployment/statefulset spec read.
- Replica count trend (to see if HPA is already compensating): `kube_deployment_status_replicas{namespace=~"<ns>"}`
- Traffic volume trend (if mesh metrics available): `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[1h]))`

If historical data does not cover the full requested window (retention limits — see open items in `context.observability.md`), record a `data_gap` finding noting the actual available range.

## Analysis Logic

- Flag `capacity_risk` when a workload's usage trend is on a trajectory to exceed its configured limit (or node allocatable, for unlimited workloads) within a projected near-term horizon (e.g. linear extrapolation crossing the threshold within 2-4 weeks).
- Flag `scaling_thrash` when replica count fluctuates frequently without a corresponding stable traffic pattern, suggesting HPA thresholds are mistuned.
- Flag `under-provisioned_growth` when traffic volume is growing faster than allocated capacity for the same workload.
- Flag `data_gap` when trend data is incomplete for the requested window.
- Do not flag `capacity_risk` from a single short-lived spike; require a sustained multi-sample trend.

## Output Format

- `capacity_trend_summary`
- `findings[]` with fields:
  - `namespace`
  - `workload`
  - `metric` (`cpu`, `memory`, `replica_count`, `traffic_volume`)
  - `trend_description` (e.g. "steady +8%/week over 4 weeks")
  - `projected_breach_date` (if applicable)
  - `issue_type` (`capacity_risk`, `scaling_thrash`, `under-provisioned_growth`, `data_gap`)
  - `evidence` (per `skills/reporting.md` evidence format)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `recommendation`

## Guardrails

- No automatic HPA/resource patching — this skill is analysis-only, recommendations require operator action.
- Use coarse query steps and namespace/workload label scoping to avoid expensive unscoped long-range queries, per the guardrails in `component-contexts/context.observability.md`.
