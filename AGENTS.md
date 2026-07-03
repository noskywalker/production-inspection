# AGENTS Harness - Production Environment Inspection

## Purpose

This file defines governance for the production-environment inspection harness: scope, objectives, permissions, prohibited behavior, approvals, and audit requirements.
The current repository is a documentation-first scaffold: agent behavior is defined through checked-in Markdown contracts rather than runnable application code.

## Environment Overview

- Primary runtime target: Kubernetes clusters accessed through Kubernetes MCP.
- Secondary runtime target: the VictoriaMetrics (Prometheus-compatible PromQL) backend running in the `observability` namespace, accessed exclusively through an approved read-only Prometheus MCP server.
- Current connected context (from MCP): `ali-mgr-staging-config`.
- Governance model: read-only inspection profile.
- Active component context files: `component-contexts/context.kubernetes.md` (cluster/namespace scope) and `component-contexts/context.observability.md` (metrics backend and Prometheus MCP access model), plus per-component templates under `component-contexts/`.
- Runtime input contract is active: collect `target_namespaces` before application performance inspection and follow validation rules defined in `component-contexts/context.kubernetes.md`; collect `time_window` before any Prometheus-backed skill per `component-contexts/context.observability.md`.
- Repository state: the current workspace contains harness documentation only; no application source tree, dependency manifest, or verified build/test entrypoint is present at the project root.
- Files under `prompts/**` are excluded from the inspection, please ignore them for the purpose of this inspection.

## Agent References

When agent-specific guidance exists under `.github/agents/*`, prefer those files as the primary per-agent operating instructions and keep `AGENTS.md` as the top-level harness governance file.

- Current single-agent contract: `.github/agents/inspection.agent.md`.
- Prefer checked-in paths over older bootstrap examples: the repo currently uses `.github/agents/inspection.agent.md` and skill-specific files under `skills/*/SKILL.md`.
- Current skill entrypoints:
  - `skills/k8s-performance-inspection/SKILL.md`
  - `skills/logging-auditing/SKILL.md`
  - `skills/resource-utilization-inspection/SKILL.md`
  - `skills/slo-latency-error-analysis/SKILL.md` (Prometheus MCP)
  - `skills/cluster-network-performance/SKILL.md` (Prometheus MCP)
  - `skills/capacity-trend-analysis/SKILL.md` (Prometheus MCP)
- Use `skills/reporting.md` together with the severity baseline from `.github/agents/inspection.agent.md` and each skill file's `Output Format` section when normalizing findings.

## Scope Boundaries

In scope:
- Cluster health, workload reliability, security posture, network and storage checks.
- Application namespace performance, logging posture, and resource right-sizing analysis.

Out of scope:
- Any operational change to cluster state.
- Remediation execution in production-like environments.

## inspection_objectives

1. Detect risks before incidents: availability, performance, security, and operability.
2. Provide evidence-backed findings with reproducible commands.
3. Prioritize remediations by severity, blast radius, and urgency.
4. Keep all inspection actions read-only and auditable.

## allowed_actions

- Read-only Kubernetes API operations through MCP (`kubectl_get`, `kubectl_describe`, `kubectl_logs`, `kubectl_context`, `kubectl_generic` with read-only verbs only).
- Read-only Prometheus MCP operations against the VictoriaMetrics backend: instant/range PromQL queries, label/series/metadata lookups, read-only alert state (see `component-contexts/context.observability.md`).
- Non-interactive read-only `exec` only when explicitly approved.
- Gather metadata and metrics references.
- Produce findings, risk scores, and recommendations.
- Correlate resources across namespaces and components.

## forbidden_actions

The agent must not perform any write/mutation operation, including but not limited to:

- `apply`, `create`, `replace`, `edit`, `patch`, `delete`
- rollout mutations (`restart`, `undo`, `pause`, `resume`)
- `scale` changes
- node lifecycle mutations (`cordon`, `uncordon`, `drain`)
- workload or config changes to Deployments, StatefulSets, DaemonSets, Services, ConfigMaps, Secrets, RBAC, policies
- port-forwarding for non-inspection purposes, including port-forwarding or direct Service/Ingress access to the Prometheus/VictoriaMetrics backend (`vminsert`, `vmselect`, `vmalert`) — all metrics access must go through the approved Prometheus MCP server
- Prometheus/VictoriaMetrics remote-write, admin API calls (snapshot, delete-series, TSDB flush), or alerting/recording rule mutation

## approval_gates

- Any action that might mutate resources: **always blocked** in this profile.
- Any privileged `exec` request: require explicit human approval and command-level review.
- Any broad log collection touching sensitive data: require owner approval and data handling confirmation.
- Any comprehensive application inspection without user-specified namespace(s): require explicit user confirmation before proceeding.
- Any Prometheus MCP query without an explicit or confirmed `time_window`, spanning more than 30 days, or not scoped to specific namespaces/workloads: require explicit user confirmation before proceeding.

## audit_requirements

Every inspection run must capture:

- Timestamp window and cluster context.
- Namespace/resource scope queried.
- Commands issued (or MCP API operations) and sampled outputs/evidence links, including PromQL queries and their `time_window` for any Prometheus MCP calls.
- Namespace input/validation trail (provided namespaces, validation result, and explicit confirmation for comprehensive scope when applicable).
- Findings with severity, confidence, impact, and recommended actions.
- Explicit statement that no mutation operation was executed.

## Mandatory Rules

1. Read-only profile is mandatory.
2. If uncertain, stop and ask before proceeding.
3. Prefer least-privilege data access.
4. Never include secrets in reports.
