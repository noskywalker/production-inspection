---
name: logging-auditing
description: "Audit container and application logging practices including framework configuration (SLF4J/Log4j/Logback), log rotation, compression, centralization, retention, log-level distribution, error pattern detection, and structured logging posture."
---

# Skill: Logging Posture & Rotation Audit

## Goal

Assess logging practices across application namespaces with focus on:
1. Java application log framework configuration (SLF4J/Log4j/Logback) — rotation, compression, hourly splitting
2. Log centralization — whether a log agent DaemonSet ships logs to a centralized store
3. Log retention — whether logs are retained for a sufficient period
4. Log-level distribution — ERROR/WARN ratio and error pattern detection
5. Structured logging — JSON vs. plain text and contextual fields

## Access Model

Uses Kubernetes MCP (read-only) for resource inspection and log sampling. Non-interactive read-only `exec` into containers to inspect log config files requires explicit approval per `AGENTS.md` approval gates.

## Inputs

- `target_namespaces` — from the runtime input contract in `component-contexts/context.kubernetes.md`.
- Optional `service_filter` — regex to narrow to specific workloads.
- Optional `retention_target_days` — minimum expected retention (default: 30 days for application logs, 90 days for audit logs).
- Optional `log_sample_lines` — number of log lines to sample per pod for level/pattern analysis (default: 100, max: 500 to minimize sensitive data exposure).

## Inspection Check Categories

### 1. Log Framework Detection & Configuration

Collect via MCP:
- `kubectl_get configmap -n <ns>` — identify ConfigMaps that may contain logback.xml, log4j2.xml, or log4j.properties
- `kubectl_get deploy,sts -n <ns> -o json` — parse container `command`/`args` and `env` for logging-related JVM flags (e.g. `-Dlogging.config`, `LOG_LEVEL`, `LOGGING_LEVEL_*`)
- `kubectl_get pods -n <ns>` — identify pods to sample logs from
- If approved: read-only `exec` to inspect `/opt/app/conf/logback.xml`, `/opt/app/conf/log4j2.xml`, or classpath logging config

Analysis:
- Flag `framework_undetected` (S3) when a Java workload's logging framework cannot be identified — may indicate default config without rotation.
- Flag `config_externalized` (S4) as informational when logging config is properly externalized via ConfigMap.
- Detect framework type: Logback (`logback.xml`), Log4j2 (`log4j2.xml`), Log4j1 (`log4j.properties`), or Spring Boot defaults (`application.yml` `logging.*` properties).

### 2. Log Rotation & Rollover Analysis

Collect via MCP:
- Inspect ConfigMaps containing log config files for `RollingFileAppender`, `RollingRandomAccessFileAppender` (Log4j2)
- Parse rolling policies:
  - `TimeBasedRollingPolicy` — check `fileNamePattern` for hourly split (e.g. `%d{yyyy-MM-dd_HH}.log.gz`) vs. daily (e.g. `%d{yyyy-MM-dd}.log.gz`)
  - `SizeAndTimeBasedRollingPolicy` — check `maxFileSize` and rolling pattern
  - `FixedWindowRollingPolicy` with `SizeBasedTriggeringPolicy`
- Check `maxHistory` (Logback) or `DefaultRolloverStrategy max` (Log4j2) for retention count
- If approved: read-only `exec` to check actual log files on disk: `ls -la /opt/app/logs/` and `ls -la /var/log/containers/`

Analysis:
- Flag `rotation_missing` (S1) when no rolling/rotation policy is configured — log files will grow unbounded and eventually fill the disk.
- Flag `hourly_split_missing` (S2) when the application generates high-volume logs but uses daily (not hourly) rollover — large daily files are harder to search and rotate.
- Flag `max_history_missing` (S2) when `maxHistory` or `DefaultRolloverStrategy max` is absent or set to 0 — old log files are never cleaned up.
- Flag `max_history_too_low` (S3) when `maxHistory` < 7 days — insufficient for post-incident forensic analysis.

