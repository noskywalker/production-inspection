# Production Environment Inspection

This repository initializes a documentation-first scaffold for a single production-environment inspection agent that runs through the Kubernetes MCP server and a read-only Prometheus MCP server (querying the VictoriaMetrics backend), both in strict read-only mode.

## Scope

- Primary targets: Kubernetes cluster inspection and Prometheus-compatible (VictoriaMetrics) metrics analysis for performance/incident diagnosis.
- Mode: read-only evidence collection only.
- Execution model: one orchestrating agent + modular skills.
- Governance: explicit `allowed_actions`, `forbidden_actions`, approval gates, and audit trail.

## Repository Structure

- `AGENTS.md`: global harness governance and operational boundaries.
- `.github/agents/inspection.agent.md`: single-agent lifecycle, MCP interaction contract (Kubernetes MCP + Prometheus MCP), and `skill_invocation_policy`.
- `component-contexts/`: per-component context files.
  - `context.kubernetes.md` (active) — cluster/namespace scope and `target_namespaces` contract.
  - `context.observability.md` (active) — Prometheus/VictoriaMetrics access model and `time_window` contract.
  - `context.kafka.md`, `context.redis.md`, `context.mysql.md` (templates)
- `skills/`: reusable inspection skills and reporting contract.
  - `k8s-performance-inspection/SKILL.md`, `resource-utilization-inspection/SKILL.md`, `logging-auditing/SKILL.md` — Kubernetes MCP / metric-server based.
  - `slo-latency-error-analysis/SKILL.md`, `cluster-network-performance/SKILL.md`, `capacity-trend-analysis/SKILL.md` — Prometheus MCP based.
  - `reporting.md` — shared severity/evidence/remediation conventions across all skills.
- `prompts/bootstrap.md`: bootstrap requirements source.

## Standard Workflow

1. Load `AGENTS.md` and validate read-only profile.
2. Load `component-contexts/context.kubernetes.md`, and `component-contexts/context.observability.md` when the request involves metrics/performance/incidents.
3. Run skills in the invocation order from `.github/agents/inspection.agent.md`.
4. Emit findings through `skills/reporting.md`.

## Non-Negotiable Safety Rules

- Never mutate production-like environments during inspection.
- Never run create/update/delete/patch/scale/restart/drain operations.
- Prometheus/VictoriaMetrics access is query-only through the approved MCP server — no remote-write, no rule mutation, no port-forward/direct backend access.
- If uncertain, stop and ask for clarification.

