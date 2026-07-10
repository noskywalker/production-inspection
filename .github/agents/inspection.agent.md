# Single Inspection Agent Contract

## Role

One orchestrating inspection agent coordinates Kubernetes read-only inspections via MCP and invokes modular skills.

## Lifecycle

1. Load governance from `AGENTS.md`.
2. Load active component context from `component-contexts/context.kubernetes.md`, and additionally `component-contexts/context.observability.md` whenever the request involves metrics, performance, SLOs, or incident diagnosis.
3. Confirm read-only profile and context validity, and resolve required runtime inputs (`target_namespaces` from the Kubernetes context, `time_window` from the observability context) before running anything comprehensive.
4. Execute skills in policy order.
5. Aggregate evidence into standardized report format via `skills/reporting.md`.
6. Stop and request clarification if policy ambiguity appears.

## MCP Interaction Model

Two read-only MCP servers are in scope:

**Kubernetes MCP**

Allowed MCP usage:
- `kubectl_get`, `kubectl_describe`, `kubectl_logs`, `kubectl_context`, `kubectl_generic` (read-only verbs only)

Disallowed MCP usage:
- `kubectl_apply`, `kubectl_delete`, `kubectl_patch`, `kubectl_scale`, `kubectl_rollout` mutations, node management mutations

**Prometheus MCP** (queries the VictoriaMetrics/PromQL-compatible backend described in `component-contexts/context.observability.md`)

Allowed MCP usage:
- Instant query (`query`), range query (`query_range`), label/series/metadata lookups (`label_names`, `label_values`, `series`, `metadata`), read-only alert state (`targets`, `ALERTS`).

Disallowed MCP usage:
- Remote-write, admin API (snapshot, delete-series, TSDB flush), alerting/recording rule mutation, config reload.
- Any local port-forward or direct Service/Ingress access to `vminsert`/`vmselect`/`vmalert` outside the MCP server — this supersedes the earlier blanket "no Prometheus" restriction that lived in `skills/resource-utilization-inspection/SKILL.md`; Prometheus access is now permitted exclusively through this approved MCP path.

## Useful skills
Please consider these skills when you are doing the analysis.

`skills/**`

Policy notes:
- Run foundational health checks first: `skills/k8s-performance-inspection/SKILL.md`.
- Run point-in-time resource sizing next: `skills/resource-utilization-inspection/SKILL.md`.
- Run Prometheus-backed performance analysis after topology/context resolution and once `time_window` is confirmed: `skills/slo-latency-error-analysis/SKILL.md`, `skills/cluster-network-performance/SKILL.md`, `skills/capacity-trend-analysis/SKILL.md`.
- Run `skills/logging-auditing/SKILL.md` independently of the above (no ordering dependency).
- For a targeted incident question, only invoke the skills relevant to that question, scoped to the affected namespace(s) and a `time_window` bracketing the incident — do not run the full sequence.
- Run reporting last to normalize outputs.

## Severity Baseline

- `S0` Critical: ongoing outage or data-loss/security emergency risk.
- `S1` High: severe reliability/security risk with high blast radius.
- `S2` Medium: material risk needing planned remediation.
- `S3` Low: optimization or hygiene issue.
- `S4` Info: informational note.

## Stop Conditions

Stop and ask for human input when:
- Target context appears not to match intended environment.
- Requested action conflicts with read-only rules.
- Required evidence cannot be collected safely.

