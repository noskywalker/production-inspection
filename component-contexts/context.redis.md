# Redis Component Context (Template)

> Status: template — not yet populated from a live MCP snapshot. Copy the structure used in `context.kubernetes.md` once a Redis deployment is in scope.

## Snapshot Metadata

- Date (UTC): _pending_
- Topology: _standalone / sentinel / cluster mode_
- Operator (if any): _pending_

## Architecture

- Primary/replica topology, sentinel or cluster-mode shard layout.
- Persistence mode (RDB/AOF) and PVC sizing.

## Namespaces and Microservices

- Namespace(s) hosting Redis: _pending_
- Consuming application namespaces/services and their usage pattern (cache, session store, queue): _pending_ (cross-reference `component-contexts/context.kubernetes.md` application namespace list).

## Invocation Topology

_Pending — describe which application services read/write which Redis instance(s)._

## Relevant Configurations to Track in Inspection

- Memory usage vs. `maxmemory` and eviction policy.
- Replication lag between primary and replicas.
- Slow log entries and command latency.
- Connection count vs. configured limits.

## Read-Only Evidence Sources

- Kubernetes MCP: pod status, resource usage, restarts/OOMKills.
- Prometheus MCP (see `context.observability.md`): `redis_*` exporter metrics if scraped (memory, connected clients, keyspace hits/misses, replication offset).

## Open Items for Operator Confirmation

- Confirm Redis is in-scope for inspection and which namespace(s) host it.
- Confirm whether a redis_exporter feeds the observability stack.
