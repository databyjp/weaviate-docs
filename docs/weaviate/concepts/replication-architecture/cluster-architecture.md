---
title: Cluster Architecture
sidebar_position: 3
description: "Node behavior and cluster coordination mechanisms in Weaviate's distributed replication system."
image: og/docs/concepts.jpg
# tags: ['architecture']
---


This page describes how the nodes or clusters in Weaviate's replication design behave.

In Weaviate, metadata replication and data replication are separate. For the metadata, Weaviate uses the [Raft](https://raft.github.io/) consensus algorithm. For data replication, Weaviate uses a leaderless design with eventual consistency.

## Node Discovery

By default, Weaviate nodes in a cluster use a gossip-like protocol through [Hashicorp's Memberlist](https://github.com/hashicorp/memberlist) to communicate node state and failure scenarios.

Weaviate is optimized to run on Kubernetes, especially when operating as a cluster. The [Weaviate Helm chart](/deploy/installation-guides/k8s-installation.md#weaviate-helm-chart) makes use of a `StatefulSet` and a headless `Service` that automatically configures node discovery.

## Metadata replication: Raft

Weaviate uses the [Raft consensus algorithm](https://raft.github.io/) for metadata replication, implemented with Hashicorp's [raft library](https://pkg.go.dev/github.com/hashicorp/raft). Metadata in this context includes collection definition and shard/tenant states.

Raft ensures that metadata changes are consistent across the cluster. A metadata change is forwarded to the leader node, which applies the change to its log before replicating it to the follower nodes. Once a majority of Raft voting members have acknowledged the change, the leader commits the change to the log. The leader then notifies the followers, which apply the change to their logs.

This architecture ensures that metadata changes are consistent across the cluster, while a majority of Raft voting members can communicate and make progress. Non-voting members do not count toward that majority.

As a result, a Weaviate cluster will include a leader node that is responsible for metadata changes. The leader node is elected by the Raft algorithm and is responsible for coordinating metadata changes.

## Data replication: Leaderless

Weaviate uses a leaderless architecture for data replication. This means there is no central leader or primary node that will replicate to follower nodes. Instead, all nodes can accept writes and reads from the client, which can offer better availability. Data operations do not depend on a fixed primary, but still need a reachable coordinator and enough eligible replicas of every required shard. A leaderless replication approach, also known as [Dynamo-style](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) data replication (after Amazon's implementation), has been adopted by other open-source projects like [Apache Cassandra](https://cassandra.apache.org).

In Weaviate, a coordination pattern is used to relay a client's read and write requests to the correct nodes. Unlike in a leader-based database, a coordinator node does not enforce any ordering of the operations.

The following illustration shows a leaderless replication design in Weaviate. There is one coordination node, which leads traffic from the client to the correct replicas. There is nothing special about this node; it was chosen to be the coordinator because this node received the request from the load balancer. A future request for the same data may be coordinated by a different node.

<p align="center"><img src="/img/docs/replication-architecture/replication-main-quorum.png" alt="Replication Architecture" width="75%"/></p>

A leaderless data design avoids routing every write through a fixed primary. Another node can coordinate a request if the entry point and required shard replicas remain available; this is not an unconditional failover guarantee. Metadata changes separately require a Raft voting majority.

On the flipside of high availability, a leaderless database tends to be less consistent. Because there is no leader node, data on different nodes may temporarily be out of date. Leaderless databases tend to be eventually consistent. Consistency in Weaviate is [tunable](./consistency.md), but stronger acknowledgment requirements can reduce availability.

[Background async replication and read repair](./consistency.md#repairs) attempt to reconcile divergent replica state, subject to conflict policy and successful communication. They are distinct from the acknowledgment threshold of an individual request.


## Replication Factor

import RaftRFChangeWarning from '/_includes/1-25-replication-factor.mdx';

<RaftRFChangeWarning/>

In Weaviate, data replication is enabled and controlled per collection. This means you can have different replication factors for different collections.

The replication factor (RF or n) determines how many copies of data are stored in the distributed setup. A replication factor of 1 means that there is only 1 copy of each data entry in the database setup, in other words there is no replication. A replication factor of 2 means that there are two copies of each data entry, which are present on two different nodes (replicas). Naturally, the replication factor cannot be higher than the number of nodes. Any node in the cluster can act as a coordinating node to lead queries to the correct target node(s).

Data quorum is calculated for the target shard's operation-eligible replica set, not all cluster nodes: `QUORUM = floor(n/2) + 1`. With three eligible replicas, two successful responses satisfy QUORUM, permitting one unavailable replica for that operation. RF=2/QUORUM needs both replicas; RF=4/QUORUM needs three. Odd replica counts can offer useful majority/cost trade-offs, but do not themselves resolve object conflicts or require an odd total cluster size. Metadata progress instead requires a majority of Raft voting members.

<p align="center"><img src="/img/docs/replication-architecture/replication-factor.png" alt="Replication Factor" width="75%"/></p>

## Write operations

On a write operation, the client's request will be sent to any node in the cluster. The first node which receives the request is assigned as the coordinator. The coordinator node sends the request to a number of predefined replicas and returns the result to the client. So, any node in the cluster can be a coordinator node. A client will only have direct contact with this coordinator node. Before sending the result back to the client, the coordinator node waits for a number of write acknowledgments from different nodes depending on the configuration. How many acknowledgments Weaviate waits for, depends on the [consistency configuration](./consistency.md).

**Steps**
1. The client sends data to any node, which will be assigned as the coordinator node
2. The coordinator attempts the write on the routed replicas of the target shard.
3. The coordinator waits for the required successful acknowledgments from those replicas. Ordinary object writes default to `QUORUM`; see [operation-specific consistency](./consistency.md#tunable-write-consistency).
4. When the required successful acknowledgments arrive, the request can succeed. Other replicas may still require repair; a failed request can also have partial effects.

As an all-healthy example, consider one shard with three eligible replicas on a three-node cluster. The coordinator attempts the write on all three destinations; QUORUM requires two successful acknowledgments.

<p align="center"><img src="/img/docs/replication-architecture/replication-rf3-size3.png" alt="Replication Factor 3 with cluster size 3" width="75%"/></p>

In an eight-node cluster, one shard with RF=3 normally has three replica destinations, not eight. A collection can span several shards with different replica sets. The coordinator uses current recorded shard placement and read/write eligibility; object-to-shard mapping is separate from shard-to-node placement, which can change through replica movement. The illustration below is a single-shard, all-healthy example.

<p align="center"><img src="/img/docs/replication-architecture/replication-rf3-size8.png" alt="Replication Factor 3 with cluster size 8" width="75%"/></p>

## Read operations

Read operations are routed by a server-side coordinator to replicas of the required shards.

- **Object by ID:** the coordinator can compare a full object and replica digests according to the operation's consistency requirement and repair consulted state using timestamps and deletion policy. This does not guarantee a globally newest value; `ONE` can return stale data.
- **Search:** shard indexes discover candidates. Requested consistency checks on known objects are separate from candidate discovery, and are not a merge of every replica's search results. Do not assume every search path performs the same checks or repairs.

The client receives the server's result; it does not perform replica conflict reconciliation. See [read consistency](./consistency.md#tunable-read-consistency) and [query consistency limitations](./consistency.md#tunable-consistency-and-queries).

Each shard has its own replica placement, so a multi-shard query can involve more nodes than the collection's replication factor. Read fan-out and acknowledgment requirements are operation-specific, not a count of all cluster nodes.

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
