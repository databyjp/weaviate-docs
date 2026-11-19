---
title: Horizontal Scaling
sidebar_position: 30
description: "Multi-node cluster architecture and horizontal scaling strategies for high-availability Weaviate deployments."
image: og/docs/concepts.jpg
# tags: ['architecture', 'horizontal scaling', 'cluster', 'replication', 'sharding']
---

Weaviate can be scaled horizontally by being run on a set of multiple nodes in a cluster. This section lays out various ways in which Weaviate can be scaled, as well as factors to consider while scaling, and Weaviate's architecture in relation to horizontal scaling.

## Basic concepts

### Shards

A collection in Weaviate comprises of one or more "shards", which are the basic units of data storage and retrieval. A shard contains an object store, configured property indexes, and zero or more configured vector indexes. Each named vector has its own index configuration. Each shard can be hosted on a different node, allowing for distributed data storage and processing.

![Shards explained](./img/shards_explained.png)

*Illustration of a vector-indexed collection; the depicted vector index is not mandatory or necessarily the only vector index in a shard.*

The number of unique shards in a single-tenant collection can only be set at collection creation time. In most cases, letting Weaviate manage the number of shards is sufficient. But in some cases, you may want to manually configure the number of shards for performance or data distribution reasons.

In a multi-tenant collection, each tenant consists of one shard. This means that the number of unique shards in a multi-tenant collection is equal to the number of tenants.

![Shards in collections](./img/shards_in_collections.png)

*Illustrative shard organization; actual vector and property indexes depend on collection configuration.*

### Replicas

Depending on the setup, each shard can have one or more "replicas", to be hosted on different nodes. This is referred to as a "high availability" setup, where the same data is available on multiple nodes. This allows for better read throughput and fault tolerance.

