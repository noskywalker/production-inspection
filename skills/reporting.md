# Reporting Contract

## Purpose

Every skill under `skills/*/SKILL.md` produces its own `Output Format` block with domain-specific fields. This file defines the shared conventions those blocks must follow so findings from different skills can be merged into one inspection report.

## severity_levels

Use the severity baseline defined in `.github/agents/inspection.agent.md` (`S0` Critical, `S1` High, `S2` Medium, `S3` Low, `S4` Info) for every skill's `severity` field. Do not introduce a parallel naming scheme in a skill's `Output Format` section.

## Evidence Format

Every finding must carry evidence sufficient for another operator to reproduce the observation without re-running the full inspection:

- `source` — where the evidence came from (`k8s-mcp`, `prometheus-mcp`, `kubectl-logs`, etc.).
- `query_or_command` — the exact read-only MCP call, PromQL query, or kubectl command issued.
- `window` — time window for metric-based evidence (start/end, UTC).
- `sample` — a minimal representative excerpt (value, log line, event message). Redact secrets and PII before including.

## Remediation Prioritization

Order recommendations by `severity` first, then by blast radius (number of dependent services/namespaces affected), then by estimated effort (favor low-effort/high-impact fixes first within the same severity+blast-radius tier). Each recommendation should state:

- `action` — what to change (never auto-applied; this repo is read-only).
- `owner_hint` — which namespace/team likely owns the fix.
- `risk_of_inaction` — one sentence on what happens if untouched.
- `confidence` — `high`/`medium`/`low`, reflecting how directly the evidence supports the conclusion.

## Aggregation Rules

- Deduplicate findings that reference the same `namespace`+`workload`+`issue_type` across skills; merge evidence instead of listing twice.
- Always include an explicit statement that no mutation operation was executed, per `AGENTS.md` audit requirements.
- If any skill hit a `data_gap` (metric/target unavailable, log path unreadable, etc.), surface it in a dedicated section rather than silently omitting the check.
