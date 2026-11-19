---
title: Consistency
sidebar_position: 4
description: "Replication factor configuration and data consistency models across Weaviate cluster replicas."
image: og/docs/concepts.jpg
# tags: ['architecture']
---

import SkipLink from '/src/components/SkipValidationLink'

The replication factor in Weaviate determines how many copies of shards (also called replicas) will be stored across a Weaviate cluster.

<p align="center"><img src="/img/docs/replication-architecture/replication-factor.png" alt="Replication factor" width="80%"/></p>

When the replication factor is > 1, consistency models balance the system's reliability, scalability, and/or performance requirements.

Weaviate uses multiple consistency models. One for its cluster metadata and another for its data objects.

### Consistency models in Weaviate

Weaviate uses the [Raft](https://raft.github.io/) consensus algorithm for [cluster metadata replication](./cluster-architecture.md#metadata-replication-raft). Cluster metadata in this context includes the collection definitions and tenant activity statuses. This allows cluster metadata updates to occur even when some nodes are down.

Data objects are replicated using a [leaderless design](./cluster-architecture.md#data-replication-leaderless) using tunable consistency levels. So, data operations can be tuned to be more consistent or more available, depending on the desired tradeoff.

These designs reflect the trade-off between consistency and availability that is described in the [CAP Theorem](./index.md#cap-theorem).

:::tip Quorum overlap is not a global consistency guarantee
For fixed replica sets, `r + w > n` describes overlap between read and write acknowledgment sets. It is not by itself a guarantee of linearizability, a query snapshot, or complete candidate discovery. The routed replica set and guarantees differ by operation; see the version-scoped note below and [query limitations](#tunable-consistency-and-queries).
:::

## Cluster metadata

The cluster metadata in Weaviate makes use of the Raft algorithm.

Weaviate uses the [Raft](https://raft.github.io/) consensus algorithm for cluster metadata replication. Raft is a consensus algorithm with an elected leader node that coordinates replication across the cluster using a log-based approach.

As a result, each request that changes the cluster metadata will be sent to the leader node. The leader node will apply the change to its logs, then propagate the changes to the follower nodes. Once a majority of Raft voting members has acknowledged the cluster metadata change, the leader node will commit the change and confirm it to the client.

This architecture ensures that cluster metadata changes are consistent across the cluster, while a majority of Raft voting members remains able to communicate. Non-voting nodes do not count toward this quorum.

<details>
  <summary>Pre-<code>v1.25</code> cluster metadata consensus algorithm</summary>

Prior to using Raft, a cluster metadata update was done via a [Distributed Transaction](https://en.wikipedia.org/wiki/Distributed_transaction) algorithm. This is a set of operations that is done across databases on different nodes in the distributed network. Weaviate used a [two-phase commit (2PC)](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) protocol, which replicates the cluster metadata updates in a short period of time (milliseconds).

A clean (without fails) execution has two phases:
1. The commit-request phase (or voting phase), in which a coordinator node asks each node whether they are able to receive and process the update.
2. The commit phase, in which the coordinator commits the changes to the nodes.

</details>

### Collection definition requests in queries

Some queries require the collection definition. Prior to the introduction of this feature, every such query led to the local (requesting) node to fetch the collection definition from the leader node. This meant that the definition was strongly consistent, but it could lead to additional traffic and load.

Where available, the `COLLECTION_RETRIEVAL_STRATEGY` [environment variable](/deploy/configuration/env-vars/index.md#multi-node-instances) can be set to `LeaderOnly`, `LocalOnly`, or `LeaderOnMismatch`.

- `LeaderOnly` (default): Always requests the definition from the leader node. This is the most consistent behavior but can lead to higher intra-cluster traffic.
- `LocalOnly`: Always use the local definition; leading to eventually consistent behavior while reducing intra-cluster traffic.
- `LeaderOnMismatch`: Checks if the local definition is outdated, and requests the definition if necessary. Balances consistency and intra-cluster traffic.

The default behavior is `LeaderOnly` to achieve strong consistency. However, `LocalOnly` and `LeaderOnMismatch` can be used to reduce intra-cluster traffic according to the desired consistency level.

## Data objects

Replicated object writes use a two-phase prepare/commit mechanism with tunable acknowledgment thresholds. For example, with five eligible, membership-resolvable shard replicas, the coordinator attempts the write on all five and QUORUM requires three successful acknowledgments. This is not an all-or-nothing transaction across replicas: a failed request can leave partial commits. Acknowledgment also does not imply immediate vector-index visibility.

As a result, data objects in Weaviate are eventually consistent. Eventual consistency provides BASE semantics:

* **Basically available**: reading and writing operations are as available as possible
* **Soft-state**: there are no consistency guarantees since updates might not yet have converged
* **Eventually consistent**: background and read-time repair aim to converge replica state when communication and repairs succeed and conflicting writes cease.

Weaviate uses eventual consistency to improve availability. Read and write consistency are tunable, so you can tradeoff between availability and consistency to match your application needs.

*The animation below is an example of how a write or a read is performed with Weaviate with a replication factor of 3 and 8 nodes. The blue node acts as the coordinator node. The consistency level is set to `QUORUM`, so the coordinator node only waits for two out of three responses before sending the result back to the client.*

<p align="center"><img src="/img/docs/replication-architecture/replication-quorum-animation.gif" alt="Write consistency QUORUM" width="75%"/></p>

### Tunable write consistency

Adding or changing data objects are **write** operations.

:::note
Write operations are tunable starting with Weaviate v1.18, to `ONE`, `QUORUM` (default) or `ALL`. In v1.17, write operations are always set to `ALL` (highest consistency).
:::

Write consistency controls successful acknowledgments, not how many destinations are attempted. The coordinator prepares all routed destinations and commits successful prepare destinations; it can return after the required successful commits. Unavailable or failed replicas may need later repair. A failure is not a guarantee that no replica applied the write.

:::note Routed object consistency in v1.40.5
For the inspected ordinary routed object writes and object-by-ID reads, the router first selects operation-eligible replicas of each shard (count `N`), then resolves their destinations through cluster membership (resolved count `R`). `QUORUM` and `ALL` require at least a majority of `N` to be membership-resolvable. Successful-answer requirements are `ONE = 1`, `QUORUM = floor(N/2) + 1`, and `ALL = R`.

Membership resolution is not proof of readiness or successful RPCs: a selected destination can still fail. Thus `ALL` is not necessarily every configured stored copy. For example, if all three RF3 replicas are operation-eligible but one is excluded by membership resolution, the two remaining destinations pass the majority gate; routed `ALL` then requires both to succeed. A successful request does not prove the excluded copy is current. Some internal read plans skip this gate, and search/aggregation do not inherit a universal object-ID contract.
:::

* **ONE** - at least one successful replica acknowledgment.
* **QUORUM** - at least `floor(N/2) + 1` successful acknowledgments. For six eligible replicas, that is four, provided enough destinations can be resolved and answer successfully.
* **ALL** - every membership-resolved, operation-eligible destination must acknowledge successfully, subject to the majority gate above. This can require more successful answers than QUORUM, but does not guarantee permanent convergence.

*Figure below: a replicated Weaviate setup with write consistency of ONE. There are 8 nodes in total out of which 3 replicas.*

<p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-ONE.png" alt="Write consistency ONE" width="60%"/></p>

*Figure below: a replicated Weaviate setup with Write Consistency of `QUORUM` (n/2+1). There are 8 nodes in total, out of which 3 replicas.*


<p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-QUORUM.png" alt="Write consistency QUORUM" width="60%"/></p>

*All-healthy illustration: one shard has three eligible, membership-resolvable replicas among eight cluster nodes. In this case, routed `ALL` requires all three successful acknowledgments.*

<p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-ALL.png" alt="Write consistency ALL" width="60%"/></p>


### Tunable read consistency

Supported object reads can request `ONE`, `QUORUM`, or `ALL`. Defaults and enforcement are operation-specific; direct object-by-ID reads are not the same as searches or aggregations. For routed object-by-ID reads, the version-scoped eligible/resolvable-set note above applies, including its `ALL` limitation.

:::note
Prior to `v1.18`, read consistency was tunable only for [requests that obtained an object by id](../../manage-objects/read.mdx#get-an-object-by-id), and all other read requests had a consistency of `ALL`.
:::

The following consistency levels are applicable to most read operations:

- Starting with `v1.18`, consistency levels are applicable to REST endpoint operations.
- Starting with `v1.19`, consistency levels are applicable to GraphQL `Get` requests.
- Do not infer identical consistency controls or enforcement for every gRPC operation, particularly aggregation. Consult the operation-specific API guidance.

* **ONE** - a read response must be returned by at least one replica. This is the fastest (most available), but least consistent option.
* **QUORUM** - for routed object-by-ID reads, at least `floor(N/2) + 1` successful answers from operation-eligible shard replicas, as defined above. With six eligible replicas, that is four; this is not a quorum of every node in the cluster.
* **ALL** - for routed object-by-ID reads in v1.40.5, successful answers are required from every membership-resolved eligible destination after the majority gate, not necessarily every configured copy. A selected destination's failure can still fail the read. Do not apply this as an unconditional search guarantee.

Examples:
* **ONE**<br/>
  In a single datacenter with a replication factor of 3 and a read consistency level of ONE, the coordinator node will wait for a response from one replica node.

  <p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-ONE.png" alt="Write consistency ONE" width="60%"/></p>

* **QUORUM**<br/>
  In a single datacenter with a replication factor of 3 and a read consistency level of `QUORUM`, the coordinator node will wait for n / 2 + 1 = 3 / 2 + 1 = 2 replicas nodes to return a response.

  <p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-QUORUM.png" alt="Write consistency QUORUM" width="60%"/></p>

* **ALL**<br/>
  In this all-healthy single-shard illustration, all three RF3 replicas are eligible and membership-resolvable, so routed object-by-ID `ALL` requires three successful answers (a full object plus replica-state information, not necessarily three full object bodies). The version-scoped routing caveat above applies when membership changes.

  <p align="center"><img src="/img/docs/replication-architecture/replication-rf3-c-ALL.png" alt="Write consistency ALL" width="60%"/></p>

### Tunable consistency strategies

Depending on the desired tradeoff between consistency and speed, below are three common consistency level pairings for write / read operations. These are common acknowledgment trade-offs for supported operations, not guarantees of convergence, complete query results, or fixed relative latency:
* `QUORUM` / `QUORUM` => balanced write and read latency
* `ONE` / `ALL` => fast write and slow read (optimized for write)
* `ALL` / `ONE` => slow write and fast read (optimized for read)

### Tunable consistency and queries

Note that tunable consistency levels for read operations do not affect consistency of the list of objects returned by a query. In other words, the list of object UUIDs returned by a query depends only on the coordinator node's (and any other required shards') local index, and is independent of the read consistency level.

This is due to the fact that each query is performed by the coordinator node and any other shards required to answer the query. Even if the read consistency level is set to `ALL`, it does not mean that multiple replicas will be queried and the results merged together.

Where the read consistency level is applied is in retrieving the identified objects from the replicas. Do not infer that a successful search at `ALL` proves every configured replica was consulted or every requested check succeeded. Candidate discovery and subsequent known-object checking are separate; the routed object-by-ID thresholds above are not a universal search contract.

In other words, the read consistency level only affects which versions of the objects are retrieved, but it does not lead to a more (or less) consistent query result.

:::note When might this occur?

Replicated object writes attempt their routed shard destinations on insert/update/delete; acknowledgments do not imply that every copy or local vector index has already converged. So, most of the time this won't matter as all shards will have identical local indexes to each other. This is a rare care which may only occur if there is a problem, such as a node being down, or there is a network problem.

:::

### Tenant states and data objects

Each tenant in a [multi-tenant collection](../data.md#multi-tenancy) has a configurable [tenant state](../../starter-guides/managing-resources/tenant-states.mdx), which determines the availability and location of the tenant's data. The tenant state can be set to `active`, `inactive`, or `offloaded`.

An `active` tenant's data should be available for queries and updates, while `inactive` or `offloaded` tenants are not.

However, there can be a delay between the time a tenant state is set, and when the tenant's data reflects the (declarative) tenant state.

As a result, a tenant's data may be available for queries for a period of time even if the tenant state is set to `inactive` or `offloaded`. Conversely, a tenant's data may not be available for queries and updates for a period of time even if the tenant state is set to `active`.

:::info Why is this not addressed by repair-on-read?
For speed, data operations on a tenant occur independently of any tenant activity status operations. As a result, tenant states are not updated by repair-on-read operations.
:::

## Repairs

In distributed systems like Weaviate, object replicas can become inconsistent due to any number of reasons - network issues, node failures, or timing conflicts. When Weaviate detects inconsistent data across replicas, it attempts to repair the out of sync data.

Weaviate uses [async replication](#async-replication), [deletion resolution](#deletion-resolution-strategies) and [repair-on-read](#repair-on-read) strategies to maintain consistency across replicas.

### Async replication

Async replication is background anti-entropy for replicas of the same shard. It periodically compares and propagates state to attempt convergence; communication failures, concurrent updates, conflict policies, and resource limits can delay or prevent a repair cycle from completing.

It uses a Merkle tree (hash tree) algorithm to monitor and compare the state of nodes within a cluster. If it identifies an inconsistency, it attempts to reconcile the differing state.

Repair-on-read works well with one or two isolated repairs. Async replication is effective in situations where there are many inconsistencies. For example, if an offline node misses a series of updates, async replication can reconcile missed updates after the node returns, without a fixed recovery-time guarantee.

Async replication supplements the repair-on-read mechanism. Applicable read-repair paths may detect inconsistencies among the objects and replicas they consult; they do not inspect every object on every read.

As of Weaviate `v1.38`, async replication is enabled by default for any collection with a replication factor greater than `1`, there is no per-collection flag to enable it. To turn it off cluster-wide, set the `ASYNC_REPLICATION_DISABLED` environment variable to `true`. Visit the [How-to: Replication](/deploy/configuration/replication.md#async-replication-settings) page to learn more about the available async replication settings.

#### Memory and performance considerations for async replication

Async replication uses a hash tree to compare and synchronize data between the database cluster nodes, based on objects' latest update time. The digest-array component grows exponentially with hash-tree height (`H`). A higher tree provides finer comparison granularity but uses more memory; it does not guarantee faster overall replication. Bitsets, metadata, workers, temporary buffers, and other shard structures add overhead beyond the digest array.

The trade-offs can be summarized like this:
  - **Higher** `H`: More digest memory and finer comparison granularity.
  - **Lower** `H`: Less digest memory and coarser comparison granularity.

:::tip Memory management for multi-tenancy
Each tenant is backed by a shard. Therefore, when there is a high number of tenants, the memory consumption of async replication can be significant. For example, 1,000 initialized tenant shard replicas actually hosted on one node need approximately 2 GB of digest arrays at height 16 or 34 GB at height 20, before other overhead. This is component arithmetic, not measured process RAM.
<br/>

As of `v1.36`, multi-tenant collections default to a hash tree height of `10` (32,752 bytes, approximately 32 KiB, of digest storage per initialized local tenant shard replica), significantly reducing memory overhead compared to the single-tenant default of `16`.
<br/>

To further reduce memory consumption, reduce the hash tree height. This changes comparison granularity and can increase repair work; measure its workload-specific effect.
:::

Use the following formulas and examples as a quick reference:

##### Memory calculation

- **Total number of nodes in the hash tree:**
  For a hash tree with height `H`, the total number of nodes is:
  ```
  Number of hash tree nodes = 2^(H+1) - 1 ≈ 2^(H+1)
  ```

- **Digest-array memory (per initialized local shard replica):**
  Each digest occupies **16 bytes**; this excludes the bitset and other runtime overhead.
  ```
  Digest-array bytes = (2^(H+1) - 1) * 16
  ```

##### Examples

- Hash tree with height `16`:
  - `Total hash tree nodes ≈ 2^(16+1) = 131,072`
  - `Memory required ≈ 131072 * 16 bytes ≈ 2,097,152 bytes (~2 MB)`

- Hash tree with height `20`:
  - `Total hash tree nodes ≈ 2^(20+1) = 2,097,152`
  - `Memory required ≈ 2,097,152 * 16 bytes ≈ 33,554,432 bytes (~33 MB)`

##### Performance Consideration: Number of Leaves

The objects in a shard (e.g. tenant) are distributed among the leaves of the hash tree.
A larger hash tree generally distributes objects across more leaves, allowing finer comparisons. End-to-end repair cost still depends on data distribution, divergence, I/O, and worker resources.

- **Number of Leaves in the hash tree:**
  ```
  Number of leaves = 2^H
  ```

##### Examples

- Hash tree with height `16`:
  - `Number of Leaves = 2^16 = 65,536`

- Hash tree with height `20`:
  - `Number of Leaves = 2^20 = 1,048,576`

:::note Default settings
The default hash tree height is `16` for single-tenant collections and `10` for multi-tenant collections. These defaults balance memory consumption with replication performance.

As of `v1.36`, these parameters can be configured per-collection via the [`asyncConfig`](/weaviate/config-refs/collections#async-config) object in `replicationConfig`. Worker concurrency is no longer a per-collection setting and as of `v1.38` each node shares an async replication worker pool across its local collections and shards, sized by [`ASYNC_REPLICATION_SCHEDULER_WORKERS`](/deploy/configuration/env-vars/index.md#async-replication).
:::

### Deletion resolution strategies

When an object is present on some replicas but not others, this can be because a creation has not yet been propagated to all replicas, or because a deletion has not yet been propagated to all replicas. It is important to distinguish between these two cases.

Deletion resolution works alongside async replication and repair-on-read to ensure consistent handling of deleted objects across the cluster. For each collection, [you can set one of the following](../../manage-collections/multi-node-setup.mdx#replication-settings) deletion resolution strategies:

- `NoAutomatedResolution`
- `DeleteOnConflict`
- `TimeBasedResolution`

Deletion resolution strategies are mutable. [Read more about how to update collection definitions](../../manage-collections/collection-operations.mdx#update-a-collection-definition).

#### `NoAutomatedResolution`

In this mode, Weaviate does not treat deletion conflicts as a special case. If an object is present on some replicas but not others, Weaviate may potentially restore the object on the replicas where it is missing.

#### `DeleteOnConflict`

A deletion conflict in `deleteOnConflict` is always resolved by deleting the object on all replicas.

To do so, Weaviate updates an object as a deleted object on a replica upon receiving a deletion request, rather than removing all traces of the object.

#### `TimeBasedResolution`

This is the default setting from `v1.36` onwards. A deletion conflict in `timeBasedResolution` is resolved based on the timestamp of the deletion request, in comparison to any subsequent updates to the object such as a creation or an update.

If the deletion request has a timestamp that is later than the timestamp of any subsequent updates, the object is deleted on all replicas. If the deletion request has a timestamp that is earlier than the timestamp of any subsequent updates, the later updates are applied to all replicas.

For example:
- If an object is deleted at timestamp 100 and then recreated at timestamp 110, the recreation wins
- If an object is deleted at timestamp 100 and then recreated at timestamp 90, the deletion wins

#### Choosing a strategy

- Use `NoAutomatedResolution` when you want maximum control and handle conflicts manually
- Use `DeleteOnConflict` when you want to ensure deletions are always honored
- Use `TimeBasedResolution` when you want the most recent operation to take precedence

### Repair-on-read

If your read consistency is set to `All` or `Quorum`, the read coordinator will receive responses from multiple replicas. If these responses differ, the coordinator can attempt to repair the inconsistency, as shown in the examples below. This process is called "repair-on-read", or "read repairs".

| Problem | Action |
| :- | :- |
| Object never existed on some replicas. | Propagate the object to the missing replicas. |
| Object is out of date. | Update the object on stale replicas. |
| Object was deleted on some replicas. | Policy-specific: unresolved tombstone conflicts can error with `NoAutomatedResolution`; `DeleteOnConflict` propagates deletion; `TimeBasedResolution` compares operation timestamps. Repair can fail if communication or concurrent-change checks fail. |

The read repair process also depends on the read and write consistency levels used.

| Write consistency level | Read consistency level | Action |
| :- | :- | :- |
| `ONE` | `ALL` | Applicable read paths attempt checks/repair of consulted routed replicas; this does not guarantee repair of every configured copy. |
| `QUORUM` | `QUORUM` or `ALL` | Weaviate attempts to fix the sync issues. |
| `ALL` | - | Later divergence or partial failed operations can still require repair; a prior successful routed ALL is not a permanent repair exemption. |

Read repairs are triggered by applicable reads; async replication also performs background repair and consumes resources. While nodes are in an inconsistent state, read operations with consistency level of `ONE` may return stale data.

## Replica movement

import ReplicaMovement from '/_includes/feature-notes/replica-movement.mdx';

<ReplicaMovement/>

A shard represents a part of the collection in a single-tenant collection, or a whole tenant in a multi-tenant collection. Weaviate allows users to manually move or copy individual shard replicas from a source node to a destination node in a Weaviate cluster. This capability addresses operational scenarios such as cluster rebalancing after scaling, node decommissioning, optimizing data locality for improved performance, or increasing data availability.

Replica movement operates as a state machine with stages that ensure data integrity throughout the process. The feature works for both single-tenant collections and multi-tenant collections. 

Unlike the static replication factor configured at collection creation, replica movement allows the replication factor to be adjusted for specific shards as replicas are moved or copied across the cluster. When a copy operation is performed, the newly created replica increases the replication factor for that specific shard. While a collection may have a default replication factor, individual shards within that collection can have a higher replication factor. However, shards can't have a replication factor lower then the one set on the collection level. 

:::info

Replica movement requires enablement. The inspected v1.40.5 parser uses [`REPLICA_MOVEMENT_ENABLED`](/deploy/configuration/env-vars/index.md#REPLICA_MOVEMENT_ENABLED); see [replica movement configuration](/deploy/configuration/replica-movement.mdx) for deployment setup. No absent-environment default is implied here.

The [`REPLICATION_ENGINE_MAX_WORKERS` environment variable](/docs/deploy/configuration/env-vars/index.md#REPLICATION_ENGINE_MAX_WORKERS) can be used to adjust the number of workers that process replica movements in parallel. 

:::

### Movement states

Each replica movement operation progresses through a workflow designed to maintain data consistency and availability. The workflow comprises of the following states:

- **REGISTERED**: The movement operation has been initiated and logged by the Raft leader. The request has been received and the operation is queued for processing.

- **HYDRATING**: A new replica is being created on the destination node. Data segments are transferred from an existing replica (usually the source replica, or another available peer) to establish the new replica.

- **FINALIZING**: The bulk data transfer is complete, and the new replica is catching up on any writes that occurred during the transfer. This ensures the replica is fully synchronized with the latest data. You can use the [`REPLICA_MOVEMENT_MINIMUM_ASYNC_WAIT` environment variable](/docs/deploy/configuration/env-vars/index.md#REPLICA_MOVEMENT_MINIMUM_ASYNC_WAIT) to adjust the wait time which ensures that any in progress writes have been completed and replicated to the target node.

- **INTEGRATING** (added in `v1.38.0`): The new replica has joined the shard's replica set and is being brought into the write path on every node. The operation waits until all nodes agree that the new replica is a write target, so that no node can acknowledge a write that skips it. Each node reports that it reached this state only after its own in-flight writes to the shard have drained, which prevents a write that was already accepted from being lost. Once the cluster agrees, the last writes recorded on the source during the transition are applied to the new replica, and the source stops recording further changes for this operation. For copy operations, the next state is **READY**. For move operations, the next state is **DEHYDRATING**.

- **DEHYDRATING**: For move operations, after the new replica is ready, the original replica on the source node is being removed. 

- **READY**: The operation has completed successfully. The new replica is fully synchronized and ready to serve traffic. For move operations, the source replica has been removed.

- **CANCELLED**: The operation has been cancelled before completion. This can happen either through manual intervention or if the operation encounters an unrecoverable error.

Replica movement supports two distinct operation modes:

- **Move**: Move a replica from one node to another, maintaining the same replication factor
- **Copy**: Copy a replica from one node to another and increase the shard replication factor by one for that specific shard

:::note Replication factor and quorum

When a shard replica is copied, the increased replication factor may become an even number. This can make achieving a quorum more difficult, as it now requires `(n/2 + 1)` nodes instead of `(n/2 + 0.5)` nodes. For example, going from `RF=3` to `RF=4` increases the required nodes for quorum from 2 to 3 (67% to 75% of replicas).

:::

## Shard self-recovery

import EnterpriseEdition from '/_includes/feature-notes/enterprise-edition.mdx';

:::info Added in `v1.40`
:::

<EnterpriseEdition/>

Shard self-recovery restores a shard replica whose data directory is missing on a node, for example after a disk replacement or a lost volume. Instead of starting the shard empty, the node copies the shard's files from a healthy replica on another node. A recovery is a replication operation, so it uses the same engine as [replica movement](#replica-movement).

This section describes how a recovery works. To enable, monitor, and control self-recovery, see [Configuration: Shard Self-Recovery](/deploy/configuration/self-recovery).

### When recovery starts

A node checks for missing shard directories at two points:

- **At startup.** While the node loads its shards, each `HOT` shard whose directory is missing is recovered.
- **When a tenant is activated.** For a multi-tenant collection, a `COLD` tenant whose shard directory is missing on this node is recovered when the tenant is activated. This only happens when the collection's schema has more than one replica.

A node that starts with no Raft state at all, for example with an empty data volume (a *wiped* node), first catches up with the cluster's schema and only then loads its shards. If the catch-up stops making progress for `SELF_RECOVERY_BARRIER_TIMEOUT` (default `3m`), the node logs a warning and loads its shards anyway. In that case some shards can start empty.

### Choosing a source replica

The node probes all other replicas of the shard in parallel, in random order. The first replica that reports having data for the shard becomes the source for the copy.

The node creates an empty shard, without copying any data, only in these cases:

- The shard has no other replica.
- No replica was unreachable, and at least one replica reported that it has no data for the shard.

If any replica is unreachable, the node doesn't fall back to an empty shard. It retries, because the unreachable replica might hold the data. A replica that is itself recovering the same shard counts as having no data.

### Copying the data

The node copies the source replica's files into a staging directory named `<shard>.recovering/` next to the shard's normal location. The copy is resumable per file: a file whose CRC32 checksum already matches the source is skipped. When the copy completes, the node renames the staging directory to the shard directory in a single atomic step, and loads the shard.

### Request routing during recovery

While a registered recovery operation for the shard is in progress, the cluster excludes the recovering replica from read and write routing, so other replicas serve the requests. Searches and reads, such as fetching an object by ID, succeed through the healthy replicas. Operations that must consult every replica, such as aggregations, can be delayed until the recovery finishes, and then succeed. Internally, the recovering replica rejects requests from other nodes, and the coordinating node handles these rejections.

### Retries and giving up

A recovery makes up to 10 attempts. Probe errors, unreachable replicas, and failures to register or track the copy operation all count as attempts. Between attempts, the node waits 5 seconds, doubling the wait each time up to a limit of 5 minutes. All attempts together take about 20 minutes.

If all attempts fail, the node gives up. The shard stays in the `RECOVERING` state: it doesn't become empty, and it doesn't serve data. On the next startup, the node submits the recovery again. An operator can also [restart the recovery or accept an empty shard](/deploy/configuration/self-recovery#operator-controls).

## Related pages
- [Configuration: Shard Self-Recovery](/deploy/configuration/self-recovery)
- [API References | GraphQL | Get | Consistency Levels](../../api/graphql/get.md#consistency-levels)
- <SkipLink href="/weaviate/api/rest#tag/objects">API References | REST | Objects</SkipLink>

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
