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
