---
name: slo-latency-error-analysis
description: "Analyze service latency (p50/p90/p95/p99/p999), error rate, traffic volume, and SLO error budget burn rate using multi-window multi-burn-rate methodology (Google SRE Workbook). Surface services trending toward or breaching SLOs before customer-facing incidents."
---

# Skill: Service Latency, Error-Rate & SLO Burn-Rate Analysis

## Goal

For services in the requested application namespace(s), analyze:
1. Request latency distribution (p50/p90/p95/p99/p999) and latency trends
2. Error rate by category (5xx, 4xx, timeout, circuit breaker) and error budget burn rate
3. Traffic volume and distribution hotspots
4. SLO compliance using multi-window, multi-burn-rate alerting methodology (per Google SRE Workbook Chapter 5)
5. Dependency chain correlation (upstream/downstream impact propagation)
6. Low-traffic service handling

Surface services trending toward or already breaching acceptable latency/error thresholds so they can be prioritized ahead of a customer-facing incident.

## Access Model

Uses the approved read-only Prometheus MCP server against the VictoriaMetrics backend described in `component-contexts/context.observability.md`. Never port-forward or connect directly to `vmselect`/`vminsert`/`vmalert`.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md` (application namespaces only).
- `time_window` — from the runtime input contract in `component-contexts/context.observability.md`. Default to `last 1h` for live diagnosis; confirm with the user before using a wider range.
- Optional `slo_targets` — e.g. `{availability: "99.9%", latency_p99: "300ms", latency_p95: "100ms"}`. If not provided, report raw figures without pass/fail judgment and note that no SLO target was supplied.
- Optional `slo_classification` — request class per SRE Workbook taxonomy: `CRITICAL` (99.99%), `HIGH_FAST` (99.9%, p99<200ms), `HIGH_SLOW` (99.9%, p99<5s), `LOW` (99%), `NO_SLO`. If not provided, attempt to infer from namespace role (see `context.kubernetes.md` topology).

## Metric Verification (Pre-Check)

Before running analysis, verify metric availability:
- `label_values(istio_requests_total, namespace)` — confirm mesh metrics are scraped for target namespaces
- `label_values(istio_request_duration_milliseconds_bucket, namespace)` — confirm latency histograms are available
- If `istio_requests_total` is not available for target namespaces (mesh metrics may only cover `istio-system` — see open items in `context.observability.md`), check for alternative metrics:
  - Application-level metrics (e.g. `http_requests_total`, `http_request_duration_seconds_bucket`) if scraped
  - Spring Boot Actuator metrics (`http_server_requests_seconds_*`) if exposed
- Record any metric unavailability as `data_gap` before proceeding.

## Inspection Check Categories

### 1. Request Volume & Traffic Distribution

Collect via Prometheus MCP:
- Request volume per workload: `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))`
- Request rate trend (for the full `time_window`): `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))` via `query_range`
- Top traffic services: `topk(10, sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m])))`
- Request method distribution: `sum by (namespace, workload, request_method) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))`

Analysis:
- Flag `traffic_hotspot` (S3) when a single workload receives >50% of total namespace traffic — capacity concentration risk.
- Flag `traffic_anomaly` (S2) when current traffic volume deviates >50% from the same time window 7 days prior — may indicate a broken upstream, a marketing surge, or a misconfigured retry loop.
- Flag `zero_traffic` (S3) when a deployed workload has zero requests in the window — may indicate a misconfigured service or broken routing.

### 2. Error Rate Analysis

Collect via Prometheus MCP:
- Overall error rate: `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>", response_code=~"5.."}[5m])) / sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))`
- Error rate by code: `sum by (namespace, workload, response_code) (rate(istio_requests_total{namespace=~"<ns>", response_code=~"5.."}[5m]))`
- 4xx rate (client errors, for context): `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>", response_code=~"4.."}[5m])) / sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))`
- Error rate trend over `time_window` via `query_range`

Analysis:
- Flag `error_rate_elevated` (S1) when error rate materially exceeds historical baseline (compare current window to same window 24h and 7d prior).
- Flag `error_rate_spike` (S2) when error rate has a sudden step-change increase correlated with a specific time point — cross-reference with deployment events from `skills/k8s-performance-inspection/SKILL.md`.
- Flag `sustained_5xx` (S1) when 5xx error rate is above 1% for more than 50% of the `time_window`.
- Flag `4xx_anomaly` (S3) when 4xx rate is abnormally high — may indicate client misconfiguration, API contract changes, or authentication failures.
- Categorize errors: `server_error` (5xx), `client_error` (4xx), `timeout` (504), `circuit_breaker_open` (503), `rate_limited` (429).

