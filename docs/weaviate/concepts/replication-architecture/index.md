---
title: Replication Architecture
sidebar_position: 0
description: "Multi-node data replication design for high availability, reliability, and improved database performance."
image: og/docs/concepts.jpg
# tags: ['architecture']
---

Weaviate allows data replication across a multi-node cluster by [setting a replication factor](../../manage-collections/multi-node-setup.mdx#replication-settings) > 1. This enables a variety of [benefits](./motivation.md) such as [high availability](./motivation.md#high-availability-redundancy). Database replication improves reliability, scalability, and/or performance.

Weaviate utilizes multiple replication architectures:

- [Cluster metadata replication](./consistency.md#cluster-metadata) is managed by the [Raft](https://raft.github.io/) consensus algorithm.
- [Data replication](./consistency.md#data-objects) is [tunable](./consistency.md) and leaderless.

<details>
  <summary>What is the cluster <code>metadata</code>?</summary>

Weaviate cluster `metadata` includes collection definitions and tenant activity statuses.
<br/>

All cluster metadata is always replicated across all nodes, regardless of the replication factor.
<br/>

Note that this is different to object metadata, such as the object creation time. Object metadata is stored alongside the object data according to the specified replication factor.

</details>

In this Replication Architecture section, you will find information about:

* **General Concepts**, on this page
  * What is replication?
  * CAP Theorem
  * Why replication for Weaviate?
  * Replication vs. Sharding
  * How does replication work in Weaviate?
  * Roadmap


* **[Use Cases](./motivation.md)**
  * Motivation
  * High Availability
  * Increased (Read) Throughput
  * Zero Downtime Upgrades
  * Regional Proximity


* **[Philosophy](./philosophy.md)**
  * Typical Weaviate use cases
  * Reasons for a leaderless architecture
  * Gradual rollout
  * Large-scale testing


* **[Cluster Architecture](./cluster-architecture.md)**
  * Leaderless design
  * Replication Factor
  * Write and Read operations


* **[Consistency](./consistency.md)**
  * Cluster metadata
  * Data objects
  * Repairs


* **[Multi-DataCenter](./multi-dc.md)**
  * Regional Proximity

## What is replication?

<p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-QUORUM.png" alt="Example setup with replication" width="75%"/></p>

Database replication refers to keeping a copy of the same data point on multiple nodes of a cluster.

The resulting system is a distributed database. A distributed database consists of multiple nodes, all of which can contain a copy of the data. If enough eligible replicas of every required shard remain healthy and a live entry point is reachable, users can continue accessing data despite a node failure. In addition, query throughput can be improved with replication.

## CAP Theorem

The primary goal of introducing replication is to improve reliability. [Eric Brewer](https://en.wikipedia.org/wiki/Eric_Brewer_(scientist)) states that there are some limits on reliability for distributed databases, described by the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem). During a network partition, a distributed system cannot guarantee both linearizable consistency and availability for every request. CAP distinguishes these properties:
* **Consistency (C)** - Operations appear to occur in a single order consistent with real time (linearizability); this does not require every physical copy to be identical at every instant.
* **Availability (A)** - Every request receives a non-error response all the time, without the guarantee that it contains the most recent write.
* **Partition tolerance (P)** - The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes.

<p align="center"><img src="/img/docs/replication-architecture/repliction-cap.png" alt="CAP Theorem" width="60%"/></p>

Ideally, you want a database, like Weaviate, to have the highest reliability as possible, but this is limited by the tradeoff between consistency, availability and partition tolerance.

### Consistency vs Availability

:::tip Partition-time trade-off
When communication is partitioned, preserving a consistency guarantee can require refusing operations that cannot establish it. Accepting operations on disconnected replicas can preserve more availability but permit divergent state. The trade-off is operation-specific, not an unconditional menu of any two properties.
:::

C over A is preferred when the database contains critical data, such as transactional bank account data. For transactional data, you want the data to always be consistent (otherwise your bank balance is not guaranteed to be correct if you make transactions while some nodes (e.g. ATMs) are down).

When a database involves less-critical data, A over C can be preferred. An example can be a messaging service, where you can tolerate showing some old data but the application should be highly available and handle large amounts of writes with minimal latency.

Weaviate generally follows this latter design, since Weaviate typically deals with less critical data and is used for approximate search as a secondary database in use cases with more critical data. More about this design decision in [Philosophy](./philosophy.md). However, you can use Weaviate's [tunable consistency](./consistency.md#tunable-consistency-strategies) options according to your needs.

## Why replication for Weaviate?

Weaviate, as a database, must provide reliable answers to users' requests. As discussed above, database reliability consists of various parts. Below are Weaviate use cases in which replication is desired. For detailed information, visit the [Replication Use Cases (Motivation) page](./motivation.md).

1. **High availability (redundancy)**<br/>
  Replicas can keep operations available when enough eligible replicas of every required shard remain healthy. A live client entry point and, where metadata is needed, a Raft voting majority are also required. Failover and retries can be visible to clients.
2. **Increased (read) throughput**<br/>
  Additional replicas can distribute read work, especially at `ONE`. Scaling depends on workload, placement, traffic distribution, and shared bottlenecks; it is not a guaranteed linear multiplier.
3. **Rolling upgrades**<br/>
  Replication can support continued service during rolling upgrades if the remaining replicas and voters meet operational requirements. Follow the [migration guide](/deploy/migration/index.md), including version-specific steps and readiness checks.
4. **Regional proximity**<br/>
  A [single cluster spanning data centers](./multi-dc.md) can place needed replicas closer to users. Required remote acknowledgments and metadata coordination can still incur WAN latency; geography alone does not guarantee local service or datacenter-failure survival.


## Replication vs. Sharding

Replication is not the same as [sharding](../cluster.md). Sharding refers to horizontal scaling, and was introduced to Weaviate in v1.8.

* **Replication** copies the data to different server nodes. For Weaviate, this increases data availability and provides redundancy in case a single node fails. Query throughput can be improved with replication.
* **Sharding** handles horizontal scaling across servers by dividing the data and sending the pieces of data (shards) to multiple replica sets. The data is thus divided, and all shards together form the entire set of data. You can use sharding with Weaviate to run larger datasets and speed up imports.

<p align="center"><img src="/img/docs/replication-architecture/replication-replication-vs-sharding.png" alt="Replication vs Sharding" width="60%"/></p>

Replication and sharding can be combined in a setup, to improve throughput and availability as well as import speed and support for large datasets. For example, you can have 3 replicas of the database and shards set to 3, which means you have 9 shards in total, where each server node holds 3 different shards.

## How does replication work in Weaviate?

### Cluster metadata replication

Weaviate’s cluster metadata changes are managed through Raft to provide consistency across the cluster. (This includes collection definitions and tenant activity statuses.)

From Weaviate `v1.25`, cluster metadata changes are committed using the Raft consensus algorithm. Raft is a leader-based consensus algorithm. A leader node is responsible for cluster metadata changes. Raft ensures that these changes are consistent across the cluster, while a majority of Raft voting members can communicate. Non-voting nodes do not count toward that majority.

<details>
  <summary>Metadata replication pre-<code>v1.25</code></summary>

Prior to Weaviate `v1.25`, each cluster metadata change was recorded via a distributed transaction with a two-phase commit.
<br/>

This is a synchronous process, which means that the cluster metadata change is only committed when all nodes have acknowledged the change. In this architecture, any node downtime would temporarily prevent metadata operations. Additionally, only one such operation could be processed at a time.

If you are using Weaviate `v1.24` or earlier, you can [upgrade to `v1.25`](/deploy/migration/weaviate-1-25.md) to benefit from the Raft consensus algorithm for cluster metadata changes.

</details>

### Data replication

In Weaviate, availability is generally favored over consistency. Weaviate's data replication uses a leaderless design, which means there are no primary and secondary nodes. Clients can connect directly to a Weaviate node or through an optional load balancer provided by their deployment. The receiving node coordinates supported data operations with the required shard replicas.

Acknowledgment thresholds for supported data operations are tunable to `ONE`, `QUORUM`, or `ALL`. These are thresholds over a shard's routed replicas, not all cluster nodes. In v1.40.5's inspected ordinary object routes, QUORUM requires `floor(N/2)+1` successful answers from the operation-eligible set; ALL requires successful answers from all membership-resolved eligible destinations after a majority-reachability gate. Membership resolution does not guarantee RPC success. See the [version-scoped explanation](./consistency.md#tunable-write-consistency); successful ALL does not prove every configured copy is current.

Writes attempt routed destinations regardless of the acknowledgment threshold. Returning after enough acknowledgments is distinct from [background anti-entropy](./consistency.md#async-replication), all-copy completion, or immediate vector-index visibility.

Each shard has its own replica placement within the cluster, and those sets can differ across a collection. Replication factor is not the total cluster size.

Read more about how replication works in Weaviate in [Philosophy](./philosophy.md), [Cluster Architecture](./cluster-architecture.md) and [Consistency](./consistency.md).

## How do I use replication in Weaviate?

See [how to configure replication](/deploy/configuration/replication.md). You can enable replication in the collection definition. In queries, you can [specify the desired consistency level](../../search/basics.md#replication).

## Related pages
- [Configuration: Replication](/deploy/configuration/replication.md)

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
