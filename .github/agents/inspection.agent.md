# Single Inspection Agent Contract

## Role

One orchestrating inspection agent coordinates Kubernetes read-only inspections via MCP and invokes modular skills.

## Lifecycle

1. Load governance from `AGENTS.md`.
2. Load active component context from `component-contexts/context.kubernetes.md`.
3. Confirm read-only profile and context validity.
4. Execute skills in policy order.
5. Aggregate evidence into standardized report format.
6. Stop and request clarification if policy ambiguity appears.

## MCP Interaction Model

Allowed MCP usage:
- `kubectl_get`, `kubectl_describe`, `kubectl_logs`, `kubectl_context`, `kubectl_generic` (read-only verbs only)

Disallowed MCP usage:
- `kubectl_apply`, `kubectl_delete`, `kubectl_patch`, `kubectl_scale`, `kubectl_rollout` mutations, node management mutations

## Useful skills
Please consider these skills when you are doing the analysis.

`skills/**`

Policy notes:
- Run foundational health checks first.
- Run performance and sizing analysis after topology/context resolution.
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

