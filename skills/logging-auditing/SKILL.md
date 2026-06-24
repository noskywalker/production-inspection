# Skill: Logging Posture and Rotation Audit

## Goal

Assess container/service logging practices, with focus on SLF4J/Log4j style application logs, rotation, compression, centralization, and retention.

## Inputs

- Namespace scope
- Optional service regex filters
- Optional retention policy target

## Read-Only Evidence Collection

- Pod logging behavior through `kubectl logs` sampling
- Log pattern checks for hourly split hints and rollover signatures
- Compression hints (`.gz`, archival markers) when available in mounted paths (read-only `exec` requires approval)
- Centralized log shipping indicators from daemonsets/agents (`td-agent`, `fluent-bit`, etc.)
- Retention evidence from platform logging stack configuration where readable

## Output Format

- `logging_compliance_summary`
- `service_issues[]` with fields:
  - `namespace`
  - `service`
  - `issue_type` (`rotation_missing`, `compression_missing`, `centralization_missing`, `retention_insufficient`)
  - `evidence`
  - `severity`
  - `recommendation`

## Guardrails

- Do not expose sensitive payloads in report output.
- Use minimal log sampling needed for evidence.

