---
name: resource-utilization-inspection
description: "Analyze resource request/limit configuration, QoS class distribution, HPA/PDB coverage, CPU throttling, OOM patterns, node pressure correlation, and ResourceQuota/LimitRange posture for application workloads using metric-server point-in-time snapshots."
---

# Skill: Resource Request/Limit Right-Sizing & Utilization Inspection

## Goal

Identify workloads in application-related namespaces with improper resource request/limit configurations that may lead to resource contention or waste. Analyze CPU/memory utilization, QoS class distribution, HPA behavior, PDB coverage, throttling/OOM signals, and node-level pressure correlation. Provide actionable right-sizing recommendations.

This skill intentionally uses the **metric-server** embedded in the k8s cluster for a fast, point-in-time usage snapshot, and does not itself query Prometheus. For historical/trend usage or SLO-based right-sizing, use `skills/capacity-trend-analysis/SKILL.md` or `skills/slo-latency-error-analysis/SKILL.md`, which query the approved read-only Prometheus MCP server described in `component-contexts/context.observability.md`.

## Access Model

Uses Kubernetes MCP (read-only): `kubectl_get`, `kubectl_describe`, `kubectl_top` (or `kubectl_generic` with `top` subcommand), and `kubectl_logs` for OOM evidence. Never patch, apply, or scale resources.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md`.
- Optional `target_utilization_band` — desired request utilization band (default: 50%-70%, meaning actual usage should be 50%-70% of the request).
- Optional `workload_filter` — regex to narrow to specific workloads.
- Optional `include_node_analysis` — whether to include node-level pressure correlation (default: `true`).

## Inspection Check Categories

### 1. Request/Limit Coverage Audit

Collect via MCP:
- `kubectl_get deploy,sts,ds -n <ns> -o json` — parse `spec.template.spec.containers[].resources.requests` and `.limits` for each container
- `kubectl_get pods -n <ns> -o json` — parse `spec.containers[].resources` for actual running pod resource specs (catches pods created outside controllers)

Analysis:
- Flag `missing_request` (S1) for any container without CPU or memory requests — scheduler cannot make informed placement decisions, risk of node overcommit.
- Flag `missing_limit` (S2) for any container without CPU or memory limits — container can consume unlimited resources, risk of noisy-neighbor.
- Flag `missing_both` (S1) for any container with neither requests nor limits — `BestEffort` QoS, first to be evicted under pressure.
- Flag `request_without_limit` (S3) for containers with requests but no limits — `Burstable` QoS with unbounded consumption potential.

### 2. QoS Class Distribution Analysis

Collect via MCP:
- `kubectl_get pods -n <ns> -o json` — compute QoS class for each pod:
  - `Guaranteed`: every container has equal CPU and memory requests and limits
  - `Burstable`: at least one container has a request or limit, but not all are Guaranteed
  - `BestEffort`: no container has requests or limits

Analysis:
- Flag `besteffort_in_production` (S1) for any `BestEffort` pod in an application namespace — first to be evicted under node pressure, no resource guarantees.
- Flag `besteffort_ratio_high` (S2) when >10% of pods in a namespace are `BestEffort` — systemic lack of resource governance.
- Flag `guaranteed_ratio_low` (S3) as informational when <20% of pods are `Guaranteed` — critical services should use Guaranteed QoS for predictable performance and to avoid CPU throttling.
- Report QoS distribution per namespace: `{Guaranteed: x%, Burstable: y%, BestEffort: z%}`.

### 3. CPU & Memory Utilization vs. Request/Limit

Collect via MCP:
- `kubectl_top pods -n <ns> --containers` (or `kubectl_generic` equivalent) — point-in-time CPU/memory usage per container
- `kubectl_top nodes` — node-level utilization for correlation
- Cross-reference with request/limit values from Check 1

Analysis (using `target_utilization_band` default 50%-70%):
- Flag `under_requested` (S2) when actual usage is consistently below 30% of the request — wasted reserved resources, over-provisioning.
- Flag `over_requested` (S1) when actual usage exceeds the request — risk of throttling (CPU) or OOMKill (memory) during bursts.
- Flag `at_limit` (S2) when actual usage is consistently above 90% of the limit — container is at its ceiling, any burst will cause throttling or OOM.
- Flag `cpu_throttling_likely` (S2) when a `Burstable` or `Guaranteed` container's usage is at or near its CPU limit — Linux CFS quota throttling is likely occurring. Cross-reference with `skills/k8s-performance-inspection/SKILL.md` restart findings and `skills/slo-latency-error-analysis/SKILL.md` latency findings.
- Flag `memory_pressure_likely` (S1) when a container's usage is above 85% of its memory limit — OOMKill is imminent under any memory spike.

### 4. CPU Throttling & OOMKill Detection

Collect via MCP:
- `kubectl_get pods -n <ns> -o json` — parse `status.containerStatuses[].lastState.terminated.reason` for `OOMKilled`
- `kubectl_describe pod <name> -n <ns>` — check for recent restart events and termination messages
- `kubectl_logs <pod> -n <ns> --previous --tail=50` — check previous container logs for OOM or throttling indicators
- `kubectl_get events -n <ns> --field-selector reason=OOMKilling` — OOM events

Analysis:
- Flag `oomkilled` (S1) for any container with `lastState.terminated.reason=OOMKilled` — memory limit is too low or there's a memory leak.
- Flag `repeated_oomkill` (S0) when a container has been OOMKilled more than 3 times in 24h — active production-impacting issue.
- Flag `cpu_throttle_symptom` (S2) when a container has high restart count with `lastState.terminated.reason=OOMKilled` but memory usage is moderate — may be CPU throttling causing GC pressure (common in JVM workloads).
- Flag `oom_without_limit` (S2) when a container was OOMKilled but has no memory limit set — the node's allocatable memory was exhausted (system-level OOM).

### 5. HPA Configuration Review

Collect via MCP:
- `kubectl_get hpa -n <ns> -o wide` — all HPAs in scope
- `kubectl_get hpa <name> -n <ns> -o json` — parse `spec.metrics`, `spec.minReplicas`, `spec.maxReplicas`, `spec.scaleTargetRef`

Analysis:
- Flag `missing_hpa` (S2) for any Deployment with variable traffic (inferred from namespace role — e.g. gateway, API services) that has no HPA — no automatic scaling for demand spikes.
- Flag `hpa_min_too_low` (S2) when `minReplicas` is 1 for a critical service — no redundancy during low-traffic periods.
- Flag `hpa_max_too_low` (S2) when `maxReplicas` is set but the service's actual usage frequently hits the max (all replicas at limit) — HPA ceiling is too restrictive.
- Flag `hpa_max_too_high` (S3) when `maxReplicas` far exceeds what the namespace ResourceQuota could support — HPA will scale up and then pods will fail to schedule due to quota.
- Flag `hpa_cpu_metric_unreliable` (S3) when HPA uses CPU utilization metric but containers lack CPU requests — CPU utilization percentage is undefined without a request baseline.
- Flag `hpa_stale` (S3) when an HPA's `CURRENT` replicas differ from `DESIRED` for an extended period — HPA may be unable to scale (resource constraints, scheduling failures).

### 6. PDB Coverage for Resource-Aware Disruption

Collect via MCP:
- `kubectl_get pdb -n <ns> -o wide`
- Cross-reference with workloads that have HPA (HPA + no PDB = risk of too many pods being disrupted during scale-down or node drain)

Analysis:
- Flag `hpa_without_pdb` (S2) for any HPA-managed Deployment without a PDB — voluntary disruptions (node drain, cluster autoscaler) can take down too many replicas simultaneously.
- Flag `pdb_blocks_hpa_scaledown` (S3) when PDB `minAvailable` is set too high relative to `minReplicas` — HPA cannot scale down because PDB blocks pod eviction.

### 7. Node Resource Pressure Correlation

Collect via MCP:
- `kubectl_top nodes` — node CPU/memory utilization
- `kubectl_get nodes -o json` — parse `status.allocatable`, `status.capacity`, `spec.taints`
- `kubectl_get pods --all-namespaces -o wide` — pod-to-node distribution for density analysis
- `kubectl_describe node <name>` — conditions, non-terminated pods count

Analysis (only if `include_node_analysis` is `true`):
- Flag `node_overcommitted` (S1) when total pod requests on a node exceed 90% of node allocatable CPU or memory — scheduling pressure, pods will compete for resources.
- Flag `node_cpu_saturation` (S2) when node CPU usage is consistently above 85% — risk of CPU throttling across all pods on that node.
- Flag `node_memory_saturation` (S2) when node memory usage is consistently above 85% — risk of system OOM and eviction.
- Flag `eviction_risk` (S2) when a node with `BestEffort` or `Burstable` pods is under memory pressure — these pods will be evicted first per kubelet eviction policy.
- Flag `scheduling_imbalance` (S3) when pod density varies significantly across nodes (e.g. one node has 2x the pod count of others) — may indicate scheduling policy issues or taint misconfiguration.

### 8. ResourceQuota & LimitRange Governance

Collect via MCP:
- `kubectl_get resourcequota -n <ns> -o json` — parse `status.hard` and `status.used`
- `kubectl_get limitrange -n <ns> -o json` — parse default/min/max resource constraints

Analysis:
- Flag `quota_near_limit` (S2) when any resource quota usage is above 80% — new pods may fail to schedule.
- Flag `quota_blocking_scale` (S1) when HPA `maxReplicas` * per-pod requests exceeds namespace quota — HPA scale-up will fail silently.
- Flag `missing_limitrange` (S3) for application namespaces without a LimitRange — no safety net for workloads deployed without resource specs.
- Flag `limitrange_defaults_low` (S3) when LimitRange default CPU is below 100m or default memory is below 128Mi — too low for most production Java workloads.

### 9. Right-Sizing Recommendation Engine

For each workload with sufficient data (request/limit + actual usage):

Compute:
- `cpu_request_utilization` = actual_cpu / cpu_request * 100%
- `cpu_limit_utilization` = actual_cpu / cpu_limit * 100%
- `memory_request_utilization` = actual_memory / memory_request * 100%
- `memory_limit_utilization` = actual_memory / memory_limit * 100%

Recommendation logic:
- If `cpu_request_utilization` < 30%: recommend reducing CPU request to `actual_cpu * 1.5` (50% headroom).
- If `cpu_request_utilization` > 90%: recommend increasing CPU request to `actual_cpu * 1.3` (leave 30% headroom for bursts).
- If `memory_request_utilization` < 50%: recommend reducing memory request to `actual_memory * 1.3` (30% headroom).
- If `memory_request_utilization` > 85%: recommend increasing memory request to `actual_memory * 1.5` (50% headroom for GC spikes, especially JVM).
- If `cpu_limit_utilization` > 95%: recommend increasing CPU limit or removing it (Guaranteed QoS).
- If `memory_limit_utilization` > 85%: recommend increasing memory limit — OOMKill risk.
- If actual usage data is unavailable (metric-server not returning data): note as `data_gap` and recommend using `skills/capacity-trend-analysis/SKILL.md` for historical analysis.

## Output Format

- `coverage_summary` — per namespace: total containers, with requests %, with limits %, QoS distribution
- `utilization_summary` — per namespace: average CPU/memory request utilization, limit utilization
- `rightsizing_findings[]`:
  - `namespace`
  - `workload`
  - `container`
  - `qos_class`
  - `current_requests_limits` (CPU and memory)
  - `observed_usage` (CPU and memory, point-in-time)
  - `utilization_pct` (request and limit)
  - `issue` (from analysis flags above)
  - `recommended_range` (CPU and memory request/limit)
  - `confidence` (`high` if multiple samples available, `medium` if single point-in-time, `low` if usage data unavailable)
  - `severity`
- `hpa_findings[]`:
  - `namespace`
  - `workload`
  - `issue` (from HPA analysis flags)
  - `current_config` (min/max replicas, metrics)
  - `recommendation`
  - `severity`
- `node_pressure_findings[]` (if `include_node_analysis`):
  - `node`
  - `issue` (from node analysis flags)
  - `allocatable` (CPU, memory)
  - `requested` (total pod requests on node)
  - `actual_usage` (from `kubectl top`)
  - `severity`
- `data_gaps[]` — checks that could not be completed and why

## Guardrails

- No automatic patching of resources — this skill is analysis-only.
- Point-in-time snapshots from metric-server may not represent peak usage — recommend cross-referencing with `skills/capacity-trend-analysis/SKILL.md` for trend data before making right-sizing decisions.
- Do not include secrets or sensitive ConfigMap values in evidence.

## Cross-Skill References

- CPU throttling and OOMKill findings feed into `skills/slo-latency-error-analysis/SKILL.md` (latency regression correlation) and `skills/k8s-performance-inspection/SKILL.md` (restart count correlation).
- Node pressure findings feed into `skills/cluster-network-performance/SKILL.md` (node saturation correlation).
- Right-sizing recommendations should be validated against `skills/capacity-trend-analysis/SKILL.md` trends before implementation.
- `oom_without_limit` findings correlate with `skills/logging-auditing/SKILL.md` (`oom_in_logs` detection).
# Skill: Resource Request/Limit Right-Sizing

## Goal

Identify workloads of the pods in the application-related namespaces.Mainly to analyze the CPU utilization and memory utilization of the workloads, and to identify any workloads that have a resource request/limit configuration that is too low or too high, which may lead to resource contention or waste. The skill is able to collect multiple metrics, including the resource request/limit configuration of the workloads, the actual resource usage of the workloads, and the resource usage of the nodes. The skill is able to identify any workloads that have a resource request/limit configuration that is too low or too high, and provide recommendations for adjusting the configuration.

This skill intentionally uses the metric-server embedded in the k8s cluster for a fast, point-in-time usage snapshot, and does not itself query Prometheus. Do not start a local port-forward or direct connection to the Prometheus/VictoriaMetrics backend from this skill. For historical/trend usage or SLO-based right-sizing, use `skills/capacity-trend-analysis/SKILL.md` or `skills/slo-latency-error-analysis/SKILL.md`, which query the approved read-only Prometheus MCP server described in `component-contexts/context.observability.md`.

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

