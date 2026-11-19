---
title: Use Cases (Motivation)
sidebar_position: 1
description: "Four key use cases demonstrating the benefits and configuration requirements for Weaviate replication."
image: og/docs/concepts.jpg
# tags: ['architecture']
---

On this page you will find four use cases which motivate replication for Weaviate. Each of them serves a different purpose and, as a result, may require different configuration.

## High Availability (Redundancy)

High availability of a database means that the database is designed to operate continuously without service interruptions. That means that the database system should be tolerant to failures and errors, which should be handled automatically.

Replication provides redundant shard copies that can serve requests when sufficient eligible replicas remain healthy; it is not an unconditional service-continuity guarantee.

Weaviate considers cluster metadata operations critical, so it treats them differently than it does data operations.

From Weaviate `v1.25`, Weaviate uses the Raft consensus algorithm for cluster metadata replication. This is a leader-based consensus algorithm, where a leader node is responsible for cluster metadata changes. Use of Raft ensures that cluster metadata changes are consistent across the cluster, while a majority of Raft voting members can communicate. Non-voting nodes do not count toward that majority.

Prior to Weaviate `v1.25`, Weaviate used a leaderless design with two-phase commit for cluster metadata operations. This required all nodes for a cluster metadata operation such as a collection definition update, or a tenant state update. This meant that one or more nodes being down temporarily prevented cluster metadata operations. Additionally, only one cluster metadata operation could be processed at a time.

Data operations can remain available after a node fails if each required shard retains enough healthy, eligible replicas and the client can reach a live entry point. Internal routing can try another replica, but timeouts, retries, or errors can still be visible. Metadata-dependent operations separately require the Raft voting majority.

Examples of applications where High Availability is desired are emergency services, enterprise IT systems, social media, and website search. Nowadays, users are used to highly available applications, so they expect little to no downtime. For e.g. website search, service (read queries) should not be interrupted if a node goes down. In that case, if writing is temporarily unavailable, it is acceptable and in the worst case scenario the site search will be stale, but still available for read requests.

<p align="center"><img src="/img/docs/replication-architecture/replication-high-availability.png" alt="High Availability" width="75%"/></p>


For existing available data, these examples describe replicas **per required shard**, not all cluster nodes:

1. **RF3, QUORUM reads/writes:** two successful replica responses are required, so one replica can be unavailable for these operations.
2. **RF2, QUORUM reads/writes:** both replicas are required; one unavailable replica prevents quorum.
3. **ONE reads/writes:** one successful replica response can suffice for supported data operations, but stale state is possible. This does not remove metadata or entry-point availability requirements.

In v1.40.5's ordinary routed object paths, `ALL` is based on membership-resolved eligible destinations after a majority-reachability gate, not necessarily every configured copy. A selected destination can still fail. See [operation-specific thresholds and limitations](./consistency.md#tunable-write-consistency), and distinguish acknowledgment from [background repair](./consistency.md#repairs).


## Increased (Read) Throughput
Replication can increase read throughput by distributing requests across shard replicas, especially with `ONE`. Gains depend on shard placement, read/write mix, traffic distribution, hardware, network, and other bottlenecks. Replication factor counts shard copies, not every server in the cluster.

The numerical QPS illustration below is an idealized scaling example, not benchmark results or guaranteed capacity.

<p align="center"><img src="/img/docs/replication-architecture/replication-increased-throughput.png" alt="Increased Throughput" width="75%"/></p>


## Zero Downtime Upgrades

Without replication, there is a window of downtime when you update a Weaviate instance. The single node needs to stop, update and restart before it's ready to serve again. With replication, rolling upgrades can maintain service if enough replicas and Raft voters remain healthy, traffic reaches ready nodes, and the remaining capacity is sufficient. Follow the [migration guide](/deploy/migration/index.md), including backups and version-specific steps; wait for each node to be ready and metadata to synchronize before proceeding.

As an example, consider you're updating the version of a Weaviate instance from v1.19 to v1.20. Without replication there is a window of downtime:
1. Node is ready to serve traffic
2. Node is stopped, no requests can be served
3. Node image is replaced with newer version
4. Node is restarted
5. Node takes time to be ready
6. Node is ready to serve traffic.

From step 2 until step 6 the Weaviate server cannot receive and respond to any requests. This leads to bad user experience.

With replication (e.g. replication factor of 3), upgrades to the Weaviate version are done using a rolling update. The sequence below is illustrative: proceed one node at a time only after readiness and synchronization checks, retaining sufficient healthy replicas and voters for the operations being served.
1. 3 nodes ready to serve traffic
2. node 1 being replaced, nodes 2,3 can serve traffic
3. node 2 being replaced, nodes 1,3 can serve traffic
4. node 3 being replaced, nodes 1,2 can serve traffic

<p align="center"><img src="/img/docs/replication-architecture/replication-zero-downtime.gif" alt="Zero downtime upgrades" width="75%"/></p>


## Regional Proximity

When users are located in different regional areas (e.g. Iceland and Australia as extreme examples), you cannot ensure low latency for all users due to the physical distance between the database server and the users. You can only place the database server at one geographical location, so the question arises where you put the server:
1. Option 1 - Put the cluster in the middle (e.g. India).<br/>
   All users will have relatively high latency, since data needs to travel between Iceland and India, and Australia and India.

    <p align="center"><img src="/img/docs/replication-architecture/replication-regional-proximity-1.png" alt="Cluster in the geographical middle" width="75%"/></p>

2. Option 2 - Put the cluster close to one user group (e.g. Iceland)<br/>
   Users from Iceland have very low latency while users from Australia experience relatively high latency since data needs to travel a long distance.

   Another option is to run one cluster whose nodes and shard replicas span regions, as described in [Multi-DC replication](./multi-dc.md).
3. Option 3 - One cluster with nodes and required shard replicas in both Iceland and Australia.<br/>
   Nearby replica placement and suitable routing can reduce user-to-replica latency. Operations requiring remote acknowledgments or metadata coordination still incur WAN latency. The regional groups in the illustration are parts of one cluster, not independently managed clusters.

    <p align="center"><img src="/img/docs/replication-architecture/replication-regional-proximity-3.png" alt="Replication multi-dc" width="75%"/></p>

Cross-region replicas can improve resilience, but surviving datacenter loss requires sufficient healthy eligible replicas of every needed shard, a reachable entry point, and a Raft voting majority for metadata-dependent operations. Spreading arbitrary nodes across regions is not sufficient.

:::note
Regional Proximity is enabled by [running a Weaviate cluster across data centers](./multi-dc.md).
:::

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