### 3. Latency Distribution & Percentile Analysis

Collect via Prometheus MCP:
- p50: `histogram_quantile(0.50, sum by (namespace, workload, le) (rate(istio_request_duration_milliseconds_bucket{namespace=~"<ns>"}[5m])))`
- p90: `histogram_quantile(0.90, sum by (namespace, workload, le) (rate(istio_request_duration_milliseconds_bucket{namespace=~"<ns>"}[5m])))`
- p95: `histogram_quantile(0.95, sum by (namespace, workload, le) (rate(istio_request_duration_milliseconds_bucket{namespace=~"<ns>"}[5m])))`
- p99: `histogram_quantile(0.99, sum by (namespace, workload, le) (rate(istio_request_duration_milliseconds_bucket{namespace=~"<ns>"}[5m])))`
- p99.9: `histogram_quantile(0.999, sum by (namespace, workload, le) (rate(istio_request_duration_milliseconds_bucket{namespace=~"<ns>"}[5m])))`
- Latency trend for each percentile over `time_window` via `query_range`

Analysis:
- Flag `latency_regression` (S2) when p99 latency has a step-change increase (>50% from previous baseline) — correlate with deployment, restart, or resource saturation signals.
- Flag `latency_tail_severity` (S2) when p99 / p50 ratio exceeds 10x — wide tail distribution indicates intermittent issues (GC pauses, lock contention, downstream timeouts).
- Flag `slo_latency_breach` (S1) when p99 latency exceeds the `slo_targets.latency_p99` threshold for more than 50% of the `time_window` (if SLO targets provided).
- Flag `latency_degradation_trend` (S2) when latency is gradually increasing over the `time_window` without a step-change — may indicate gradual resource exhaustion or data growth.
- Report percentile spread per workload: `{p50: xms, p90: yms, p95: zms, p99: wms, p999: vms}`.

### 4. SLO Error Budget Burn Rate Analysis

Based on the **multi-window, multi-burn-rate** methodology from the Google SRE Workbook (Chapter 5: Alerting on SLOs).

**Prerequisite:** If `slo_targets.availability` is not provided, skip burn rate calculation and report raw error rate only.

Compute error budget burn rate:
- `burn_rate = error_rate / (1 - SLO)` where `SLO` is the availability target as a decimal (e.g. 99.9% = 0.999)
- A burn rate of 1 means the service is consuming its error budget at exactly the rate that exhausts it by the end of the SLO period (e.g. 30 days).

Collect via Prometheus MCP (assuming `SLO = 99.9%`, error budget = 0.1% = 0.001):

**Page-level burn rate (fast detection):**
- 1h window: `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>", response_code=~"5.."}[1h])) / sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[1h])) > 14.4 * 0.001`
- 5m window (short window for reset time): `... rate(...[5m]) > 14.4 * 0.001`
- Combined: `(1h_error_rate > 14.4 * error_budget) AND (5m_error_rate > 14.4 * error_budget)` — 2% budget consumed in 1h

**Page-level burn rate (slower, sustained detection):**
- 6h window: `... rate(...[6h]) > 6 * 0.001`
- 30m window: `... rate(...[30m]) > 6 * 0.001`
- Combined: `(6h_error_rate > 6 * error_budget) AND (30m_error_rate > 6 * error_budget)` — 5% budget consumed in 6h

**Ticket-level burn rate (slow burn):**
- 3d window: `... rate(...[3d]) > 1 * 0.001`
- 6h window: `... rate(...[6h]) > 1 * 0.001`
- Combined: `(3d_error_rate > 1 * error_budget) AND (6h_error_rate > 1 * error_budget)` — 10% budget consumed in 3d

| Burn Rate | Long Window | Short Window | Budget Consumed | Severity |
|---|---|---|---|---|
| 14.4x | 1h | 5m | 2% | Page (S0/S1) |
| 6x | 6h | 30m | 5% | Page (S1/S2) |
| 1x | 3d | 6h | 10% | Ticket (S2/S3) |

Analysis:
- Flag `burn_rate_critical` (S0) when the 14.4x multi-window condition is met — 2% of 30-day error budget consumed in 1 hour, immediate action needed.
- Flag `burn_rate_high` (S1) when the 6x multi-window condition is met — 5% of 30-day error budget consumed in 6 hours.
- Flag `burn_rate_sustained` (S2) when the 1x multi-window condition is met — 10% of 30-day error budget consumed in 3 days, slow but sustained burn.
- Flag `error_budget_depleted` (S0) when cumulative error budget consumption over the SLO period (30 days) exceeds 100% — SLO is already breached.
- Report remaining error budget percentage per workload.

### 5. Availability Calculation