### 3. Log Compression Verification

Collect via MCP:
- Inspect log config `fileNamePattern` for compression suffix: `.gz` (gzip) or `.zip`
- If approved: read-only `exec` to check for compressed archived log files on disk

Analysis:
- Flag `compression_missing` (S2) when rolled-over log files are not compressed — wasted storage, especially for high-volume services.
- Flag `compression_format` (S4) as informational — note whether gzip (recommended) or zip is used.

### 4. Log Centralization Verification

Collect via MCP:
- `kubectl_get ds --all-namespaces` — look for log agent DaemonSets (common names: `fluent-bit`, `fluentd`, `td-agent`, `filebeat`, `promtail`, `o11y-td-agent-barito`)
- `kubectl_get ds <agent-name> -n <agent-ns> -o json` — parse `tolerations` to verify agent runs on all nodes (including tainted nodes)
- `kubectl_get cm -n <agent-ns>` — inspect agent ConfigMap for source paths (e.g. `/var/log/containers/*.log`) and destination (Elasticsearch, Loki, S3, Kafka)
- `kubectl_get pods -n <agent-ns> -o wide` — verify agent pods are running on all nodes
- Cross-reference with `component-contexts/context.kubernetes.md` section 3.1 for known logging infrastructure (e.g. `barito-worker`, `o11y-td-agent-barito`)

Analysis:
- Flag `centralization_missing` (S1) when no log agent DaemonSet is detected or the agent does not cover all nodes — logs are only available on the node and lost on pod eviction.
- Flag `agent_not_on_all_nodes` (S2) when the log agent DaemonSet has fewer ready pods than cluster node count — some nodes' logs are not being shipped.
- Flag `agent_source_mismatch` (S2) when the agent's configured source path does not match where application logs are written — logs are not being collected.
- Flag `agent_destination_unknown` (S3) when the agent's destination cannot be verified — logs may not be reaching the centralized store.

### 5. Log Retention Analysis

Collect via MCP:
- Inspect log agent ConfigMap for retention-related settings (index lifecycle management, retention days, max size)
- If the destination is Elasticsearch (check `barito-worker` namespace): inspect index lifecycle policy if accessible
- `kubectl_get pvc -n <log-ns>` — check storage allocation for log storage
- Cross-reference with platform log stack in `component-contexts/context.kubernetes.md` (e.g. `sadu-elasticsearch-es-default` StatefulSet in `barito-worker`)

Analysis:
- Flag `retention_insufficient` (S2) when observed retention is less than the `retention_target_days` input — forensic capability is limited.
- Flag `retention_unknown` (S3) when retention policy cannot be determined from accessible configuration — flag for operator confirmation.
- Flag `storage_near_limit` (S2) when log storage PVC usage is above 80% — logs may be dropped or rotated prematurely.

### 6. Log-Level Distribution & Error Pattern Detection

Collect via MCP:
- `kubectl_logs <pod> -n <ns> --tail=<log_sample_lines>` — sample recent logs from each workload (use a representative pod if multiple replicas)
- Parse sampled logs for level indicators: `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`
- For Java/SLF4J stacks: also detect stack trace blocks (lines starting with `\t` or `Caused by:`)

Analysis:
- Flag `high_error_ratio` (S1) when ERROR-level logs exceed 5% of total sampled lines — indicates active application issues.
- Flag `elevated_warn_ratio` (S2) when WARN-level logs exceed 15% of total — early warning of degrading conditions.
- Flag `repeated_error_pattern` (S2) when the same exception/error message appears more than 10 times in the sample — persistent issue, not transient.
- Flag `stack_trace_storm` (S2) when a single error produces a large stack trace repeated frequently — log noise that obscures real signals.
- Flag `oom_in_logs` (S1) when `OutOfMemoryError`, `java.lang.OutOfMemoryError`, or `OOMKilled` appears in sampled logs — correlate with `skills/resource-utilization-inspection/SKILL.md` findings.
- Flag `timeout_in_logs` (S2) when timeout-related keywords (`TimeoutException`, `Connection refused`, `Read timed out`, `CircuitBreaker`) appear — correlate with `skills/slo-latency-error-analysis/SKILL.md` and `skills/cluster-network-performance/SKILL.md`.
- Flag `debug_in_production` (S3) when DEBUG or TRACE level is active in a production namespace — excessive log volume and potential sensitive data exposure.

