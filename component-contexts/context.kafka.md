# Kafka Component Context (Template)

> Status: template — not yet populated from a live MCP snapshot. Copy the structure used in `context.kubernetes.md` once a Kafka cluster is in scope.

## Snapshot Metadata

- Date (UTC): _pending_
- Cluster/operator: _e.g. Strimzi (`strimzi-system` observed in `context.kubernetes.md` section 3.1), Confluent, or self-managed_
- Broker count / replication factor defaults: _pending_

## Architecture

- Brokers, controllers (KRaft or ZooKeeper-based — `zookeeper` namespace is observed in the cluster inventory), and any schema registry.
- Storage class / PVC sizing per broker.

## Namespaces and Microservices

- Namespace(s) hosting brokers: _pending_
- Producer/consumer application namespaces and their topics: _pending_ (cross-reference `component-contexts/context.kubernetes.md` application namespace list).

## Invocation Topology

_Pending — describe producer -> topic -> consumer group flows once discovered._

## Relevant Configurations to Track in Inspection

- Under-replicated partitions, ISR shrink events.
- Consumer group lag per topic/partition.
- Broker disk utilization and retention policy per topic.
- Topic partition count vs. consumer parallelism.

## Read-Only Evidence Sources

- Kubernetes MCP: broker/StatefulSet pod status, PVC usage, events/restarts.
- Prometheus MCP (see `context.observability.md`): `kafka_server_*` JMX exporter metrics if scraped, consumer lag exporter metrics if present.

## Open Items for Operator Confirmation

- Confirm Kafka is in-scope for inspection and which namespace(s) host it.
- Confirm whether a JMX/lag exporter feeds the observability stack.