Compute availability from metrics:
- `availability = 1 - (5xx_count / total_request_count)` over the `time_window`
- `availability = 1 - (error_count / total_count)` if non-5xx errors are also counted (depends on SLO definition)

Analysis:
- Flag `availability_below_slo` (S0) when computed availability is below the `slo_targets.availability` threshold.
- Flag `availability_trending_down` (S1) when availability is decreasing over the `time_window` even if still above SLO — early warning.
- Report availability per workload: `{availability: 99.97%, slo_target: 99.9%, margin: 0.07%}`.

### 6. Dependency Chain Correlation

Collect via Prometheus MCP and Kubernetes MCP:
- Identify upstream callers: `sum by (source_workload, destination_workload) (rate(istio_requests_total{destination_namespace=~"<ns>"}[5m]))` — if source/destination labels are available
- Identify downstream dependencies: `sum by (source_workload, destination_workload) (rate(istio_requests_total{source_namespace=~"<ns>"}[5m]))`
- Cross-reference with Kubernetes MCP: check if flagged workloads are called by gateway services (see `context.kubernetes.md` topology — `pinjam` and `ml-cashloan` host front-door APIs)

Analysis:
- Flag `upstream_impact` (S2) when a downstream dependency's error rate correlates (within the same time window) with an upstream service's latency increase — the downstream is causing the upstream's latency.
- Flag `cascade_risk` (S1) when a critical-path service (e.g. `api-gateway-service` in `pinjam`) has elevated error rate — downstream services depending on it will be impacted.
- Flag `dependency_timeout` (S2) when a service's p99 latency increase correlates with a downstream service's error rate spike — the downstream is timing out.

### 7. Low-Traffic Service Handling

For services with very low request volume (< 100 req/min), standard burn-rate thresholds produce noisy results. Apply special handling:

Analysis:
- Flag `low_traffic_unreliable_metrics` (S3) when a service has < 10 requests in a 5-minute window — error rate and percentile calculations are statistically unreliable.
- For low-traffic services, use longer aggregation windows (1h instead of 5m) to get meaningful signal.
- Consider aggregating with parent service if the low-traffic service is part of a larger product flow (per SRE Workbook guidance on combining services).
- Flag `single_request_failure` (S4) as informational when a single failed request in a low-traffic service produces a high error rate percentage — note that this may not represent a systemic issue.

### 8. Currently Firing Alerts Correlation

Collect via Prometheus MCP:
- `ALERTS{alertstate="firing", namespace=~"<ns>"}` — currently firing alerts in scope
- `ALERTS{alertstate="pending", namespace=~"<ns>"}` — pending alerts

Analysis:
- Flag `firing_alert_unacknowledged` (S2) when a firing alert exists for a workload but no corresponding issue was identified in checks 1-7 — may indicate an alerting rule for a condition not covered by this skill.
- Correlate firing alerts with findings from checks 1-7 to validate alert accuracy.
- Flag `alert_storm` (S2) when >5 alerts are firing simultaneously in a single namespace — may indicate a cascading failure or an overly sensitive alerting configuration.

## Output Format

- `slo_analysis_summary` — per namespace: total services analyzed, services with SLO targets, services breaching SLO, average error budget remaining
- `service_findings[]`:
  - `namespace`
  - `workload`
  - `metric` (`latency_p50`, `latency_p90`, `latency_p95`, `latency_p99`, `latency_p999`, `error_rate`, `availability`, `burn_rate`, `traffic_volume`)
  - `observed_value`
  - `slo_target` (if supplied)
  - `baseline_comparison` (vs 24h/7d prior, if data available)
  - `issue_type` (`slo_breach`, `error_rate_elevated`, `latency_regression`, `burn_rate_critical`, `burn_rate_high`, `burn_rate_sustained`, `error_budget_depleted`, `availability_below_slo`, `availability_trending_down`, `traffic_hotspot`, `traffic_anomaly`, `cascade_risk`, `dependency_timeout`, `low_traffic_unreliable_metrics`, `data_gap`)
  - `evidence` (per `skills/reporting.md` evidence format: source, query, window, sample)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `confidence` (`high` if multi-window data available, `medium` if single window, `low` if low-traffic or data gaps)
  - `recommendation`
  - `owner_hint`
- `error_budget_summary{}` — per workload: `{slo_target, current_availability, error_budget_remaining_pct, burn_rate_1h, burn_rate_6h, burn_rate_3d}`
- `latency_distribution{}` — per workload: `{p50, p90, p95, p99, p999, p99_p50_ratio}`
- `dependency_correlation[]` — upstream/downstream impact pairs with correlation evidence
- `firing_alerts[]` — currently firing/pending alerts in scope
- `data_gaps[]` — metrics that were unavailable and what was checked instead

