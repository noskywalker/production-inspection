# MySQL Component Context (Template)

> Status: template — not yet populated from a live MCP snapshot. Copy the structure used in `context.kubernetes.md` once a MySQL deployment is in scope.

## Snapshot Metadata

- Date (UTC): _pending_
- Deployment model: _managed (e.g. RDS/PolarDB/ApsaraDB), in-cluster operator, or self-managed StatefulSet_
- Topology: _single primary / primary-replica / group replication_

## Architecture

- Primary/replica layout, connection pooling/proxy layer (e.g. ProxySQL, Vitess) if present.
- Storage class / PVC sizing if in-cluster.

## Namespaces and Microservices

- Namespace(s) hosting MySQL (if in-cluster) or managed-service endpoint reference: _pending_
- Consuming application namespaces/services: _pending_ — note from `context.kubernetes.md` that `dme` (data movement/CDC via Canal/Flink) replicates from production MySQL to DWH, so DME lag is a relevant downstream signal.

## Invocation Topology

_Pending — describe which application services connect to which database/schema, and CDC consumers (e.g. `dme` namespace's `canal-admin`)._

## Relevant Configurations to Track in Inspection

- Replication lag (seconds behind primary).
- Slow query log volume and top offending queries.
- Connection count vs. `max_connections`.
- InnoDB buffer pool hit ratio, lock wait time.

## Read-Only Evidence Sources

- Kubernetes MCP: pod status/restarts if in-cluster; proxy layer pod health.
- Prometheus MCP (see `context.observability.md`): `mysqld_exporter` metrics if scraped (`mysql_global_status_*`, `mysql_slave_status_seconds_behind_master`).

## Open Items for Operator Confirmation

- Confirm MySQL is in-scope, whether it is in-cluster or an external managed service, and which namespace(s)/services depend on it.
- Confirm whether mysqld_exporter feeds the observability stack (external managed DBs may not be scrapeable at all).
