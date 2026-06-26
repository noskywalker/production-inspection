# Skill: Resource Request/Limit Right-Sizing

## Goal

Identify workloads of the pods in the application-related namespaces.Mainly to analyze the CPU utilization and memory utilization of the workloads, and to identify any workloads that have a resource request/limit configuration that is too low or too high, which may lead to resource contention or waste. The skill is able to collect multiple metrics, including the resource request/limit configuration of the workloads, the actual resource usage of the workloads, and the resource usage of the nodes. The skill is able to identify any workloads that have a resource request/limit configuration that is too low or too high, and provide recommendations for adjusting the configuration.
You should use metric-server embedded in k8s cluster, please do not try to connect a prometheus mcp server or start a port forward to collect metrics from prometheus server, as it may cause security issues and is not allowed in production environment.

## Inputs

- The application namespaces provided by the previous context.
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

