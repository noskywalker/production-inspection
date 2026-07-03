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
