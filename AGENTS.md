# AGENTS Harness - Production Environment Inspection

## Purpose

This file defines governance for the production-environment inspection harness: scope, objectives, permissions, prohibited behavior, approvals, and audit requirements.
The current repository is a documentation-first scaffold: agent behavior is defined through checked-in Markdown contracts rather than runnable application code.

## Environment Overview

- Primary runtime target: Kubernetes clusters accessed through Kubernetes MCP.
- Current connected context (from MCP): `ali-mgr-staging-config`.
- Governance model: read-only inspection profile.
- Active component context file: `component-contexts/context.kubernetes.md`.
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
- `skills/reporting.md` is not present in this workspace; use the severity baseline from `.github/agents/inspection.agent.md` together with each skill file's `Output Format` section when normalizing findings.

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

- Read-only Kubernetes API operations through MCP (`get`, `list`, `describe`, `logs` for diagnosis, non-interactive read-only `exec` only when explicitly approved).
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
- port-forwarding for non-inspection purposes

## approval_gates

- Any action that might mutate resources: **always blocked** in this profile.
- Any privileged `exec` request: require explicit human approval and command-level review.
- Any broad log collection touching sensitive data: require owner approval and data handling confirmation.

## audit_requirements

Every inspection run must capture:

- Timestamp window and cluster context.
- Namespace/resource scope queried.
- Commands issued (or MCP API operations) and sampled outputs/evidence links.
- Findings with severity, confidence, impact, and recommended actions.
- Explicit statement that no mutation operation was executed.

## Mandatory Rules

1. Read-only profile is mandatory.
2. If uncertain, stop and ask before proceeding.
3. Prefer least-privilege data access.
4. Never include secrets in reports.