## Guardrails

- No automatic alert/rule creation or modification.
- Do not include raw request/response payloads in evidence samples — aggregate metric values only.
- Respect the 30-day/unscoped-query confirmation gate in `AGENTS.md`.
- For burn-rate queries that span 3 days (`3d` window), ensure `time_window` covers at least 3 days or adjust the long window accordingly.
- If `istio_requests_total`/`istio_request_duration_milliseconds_bucket` are not scraped for the target namespace(s), record a `data_gap` finding and note which alternative signal was checked.

## Cross-Skill References

- Latency regression findings correlate with `skills/cluster-network-performance/SKILL.md` (DNS/mesh latency) and `skills/resource-utilization-inspection/SKILL.md` (CPU throttling/OOM).
- Error rate spikes correlate with `skills/k8s-performance-inspection/SKILL.md` (crash loops, rollout failures) and `skills/logging-auditing/SKILL.md` (error patterns in logs).
- Burn-rate trends over multi-week windows should be cross-referenced with `skills/capacity-trend-analysis/SKILL.md` (capacity-driven SLO degradation).
- Dependency chain correlations should be validated against the topology in `component-contexts/context.kubernetes.md` section 5.
# Skill: Service Latency, Error-Rate, and SLO Burn Analysis

## Goal

For services in the requested application namespace(s), analyze request latency (p50/p95/p99), error rate, and traffic volume over a confirmed time window, and evaluate SLO burn rate where a target is known. Surface services trending toward or already breaching acceptable latency/error thresholds so they can be prioritized ahead of a customer-facing incident.

## Access Model

Uses the approved read-only Prometheus MCP server against the VictoriaMetrics backend described in `component-contexts/context.observability.md`. Never port-forward or connect directly to `vmselect`/`vminsert`/`vmalert`.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md` (application namespaces only).
- `time_window` — from the runtime input contract in `component-contexts/context.observability.md`. Default to `last 1h` for live diagnosis; confirm with the user before using a wider range.
- Optional `slo_targets` — e.g. `p99 < 300ms`, `error_rate < 1%`. If not provided, report raw latency/error-rate figures without a pass/fail judgment and note that no SLO target was supplied.

## Read-Only Evidence Collection

Representative PromQL (adjust label names to what is actually scraped; verify via `label_values` before relying on a specific label):

- Request volume: `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))`
- Error rate: `sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>", response_code=~"5.."}[5m])) / sum by (namespace, workload) (rate(istio_requests_total{namespace=~"<ns>"}[5m]))`
- Latency percentiles: `histogram_quantile(0.99, sum by (namespace, workload, le) (rate(istio_request_duration_milliseconds_bucket{namespace=~"<ns>"}[5m])))`
- Currently firing alerts in scope: `ALERTS{alertstate="firing", namespace=~"<ns>"}`
- Restart/crash correlation (Kubernetes MCP, not Prometheus): recent restarts for workloads flagged above, to distinguish app-level latency from pod churn.

If `istio_requests_total`/`istio_request_duration_milliseconds_bucket` are not scraped for the target namespace(s) (mesh metrics may only cover `istio-system` — see open items in `context.observability.md`), record a `data_gap` finding rather than assuming healthy state, and note which alternative signal (e.g. application-level metrics, if any are scraped) was checked instead.

## Analysis Logic

- Flag `slo_breach` when a supplied `slo_targets` threshold is violated for a sustained portion (e.g. >50%) of the time window, not a single sample spike.
- Flag `error_rate_elevated` when error rate materially exceeds namespace/service historical baseline (compare current window to the same window 24h/7d prior if data is available).
- Flag `latency_regression` when p99 latency has a step-change increase correlated with a deploy, restart, or resource saturation signal (cross-reference `skills/cluster-network-performance/SKILL.md` and `skills/resource-utilization-inspection/SKILL.md` findings if run in the same session).
- Flag `data_gap` when a needed metric/target is absent for a namespace/workload in scope.

## Output Format

- `slo_analysis_summary`
- `service_findings[]` with fields:
  - `namespace`
  - `workload`
  - `metric` (`latency_p99`, `error_rate`, `traffic_volume`)
  - `observed_value`
  - `slo_target` (if supplied)
  - `issue_type` (`slo_breach`, `error_rate_elevated`, `latency_regression`, `data_gap`)
  - `evidence` (per `skills/reporting.md` evidence format: source, query, window, sample)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `recommendation`

## Guardrails

- No automatic alert/rule creation or modification.
- Do not include raw request/response payloads in evidence samples — aggregate metric values only.
- Respect the 30-day/unscoped-query confirmation gate in `AGENTS.md`.
