# Skill: Resource Request/Limit Right-Sizing

## Goal

Identify workloads with missing or mis-sized requests/limits and provide evidence-backed recommendations.

## Inputs

- Application namespace list
- Optional target utilization band (for example 50%-70% request utilization)

## Read-Only Evidence Collection

- Deployment/statefulset container requests and limits
- Missing request/limit coverage by namespace/service
- Node allocatable snapshots and scheduling pressure signals
- Actual usage indicators when metrics are available; fallback to events/restarts if not

## Analysis Logic

- Flag `missing_config` when requests/limits are absent.
- Flag `too_low` when throttling/OOM-like symptoms appear with low requests/limits.
- Flag `too_high` when requests materially exceed observed steady usage.
- Recommend per-service adjustment range and rollout caution notes.

## Output Format

- `coverage_summary`
- `rightsizing_findings[]`:
  - `namespace`
  - `workload`
  - `current_requests_limits`
  - `observed_usage`
  - `issue`
  - `recommended_range`
  - `confidence`

## Guardrails

- No automatic patching of resources.