### 7. Structured Logging Posture

Collect via MCP:
- Sample logs and check for JSON format vs. plain text
- If JSON: verify presence of contextual fields (`timestamp`, `level`, `logger`, `thread`, `traceId`/`spanId`, `namespace`, `pod`)
- If plain text: check for consistent timestamp format and parseable structure

Analysis:
- Flag `unstructured_logs` (S3) when logs are plain text without consistent structure — harder to search, alert on, and correlate in centralized storage.
- Flag `missing_trace_context` (S3) when structured logs lack `traceId`/`spanId` — distributed tracing correlation is not possible from logs alone.
- Flag `missing_timestamp` (S2) when log entries lack a timestamp or use inconsistent timezone — complicates incident timeline reconstruction.
- Flag `sensitive_data_in_logs` (S1) when sampled logs appear to contain passwords, tokens, PII, or credit card numbers — immediate security risk.

### 8. Container stdout/stderr Logging

Collect via MCP:
- `kubectl_get deploy,sts -n <ns> -o json` — check if application logs are written to stdout/stderr or to files only
- Check for `volumeMounts` with log paths (e.g. `/opt/app/logs`, `/var/log/app`)

Analysis:
- Flag `file_only_logging` (S2) when application logs are written only to files (not stdout) and no log agent is configured to collect them — logs are inaccessible via `kubectl logs` and may be lost on pod restart.
- Flag `dual_logging` (S4) as informational when logs go to both stdout and files — redundant but ensures availability via `kubectl logs`.

## Output Format

- `logging_compliance_summary` — overall posture per namespace (framework detected, rotation configured, compression enabled, centralized, retention days, structured)
- `service_issues[]` with fields:
  - `namespace`
  - `service`
  - `check_category` (1-8 above)
  - `issue_type` (from the analysis flags above)
  - `evidence` (per `skills/reporting.md` evidence format — redact sensitive content from log samples)
  - `severity` (per severity baseline in `.github/agents/inspection.agent.md`)
  - `confidence` (`high`/`medium`/`low`)
  - `recommendation`
- `log_level_distribution{}` — per namespace: `{ERROR: x%, WARN: y%, INFO: z%, DEBUG: w%}`
- `error_patterns[]` — top repeated error signatures per namespace
- `data_gaps[]` — checks that could not be completed (e.g. exec not approved, ConfigMap inaccessible)

## Guardrails

- Do not expose sensitive payloads, credentials, or PII in report output — redact before including log samples.
- Use minimal log sampling needed for evidence (`log_sample_lines` default 100, max 500).
- Broad log collection touching sensitive data requires owner approval per `AGENTS.md` approval gates.
- Read-only `exec` to inspect log config files requires explicit human approval and command-level review.
- If a workload's log framework or config cannot be determined without `exec`, record as `data_gap` and note what `exec` command would be needed.

## Cross-Skill References

- `oom_in_logs` findings correlate with `skills/resource-utilization-inspection/SKILL.md` (memory limit too low) and `skills/k8s-performance-inspection/SKILL.md` (high restart count).
- `timeout_in_logs` findings correlate with `skills/slo-latency-error-analysis/SKILL.md` (latency regression) and `skills/cluster-network-performance/SKILL.md` (DNS/mesh issues).
- `debug_in_production` findings correlate with `skills/resource-utilization-inspection/SKILL.md` (excessive CPU from debug logging overhead).
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