The replication factor sets the desired number of shard copies. [`REPLICATION_MINIMUM_FACTOR`](/deploy/configuration/env-vars/index.md) is an enforced floor and the default when no collection factor is specified. A [collection factor](../manage-collections/multi-node-setup.mdx#replication-settings) must be at least that minimum, subject to available nodes and any configured maximum; it cannot override the floor downward.

## Motivation to scale Weaviate
Generally there are (at least) three distinct motivations to scale out horizontally which all will lead to different setups.

### Motivation 1: Maximum Dataset Size
Due to the [memory footprint of an HNSW graph](./resources.md#the-role-of-memory) it may be desirable to spread a dataset across multiple servers ("nodes"). In such a setup, a single collection may be split into shards and shards are spread across nodes. The disk-based [HFresh index](./indexing/vector-index.md#hfresh-index) can also reduce the need to shard purely for memory reasons.

Weaviate does the required orchestration at import and query time fully automatically.

See [Sharding vs Replication](#sharding-vs-replication) below for trade-offs involved when running multiple shards.

**Solution: Sharding across multiple nodes in a cluster**

:::note
The ability to shard across a cluster was added in Weaviate `v1.8.0`.
:::

### Motivation 2: Higher Query Throughput
When you receive more queries than a single Weaviate node can handle, it is desirable to add more Weaviate nodes which can help in responding to your users' queries.

Instead of sharding across multiple nodes, you can replicate (the same data) across multiple nodes. Configure replication and verify replica placement; merely adding nodes does not copy existing shards onto them. Sharding and replication can also be combined.

**Solution: Replicate your classes across multiple nodes in a cluster**

### Motivation 3: High Availability

When serving critical loads with Weaviate, it may be desirable to be able to keep serving queries even if a node fails completely. Such a failure could be either due to a software or OS-level crash or even a hardware issue. Replication can also support rolling maintenance when sufficient healthy replicas and capacity remain.

Availability requires enough healthy, operation-eligible replicas of every required shard, a reachable client entry point, and a majority of Raft voting members for metadata-dependent operations. For example, three eligible replicas with QUORUM require two successful responses; two replicas with QUORUM require both. See [consistency](./replication-architecture/consistency.md).

**Solution: Replicate your classes across multiple nodes in a cluster**

## Sharding vs Replication

The motivation sections above outline when it is desirable to shard your classes across multiple nodes and when it is desirable to replicate your classes - or both. This section highlights the implications of a sharded and/or replicated setup.

:::note
Benefits depend on actual placement and sufficient node resources. Nodes hosting none of a collection's relevant replicas do not add capacity for that collection. Adding nodes alone does not redistribute existing shards; use [replica movement](#shard-replica-movement) where appropriate.
:::

### Advantages when increasing sharding
* Run larger datasets
* Speed up imports.

Multiple shards can expose import parallelism, including on one node, but add overhead. Benchmark shard counts for the selected indexes, workload, and available resources rather than assuming more shards yield the fastest import.

### Disadvantages when increasing sharding
* Query fan-out and result merging add overhead; throughput effects depend on query patterns and bottlenecks.

### Advantages when increasing replication
* Availability can improve when enough eligible replicas remain healthy for each operation.
* Read throughput can improve when traffic is distributed across replicas; gains depend on consistency, workload, placement, and resource bottlenecks.

### Disadvantages when increasing replication
* Import speed does not improve when adding more replicated nodes

### Sharding Keys ("Partitioning Keys")
Weaviate uses specific characteristics of an object to decide which shard it belongs to. As of `v1.8.0`, a sharding key is always the object's UUID. The sharding algorithm is a 64bit Murmur-3 hash. Other properties and other algorithms for sharding may be added in the future.

Note that in a multi-tenant collection, each tenant consists of one shard.

## Shard replica movement

import ReplicaMovement from '/_includes/feature-notes/replica-movement.mdx';

<ReplicaMovement/>

A shard replica can be moved or copied from one node to another. This is useful when you want to balance the load across nodes or when you want to change the replication factor of a part of a collection.

[See this page](/docs/deploy/configuration/replica-movement.mdx) for more details on how to move shard replicas.

### Use cases for moving shard replicas

1. **Load Balancing**: If certain nodes are experiencing higher loads than others, moving shard replicas can help distribute the load more evenly across the cluster.

2. **Scaling**: If you need to scale your cluster (e.g., adding more nodes to handle increased load), shard replicas can be moved to the new nodes to improve distribution; balance is not automatic or guaranteed.

3. **Node Maintenance or Replacement**: If a node requires maintenance (e.g., hardware upgrades) or replacement, shard replicas can be moved to temporary or replacement nodes to support availability during maintenance, provided the remaining replicas and metadata voters meet the operations' requirements.

## Node Discovery

By default, Weaviate nodes in a cluster use a gossip-like protocol through [Hashicorp's Memberlist](https://github.com/hashicorp/memberlist) to communicate node state and failure scenarios.

Weaviate - especially when running as a cluster - is optimized to run on Kubernetes. The [Weaviate Helm chart](/deploy/installation-guides/k8s-installation.md#weaviate-helm-chart) makes use of a `StatefulSet` and a headless `Service` that automatically configures node discovery. All you have to do is specify the desired node count.

## Node affinity of shards and/or replication shards

Ordinary new-shard allocation uses disk-aware candidate ordering and distributes replicas across eligible nodes. This includes new tenant shards added to an existing multi-tenant collection, not just collection creation. Specialized placement constraints, such as namespace home-node placement, can restrict candidates.

Adding objects to an existing shard does not relocate it. Adding nodes also does not rebalance existing shards; use explicit replica movement to change their placement.

## Consistency and current limitations

* From `v1.25`, Weaviate uses the [Raft consensus algorithm](https://raft.github.io/) for cluster metadata such as collection definitions and tenant activity statuses. Raft is a log-based algorithm coordinated by an elected leader, so metadata progress requires a communicating majority of Raft voting members, not a majority of all cluster nodes, and concurrent schema changes are supported. For details, see [Replication architecture: Cluster metadata](/weaviate/concepts/replication-architecture/consistency.md#cluster-metadata).<br/>If you are a Kubernetes user, see the [`1.25 migration guide`](/deploy/migration/weaviate-1-25.md) before you upgrade. To upgrade, you have to delete your existing StatefulSet.
* Adding a node to an existing cluster does not by itself change the ownership of existing shards. To rebalance data across nodes, or to drain a node before you remove it, move its shard replicas with [replica movement](/deploy/configuration/replica-movement.mdx) as described in [Shard replica movement](#shard-replica-movement) above.

<details>
  <summary>Behavior before `v1.25` and `v1.32`</summary>

Prior to `v1.25`, schema changes were broadcast across the cluster with a form of two-phase transaction that could not tolerate node failures during the lifetime of the transaction. Raft replaced this mechanism. See [Replication architecture: Cluster metadata](/weaviate/concepts/replication-architecture/consistency.md#cluster-metadata) for the comparison.

Prior to `v1.32`, shard replicas could not be moved between nodes, so a node that still held data could not be removed from a cluster. [Replica movement](/deploy/configuration/replica-movement.mdx) removes that limitation.

</details>

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
