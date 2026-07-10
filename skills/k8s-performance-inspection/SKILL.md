---
name: k8s-performance-inspection
description: "Comprehensive Kubernetes cluster health, workload reliability, and security posture inspection through read-only Kubernetes MCP. Use as the foundational skill before running any deeper performance or SLO analysis."
---

# Skill: Kubernetes Cluster Health & Reliability Inspection

## Goal

Perform a comprehensive, read-only health and reliability audit of the Kubernetes cluster covering: node health, control-plane readiness, workload reliability, probe strategy, disruption resilience, topology/HA posture, image supply-chain hygiene, Kubernetes version compliance, configuration drift, and storage/PVC health. This is the **foundational skill** — run it first before deeper performance or SLO analysis.

## Access Model

Uses the Kubernetes MCP server (read-only). All operations must be `get`, `describe`, `logs`, or `kubectl_generic` with read-only verbs. Never `apply`, `patch`, `delete`, `scale`, `rollout`, `cordon`, `uncordon`, or `drain`.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md` (application namespaces only; system/infra namespaces are inspected separately for platform health).
- Optional `focus_areas` — array of specific check categories to narrow scope (e.g. `["workload_reliability", "probe_strategy"]`). If omitted, run all checks.
- Optional `severity_filter` — minimum severity to include in output (e.g. `S2` to exclude `S3`/`S4`).

## Inspection Check Categories

### 1. Node Health

Collect via MCP:
- `kubectl_get nodes -o wide` — node status, roles, versions, internal/external IPs
- `kubectl_describe node <name>` — conditions (MemoryPressure, DiskPressure, PIDPressure, Ready), capacity, allocatable, taints, system info
- `kubectl_get events --field-selector type=Warning --all-namespaces` filtered to node-related events

Analysis:
- Flag `node_not_ready` (S0) when any node condition is not `Ready` or has `Unknown` status.
- Flag `node_pressure` (S1) when MemoryPressure, DiskPressure, or PIDPressure is `True`.
- Flag `node_taint_issue` (S2) when a node carries an unexpected taint (e.g. `node.kubernetes.io/unreachable`, `node.kubernetes.io/not-ready`) that could be affecting scheduling.
- Flag `node_version_skew` (S3) when kubelet versions across nodes differ by more than one minor version.

### 2. Control-Plane Health

Collect via MCP:
- `kubectl_get componentstatuses` (if available) or equivalent — scheduler, controller-manager, etcd health
- `kubectl_get pods -n kube-system` — control-plane addon pods (coredns, metrics-server, etc.)
- `kubectl_get events -n kube-system --field-selector type=Warning` — recent control-plane warnings

Analysis:
- Flag `component_unhealthy` (S0) when any control-plane component is not `Healthy`.
- Flag `addon_degraded` (S1) when critical addons (coredns, metrics-server, kube-proxy) have CrashLoopBackOff or high restart counts.
- Flag `api_server_latency` (S2) if API server response latency appears elevated (correlate with slow `kubectl_get` responses observed during inspection).

### 3. Workload Reliability

Collect via MCP:
- `kubectl_get deployments,sts,ds -n <ns> -o wide` — replica availability
- `kubectl_get pods -n <ns> --field-selector=status.phase!=Running` — non-running pods
- `kubectl_get pods -n <ns> -o json` — parse for `containerStatuses` restart counts, last termination state, waiting reasons
- `kubectl_get events -n <ns> --field-selector type=Warning --sort-by=.lastTimestamp` — recent warnings
- `kubectl_get replicasets -n <ns> -o wide` — old ReplicaSets from failed rollouts

Analysis:
- Flag `crash_loop` (S0) for pods in `CrashLoopBackOff`.
- Flag `image_pull_failure` (S1) for pods in `ImagePullBackOff` / `ErrImagePull`.
- Flag `pending_pod` (S1) for pods stuck in `Pending` — inspect events for `FailedScheduling` (insufficient resources, taint mismatch, affinity rules).
- Flag `high_restart_count` (S2) when a container has restarted more than 5 times in the last 24h (parse `restartCount` and `lastState`).
- Flag `rollout_stuck` (S2) when a Deployment's `updatedReplicas < replicas` or `unavailableReplicas > 0` for more than 15 minutes.
- Flag `evicted_pod` (S3) for pods with `Evicted` phase — check for `node-pressure` eviction reason.
- Flag `stale_replicaset` (S3) when old ReplicaSets with 0 replicas linger beyond the `revisionHistoryLimit` — indicates failed rollout history.

### 4. Probe Strategy Audit

Collect via MCP:
- `kubectl_get deploy,sts -n <ns> -o json` — parse `spec.template.spec.containers[].readinessProbe`, `livenessProbe`, `startupProbe`

Analysis:
- Flag `missing_readiness_probe` (S2) for any container receiving traffic (identified by being in a Service selector) without a readiness probe — traffic may be sent to a not-ready container.
- Flag `missing_liveness_probe` (S3) for any container without a liveness probe — unresponsive containers won't be restarted automatically.
- Flag `missing_startup_probe` (S3) for slow-starting containers (identified by high restart count during startup) without a startup probe.
- Flag `probe_misconfigured` (S2) when:
  - `timeoutSeconds` is too low (e.g. <2s) for a service with known latency variability.
  - `failureThreshold` combined with `periodSeconds` creates a too-aggressive or too-lenient detection window.
  - Liveness probe endpoint is the same as readiness probe (common anti-pattern — liveness should check deeper health).

### 5. Pod Disruption Budget (PDB) Coverage

Collect via MCP:
- `kubectl_get pdb -n <ns> -o wide` — all PDBs
- Cross-reference with deployments/statefulsets that have replicas >= 2

Analysis:
- Flag `missing_pdb` (S2) for any Deployment/StatefulSet with replicas >= 2 that is likely critical (inferred from namespace role — see `context.kubernetes.md` topology) but has no PDB.
- Flag `pdb_too_restrictive` (S2) when `minAvailable` equals `replicas` — voluntary disruptions are fully blocked, which can block node drains and HPA scale-down.
- Flag `pdb_mismatch` (S3) when a PDB selector does not match any existing pods (orphaned PDB).

### 6. Topology Spread & High-Availability Posture

Collect via MCP:
- `kubectl_get deploy,sts -n <ns> -o json` — parse `topologySpreadConstraints`, `podAntiAffinity`, `podAffinity`
- `kubectl_get pods -n <ns> -o wide` — actual pod-to-node distribution
- `kubectl_get nodes -o wide` — node zones/labels

Analysis:
- Flag `no_anti_affinity` (S2) for any multi-replica Deployment/StatefulSet without `podAntiAffinity` or `topologySpreadConstraints` — all replicas could land on one node/zone.
- Flag `topology_concentration` (S2) when all replicas of a workload are on the same node or same zone despite multi-zone cluster (from `context.kubernetes.md` zones `ap-southeast-5a/5b/5c`).
- Flag `single_replica` (S2) for any critical service (identified by topology role — e.g. gateway, core domain service) running with only 1 replica — no fault tolerance.

### 7. Image Supply-Chain Hygiene

Collect via MCP:
- `kubectl_get deploy,sts,ds -n <ns> -o json` — parse container `image` fields

Analysis:
- Flag `latest_tag` (S2) for any container using `:latest` or no tag — non-reproducible, can introduce unexpected changes.
- Flag `unpinned_digest` (S3) for images using mutable tags (e.g. `v1.2.3`) without a digest (`@sha256:...`) — susceptible to registry tag reassignment.
- Flag `registry_mismatch` (S3) if images come from an unexpected/untrusted registry vs. the cluster's standard registry.

### 8. Kubernetes Version Compliance

Collect via MCP:
- `kubectl_get nodes -o json` — parse `status.nodeInfo.kubeletVersion`
- `kubectl version` or `kubectl_generic` equivalent — cluster server version
- `kubectl_api-resources` — detect deprecated API resources (alpha, deprecated)

Analysis:
- Flag `eol_version` (S1) if the cluster server version is past end-of-life (check against Kubernetes support calendar — versions older than N-2 are out of support).
- Flag `deprecated_api` (S2) if workloads are using deprecated API versions (e.g. `extensions/v1beta1`, `networking.k8s.io/v1beta1` in clusters >= 1.22).
- Flag `version_skew_excessive` (S2) if node kubelet versions differ from API server by more than 2 minor versions.

### 9. Configuration Drift Detection

Collect via MCP:
- `kubectl_get deploy,sts -n <ns> -o json` — compare live spec against expected (if a GitOps source is accessible, otherwise note the limitation)
- `kubectl_get configmap -n <ns>` — inventory ConfigMaps used by workloads
- Check for annotations like `argocd.argoproj.io/sync-status` or `fluxcd.io/sync-status`

Analysis:
- Flag `config_drift` (S2) when a live resource has a `OutOfSync` status annotation (ArgoCD/Flux) or when manual changes are detectable.
- Flag `manual_override` (S3) when a resource has no GitOps annotation at all in a cluster that uses GitOps — may indicate manual creation outside the declared state.

### 10. Storage & PVC Health

Collect via MCP:
- `kubectl_get pvc -n <ns> -o wide` — PVC status
- `kubectl_get pv -o wide` — PV status
- `kubectl_get sc` — storage classes
- `kubectl_get events -n <ns> --field-selector reason=FailedMount` — mount failures
- `kubectl_describe pvc <name> -n <ns>` — for PVCs not in `Bound` state

Analysis:
- Flag `pvc_pending` (S1) for any PVC stuck in `Pending` — check for missing StorageClass, capacity exhaustion, or provisioning errors.
- Flag `pvc_high_usage` (S2) for PVCs near capacity — cross-reference with node disk pressure findings.
- Flag `storage_class_missing` (S2) if a PVC references a non-existent StorageClass.
- Flag `failed_mount` (S1) for any `FailedMount` events — CSI driver issues, node unavailability, or permission errors.

### 11. Namespace & Resource Quota Health

Collect via MCP:
- `kubectl_get resourcequota -n <ns> -o json` — quota usage vs. limits
- `kubectl_get limitrange -n <ns> -o json` — default resource constraints
- `kubectl_get ns <ns> -o json` — namespace status and labels

Analysis:
- Flag `quota_near_limit` (S2) when any resource quota is above 80% utilization (CPU, memory, PVC count, pod count) — may block new deployments or scaling.
- Flag `quota_exceeded` (S1) when any resource quota hard limit is reached — pods will fail to schedule.
- Flag `missing_limitrange` (S3) for application namespaces without a LimitRange — containers can be deployed without resource defaults.
- Flag `namespace_terminating` (S1) for any namespace stuck in `Terminating` state.

## Output Format

- `cluster_health_summary` — overall cluster posture (node count, ready nodes, version, total workloads, healthy/unhealthy breakdown)
- `category_findings{}` keyed by check category (1-11 above), each containing:
  - `namespace` (if applicable)
  - `workload` (if applicable)
  - `issue_type` (from the analysis flags above)
  - `evidence` (per `skills/reporting.md` evidence format: source, query_or_command, window, sample)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `confidence` (`high`/`medium`/`low`)
  - `recommendation`
  - `owner_hint`
- `data_gaps[]` — checks that could not be completed and why

## Inspection Order

Run checks in this order for maximum efficiency (earlier checks provide context for later ones):

1. Node Health (sets the stage — unhealthy nodes affect everything)
2. Control-Plane Health (cluster-level reliability)
3. Workload Reliability (the most impactful for application health)
4. Storage & PVC Health (often a root cause of workload failures)
5. Probe Strategy Audit (root cause of traffic-related failures)
6. PDB Coverage (disruption resilience)
7. Topology Spread & HA Posture (availability risk)
8. Namespace & Resource Quota Health (capacity constraints)
9. Image Supply-Chain Hygiene (security/reproducibility)
10. Kubernetes Version Compliance (long-term sustainability)
11. Configuration Drift Detection (operational hygiene)

## Cross-Skill References

- Findings from Node Health feed into `skills/cluster-network-performance/SKILL.md` (node saturation correlation) and `skills/resource-utilization-inspection/SKILL.md` (node pressure correlation).
- Findings from Workload Reliability feed into `skills/slo-latency-error-analysis/SKILL.md` (restart/crash correlation with latency regressions).
- Findings from Storage & PVC Health feed into `skills/capacity-trend-analysis/SKILL.md` (storage growth trends).

## Guardrails

- No automatic patching, scaling, or rolling restart — this skill is analysis-only.
- Do not include secrets, tokens, or sensitive ConfigMap values in evidence samples.
- When `kubectl_describe` output is very large, extract only the relevant section (conditions, events, or container statuses) rather than including the full output.
- If a check cannot be completed due to MCP limitations or RBAC restrictions, record it as a `data_gap` rather than skipping silently.
---
name: kubernetes-troubleshoot
description: "Troubleshoot and manage Kubernetes clusters, including resource inspection, debugging, pod logs, events, and cluster operations. Use when the user needs to diagnose issues, inspect workloads, analyze pod failures, or perform Kubernetes cluster operations."
---

# Kubernetes Troubleshooting & Management Skill

## What this Skill does

Use this skill when the user needs to troubleshoot or manage Kubernetes clusters. This includes operations such as:

- Listing pods, deployments, namespaces, nodes
- Getting pod logs
- Fetching events for a resource
- Inspecting workloads and resource conditions
- Understanding CrashLoopBackOff, ImagePullBackOff, pending pods
- Multi-cluster interactions through kubeconfig contexts
- Suggesting next debugging steps
- Only focusing on application-related namespaces when requested

## Tool Preference: MCP First, kubectl as Fallback

**Preferred Method**: Use the Kubernetes MCP server when available

- MCP tools allow pre-approved operations for faster execution
- More efficient for common read operations
- Built-in safety guardrails

**Fallback Method**: Use kubectl commands via terminal when:

- MCP server is not available or fails
- MCP cannot provide the information needed
- Specific kubectl features are required (port-forward, plugins, etc.)

This skill helps to:

- Choose the appropriate tool based on availability and capabilities
- Restrict queries to namespace/cluster automatically  
- Ask for confirmation before destructive actions  
- Recommend stepwise debugging strategies  
- Provide safe, context-efficient responses

## Best Practices

### 1. Always scope operations

- Include **namespace** unless user explicitly wants cluster-wide.
- Include **context** when user has multiple clusters.
- Encourage **label selectors** instead of listing all resources.

### 2. Prefer read-only operations first

Recommended sequence for debugging:

1. List pods matching a selector  
2. Describe pod  
3. Fetch pod events  
4. Retrieve logs  
5. Inspect configmaps/secrets/environment  
6. Only then consider restart/delete/scale

### 4. Tool Usage Guidelines

**When using MCP** (preferred):
Examples of safe MCP-driven operations:

- **List Pods**: `list pods --namespace=<ns> --context=<cluster>`
- **Get Pod Logs**: `logs --namespace=<ns> --pod=<pod>`
- **Get Events**: `get events --namespace=<ns> --field-selector=involvedObject.name=<name>`
- **Inspect Deployment**: `get deployment <name> --namespace=<ns>`

**When using kubectl** (fallback but not recommended for multi-step operations):

- Always specify namespace with `-n <namespace>` or `--namespace=<namespace>`
- Use `--context` when multiple clusters are configured
- Consider using `-o yaml` or `-o json` for detailed inspection
- Use `kubectl explain` for resource documentation

- **List Pods**
  - `list pods --namespace=<ns> --context=<cluster>`

- **Get Pod Logs**
  - `logs --namespace=<ns> --pod=<pod>`

- **Get Events**
  - `get events --namespace=<ns> --field-selector=involvedObject.name=<name>`

- **Inspect Deployment**
  - `get deployment <name> --namespace=<ns>`

### 5. Multi-Cluster Awareness

If multiple contexts exist:

- Always request or infer the correct context.
- Avoid ambiguous commands that default to the wrong cluster.

## Example User Requests → Recommended Actions

| User wants                          | Preferred (MCP)                                                                         | Fallback (kubectl)                                                  |
|-------------------------------------|-----------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| "List pods in frontend namespace" | `list pods --namespace=frontend` | `kubectl get pods -n frontend` |
| "Show me events for api-server" | `get events --namespace=<ns> --field-selector=involvedObject.name=api-server` | `kubectl get events -n <ns> --field-selector involvedObject.name=api-server` |
| "Get logs for db-0" | `logs --namespace=<ns> --pod=db-0` | `kubectl logs -n <ns> db-0` |
| "Why is pod web-123 CrashLooping?" | List pod → describe → events → logs (stepwise) | `kubectl describe pod -n <ns> web-123`, then logs |
| "Restart worker deployment" | Ask confirmation → delete pods or rollout restart if supported | `kubectl rollout restart deployment/<name> -n <ns>` |

## Tool Limitations

**MCP Limitations**:

- Some advanced operations may not be available (port-forward, plugin commands)
- Complex manifest edits may require kubectl fallback

**kubectl Limitations**:

- Requires user approval for each command
- Less efficient for multiple sequential operations
- No pre-approval mechanisms) may not be available  
- Complex edits to manifests may require manual patching  
