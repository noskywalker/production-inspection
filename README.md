# Production Environment Inspection

This repository initializes a documentation-first scaffold for a single Kubernetes inspection agent that runs through the Kubernetes MCP server in strict read-only mode.

## Scope

- Primary target: Kubernetes cluster inspection.
- Mode: read-only evidence collection only.
- Execution model: one orchestrating agent + modular skills.
- Governance: explicit `allowed_actions`, `forbidden_actions`, approval gates, and audit trail.

## Repository Structure

- `AGENTS.md`: global harness governance and operational boundaries.
- `agents/agent.md`: single-agent lifecycle and MCP interaction contract.
- `agents/inspection.agent.md`: compatibility alias for `agents/agent.md`.
- `component-contexts/`: per-component context files.
  - `context.kubernetes.md` (active)
  - `context.kafka.md`, `context.redis.md`, `context.mysql.md` (templates)
- `skills/`: reusable inspection skills and reporting contract.
- `prompts/bootstrap.md`: bootstrap requirements source.

## Standard Workflow

1. Load `AGENTS.md` and validate read-only profile.
2. Load `component-contexts/context.kubernetes.md`.
3. Run skills in the invocation order from `agents/agent.md`.
4. Emit findings through `skills/reporting.md`.

## Non-Negotiable Safety Rules

- Never mutate production-like environments during inspection.
- Never run create/update/delete/patch/scale/restart/drain operations.
- If uncertain, stop and ask for clarification.

