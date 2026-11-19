---
title: Storage
sidebar_position: 18
description: "Persistent, fault-tolerant storage architecture for objects, vectors, and inverted index management, including HNSW snapshots and commit log compaction."
image: og/docs/concepts.jpg
# tags: ['architecture', 'storage']
---

Weaviate is a persistent and fault-tolerant database. This page gives you an overview of how objects and vectors are stored within Weaviate and how an inverted index is created at import time.

The components mentioned on this page aid Weaviate in creating some of its unique features:

* Write-ahead logs support recovery, but acknowledgment, buffer flushing, and synchronization to durable media are different stages. Process crashes and OS, power, or storage failures have different durability implications.
* On a vector search query, Weaviate returns the entire object (in other databases sometimes called a "document"), not just a reference, such as an ID.
* When combining structured search with vector search, filters are applied prior to performing the vector search. This avoids discarding an already-limited result list through post-filtering, but returns **up to** the requested limit from eligible indexed objects, not a guaranteed count.
* Objects and their vectors can be updated or deleted at will, even while reading from the database.

## Logical Storage Units: Indexes, Shards, Stores

Each class in Weaviate's user-defined schema leads to the creation of an index internally. An index is a wrapper type that is comprised of one or many shards. Shards within an index are self-contained storage units. Multiple shards can be used to distribute the load among multiple server nodes automatically.

### Components of a Shard

A shard groups object storage with its configured indexes:

* An object store, essentially a key-value store
* Applicable, enabled [inverted indexes](https://en.wikipedia.org/wiki/Inverted_index)
* Zero or more configured [vector indexes](./indexing/vector-index.md); named vector targets have their own indexes

#### Object and Inverted Index Store

Since version `v1.5.0`, the object and inverted store are implemented using an [LSM-Tree approach](https://en.wikipedia.org/wiki/Log-structured_merge-tree). Writes are buffered in memory and recorded in WALs; after a configured threshold, a sorted memtable is written to a disk segment. Actual ingestion throughput depends on workload, I/O, and available resources. When a read request comes in, Weaviate will first check the Memtable for the latest update for a specific object. If it is not present in the memtable, Weaviate will then check all previously written segments starting with the newest. To avoid checking segments which don't contain the desired objects, [Bloom filters](https://en.wikipedia.org/wiki/Bloom_filter) are used.

Weaviate periodically merges smaller, older segments to make larger segments. Since the segments are already sorted, this is a relatively cheap operation. It happens constantly in the background. Fewer, larger segments make lookups more efficient. In the inverted index data is rarely replaced, but it is often appended. Merging means that, instead of checking all past segments and aggregating potential results, Weaviate can check a single segment (or a few large segments) and immediately find all the relevant object pointers. In addition, segments are used to remove earlier versions of an object that are out-dated because of a delete or a more recent update.

Considerations

Object and inverted-index storage use segmented LSM trees. HNSW persists graph topology through commit logs and snapshots rather than independent searchable graph segments. Other vector layouts differ: flat vectors and HFresh posting lists use LSM-backed buckets, and dynamic starts flat before transitioning to HNSW. HNSW auxiliary data can also use shard buckets; graph persistence is not all vector-related storage.

Weaviate versions before `v1.5.0` use a B+Tree storage mechanism. The LSM design buffers writes and batches disk output for efficient ingestion; it does not guarantee constant write latency or throughput.

To learn more about Weaviate's LSM store, see the LSM library documentation in the [Go package repository](https://pkg.go.dev/github.com/weaviate/weaviate/adapters/repos/db/lsmkv)

#### HNSW Vector Index Storage

For each configured HNSW target, the shard maintains a graph index alongside object and property storage. The graph is not split into independently searched LSM segments; this description is specific to HNSW, not every supported vector-index type.

By grouping a vector index with the object storage within a shard, Weaviate can make sure that each shard is a fully self-contained unit which can independently serve requests for the data it owns. By placing the vector index next to the object store (instead of within), Weaviate can avoid the downsides of a segmented vector index.

Furthermore, its persistence and loading at startup are optimized through a combination of Write-Ahead-Logging and HNSW snapshots, detailed in the [Persistence and Crash Recovery](#persistence-and-crash-recovery) section.

### Shard Components Optimizations

Weaviate's storage mechanisms use segmentation for structured/object data. Segments are cheap to merge and even unmerged segments can be navigated efficiently thanks to Bloom filters. These mechanisms are designed for efficient ingestion; compaction, dataset changes, workload, and resource pressure still affect performance.

For HNSW, Weaviate maintains a graph per configured target within a shard. HNSW indexes cannot be merged efficiently. Querying a single large index is more efficient than sequentially querying many small indexes.

Multiple shards can expose import parallelism, including on one node, but add overhead. Benchmark shard counts for your indexes and workload rather than assuming more shards guarantee the fastest import.

### Lazy shard loading

When a node starts, it initializes its eligible local active shards. Inactive/offloaded tenants are not activated by shard loading, and empty or otherwise deferred shards can load on demand. This process can take a long time. Since every tenant is a shard, multi-tenant deployments with many tenants can have reduced availability after a restart.

Lazy shard loading allows you to start working with your data sooner. After a restart, eligible shards can load in the background, subject to warmup settings; deferred shards load on access. If the shard you want to query is already loaded, you can get your results sooner. If the shard is not loaded yet, Weaviate prioritizes loading that shard and returns a response when it is ready.

#### Dynamic lazy shard loading

:::info Added in `v1.36.6`
:::

Starting in v1.36.6, Weaviate automatically decides **per collection on each node** whether to use lazy shard loading, unless an explicit loading override applies. Auto-detection only applies to **multi-tenant** collections and is based on two local thresholds:

- **Shard count threshold** ([`LAZY_LOAD_SHARD_COUNT_THRESHOLD`](/docs/deploy/configuration/env-vars/index.md#LAZY_LOAD_SHARD_COUNT_THRESHOLD)): Number of local active tenant shards in the collection. Default: `1000`.
- **Shard size threshold** ([`LAZY_LOAD_SHARD_SIZE_THRESHOLD_GB`](/docs/deploy/configuration/env-vars/index.md#LAZY_LOAD_SHARD_SIZE_THRESHOLD_GB)): Total local shard disk size for the collection. Default: `100` GiB (the setting converts using `1024^3` bytes).

Absent an explicit override, exceeding either local threshold selects lazy loading on that node; otherwise eligible shards load eagerly before readiness. Empty/deferred shards may still load on demand. Explicit settings can override auto-detection: `DISABLE_LAZY_LOAD_SHARDS=true` forces eager loading, while a count threshold of `0` forces lazy loading for all collections, including single-tenant collections. See the [loading settings](/deploy/configuration/env-vars/index.md) for deployment configuration.

This change improves reliability during rolling restarts and upgrades. Eager loading eliminates the increased query and ingestion latency that lazy loading can introduce for smaller deployments during rollouts.

#### Vector cache prefill behavior

The [`HNSW_STARTUP_WAIT_FOR_VECTOR_CACHE`](/deploy/configuration/env-vars#HNSW_STARTUP_WAIT_FOR_VECTOR_CACHE) environment variable controls whether vector cache prefill is synchronous (blocking) or asynchronous (background) at startup. Its default changed to `true` in v1.36.6.

For collections where lazy shard loading is active, vector cache prefill is always **asynchronous**: the `HNSW_STARTUP_WAIT_FOR_VECTOR_CACHE` value is overridden to `false` regardless of the configured value. For eagerly-loaded collections, the configured value applies (default: `true`, meaning synchronous prefill).

:::note Behavior change from v1.36.6
Prior to v1.36.6, lazy shard loading was enabled by default for all collections. From v1.36.6 onward, shards are **eagerly loaded by default** until a multi-tenant collection crosses the count or size threshold. This may increase startup time for smaller deployments but provides better reliability during rollouts.
:::

## Persistence and Crash Recovery

### Write-Ahead-Log

Both the LSM stores used for object and inverted storage, as well as the HNSW vector index store make use of memory at some point of the ingestion journey. To prevent data loss on a crash, each operation is additionally written into a **[Write-Ahead-Log (WAL)](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html)** (also known as a *commit log*). WALs are append-only files that are very efficient to write to and that are rarely a bottleneck for ingestion.

By the time Weaviate has responded with a successful status to your ingestion request, an LSM store WAL entry will have been created. If a WAL entry could not be created - for example because the disks are full - Weaviate will respond with an error to the insert or update request. The HNSW graph has its own commit log, described [below](#hnsw-snapshots). With synchronous vector indexing, graph updates occur on the originating request path. With [asynchronous indexing](./indexing/vector-index.md#asynchronous-indexing), vector work enters a persistent queue and graph/log updates occur later. Accepted object writes and completed vector-search visibility are distinct.

Flushing a WAL buffer to the operating system is not the same as `fsync` to durable media. A successful ingestion response does not mean every WAL was individually synchronized for that write. WAL recovery supports process-crash recovery, but does not guarantee survival of every acknowledged write through OS, power, or storage failures.

The LSM stores try to flush segments on orderly shutdown. After an unexpected stop, recovery replays surviving WAL records. A suitable final WAL that can be faithfully replayed may be adopted as the active memtable/log and continue receiving writes. Otherwise, recovered data is flushed into one or more segments and the covered WAL is removed after the recovered output is committed. Reusing an active WAL means a later restart can legitimately replay it again.

For the HNSW vector index, the Write-Ahead-Log (WAL) is a critical component for disaster recovery and persisting the most recent changes. The cost in building up an HNSW index is in figuring out where to place a new object and how to link it with its neighbors. The WAL contains only the result of those calculations.

The entire HNSW index state can be reconstructed by replaying these WAL entries.

For very large indexes of tens or hundreds of millions of objects, this can be time-consuming. To avoid replaying the entire commit log on every restart, Weaviate writes **[HNSW snapshots](#hnsw-snapshots)**.

### HNSW snapshots

import HnswSnapshots from '/_includes/feature-notes/hnsw-snapshots.mdx';

<HnswSnapshots/>

A snapshot represents a point-in-time state of an HNSW graph; it is not an independent backup of the collection. When Weaviate starts, it loads the most recent snapshot and replays only the commit log entries written after it. This significantly reduces startup time, because the number of entries that have to be replayed no longer grows with the age of the index.

The commit log records every change to the index as it happens. Entries are written to the log as batches are processed, and a log file is synced to disk when it is rotated. Even with a fresh snapshot, Weaviate typically still has to load at least one subsequent commit log file.

Starting in `v1.39`, snapshots are part of how the vector index is stored rather than an optional speedup. A background process called the commit log compactor owns the on-disk lifecycle of the index: it compacts newly flushed commit logs, merges them together, and writes a new snapshot when doing so is worthwhile. Snapshots and commit logs live in the same directory, and a snapshot replaces the commit logs it covers rather than duplicating them, so the commit logs left on disk hold only the delta since the last snapshot. This keeps the disk footprint proportional to the size of the index. Snapshots are also written as a stream. Weaviate still loads the snapshot it supersedes into memory, but the commit log delta and the new snapshot itself are streamed rather than also held there, as they were before `v1.39`.

Upgrading to `v1.39` reduces the disk space the vector index uses, in some cases substantially. Earlier versions keep the full commit log alongside the snapshot, and a snapshot is a more compact representation of the same index than the commit logs it replaces, because compaction keeps only the final state of each vector's connections instead of every change made to them.

A few caveats apply. The saving appears once the compactor has run its first cycles on each loaded shard rather than at the moment you upgrade, and inactive tenants do not shrink until they are next activated. Plan headroom for the peak rather than the steady state: while a snapshot is being written, the directory transiently holds the previous snapshot, the files being merged, and the new snapshot as it is assembled, so disk usage during snapshot creation is meaningfully above the size the index settles at.

Weaviate protects this on-disk state in several ways. Snapshots and compacted commit logs are written to a temporary path and atomically renamed into place, so an interrupted write can never be mistaken for a complete file, and orphaned temporary files are cleaned up on the next startup. When a new snapshot is written, the snapshot it supersedes and the commit logs it covers are removed only after the new one is durably on disk.

Commit logs are self-healing. If a crash leaves the last entry of a log incomplete, the file is truncated back to its last valid entry. The entries written before the tear are retained and the file becomes valid again for later compaction, so only the incomplete tail is lost.

Snapshots are handled differently. A snapshot is stored in a checksummed block format and every block is verified when it is read, but unlike a commit log, a snapshot is not truncated or repaired.

In the rare case that the current snapshot cannot be read, restore the affected data from a [backup](/deploy/configuration/backups.md), which includes the snapshot. Weaviate does not load a partial index, and because the commit logs the snapshot covers have already been removed, nothing remains on the node to replay in its place.

That failure is scoped to the shard that owns the snapshot: the shard fails to load, and so does every other vector index on it. If that shard uses [dynamic lazy shard loading](#dynamic-lazy-shard-loading), the node stays up and requests to the shard return an error. If the shard is loaded eagerly, which is the default for single-tenant collections and for multi-tenant collections below the auto-detection thresholds, node startup fails instead.

Weaviate creates and maintains snapshots automatically, so there is nothing to enable, disable, schedule, or tune. [`PERSISTENCE_HNSW_MAX_LOG_SIZE`](/deploy/configuration/env-vars/index.md#PERSISTENCE_HNSW_MAX_LOG_SIZE) still influences the size at which commit log files are rotated, and therefore how often there is new material to compact, but it does not configure snapshots.

The environment variables that configured snapshots before `v1.39` are deprecated. That version and later still recognize `PERSISTENCE_HNSW_DISABLE_SNAPSHOTS` and the `PERSISTENCE_HNSW_SNAPSHOT_*` variables, so an existing deployment starts without a configuration error, but their values are ignored. For each of these variables that is set, Weaviate logs a warning at startup stating that the variable has no effect and will be removed in a future version. If these options are set through a configuration file rather than as environment variables, they are ignored in the same way, but no startup warning is logged. Remove the variables from your deployment configuration to clear the warnings.

#### Snapshot configuration before `v1.39` {#pre-v1-39-configuration}

In `v1.31` through `v1.38`, snapshots are an optional feature layered on top of the commit log rather than part of it, and the `PERSISTENCE_HNSW_SNAPSHOT_*` environment variables control when Weaviate creates them. Historical defaults vary by version; consult the reference for the release you operate. Weaviate can create one at startup and periodically thereafter, once enough new commit log data has accumulated since the last snapshot. Only commit log files that have been rotated count toward that threshold, so changes still in the active file are not considered until the next rotation. If a snapshot cannot be read in these versions, it is discarded and Weaviate replays the full commit log instead. For the variables themselves, including their defaults and deprecation status, see [`PERSISTENCE_HNSW_DISABLE_SNAPSHOTS`](/deploy/configuration/env-vars/index.md#PERSISTENCE_HNSW_DISABLE_SNAPSHOTS) and the rows that follow it.

<details>
  <summary>Periodic snapshot conditions and memory requirements</summary>

Periodic snapshot creation is governed by three variables, and **all** of the following conditions must be met before Weaviate creates a snapshot:

- `PERSISTENCE_HNSW_SNAPSHOT_INTERVAL_SECONDS` — the minimum time since the previous snapshot has elapsed (default `21600` seconds, or six hours).
- `PERSISTENCE_HNSW_SNAPSHOT_MIN_DELTA_COMMITLOGS_NUMBER` — enough new commit log files have been created since the last snapshot (default `1`).
- `PERSISTENCE_HNSW_SNAPSHOT_MIN_DELTA_COMMITLOGS_SIZE_PERCENTAGE` — the new commit logs are large enough, measured as a percentage of the previous snapshot's size (default `5`). This condition does not apply to the first snapshot, when there is no previous snapshot to measure against.

Meeting these conditions makes a snapshot eligible rather than guaranteed. The background process that condenses and combines commit log files is also the one that writes the snapshot, so a snapshot can be created on a later pass than the one where the conditions are first met.

In these versions, before creating a new snapshot, Weaviate loads the previous snapshot and the commit log difference into memory, so the node needs enough memory to accommodate both.

</details>

## Conclusions

Object/property storage, vector indexing, and their recovery logs have separate responsibilities. LSM buffering and compaction support efficient writes without guaranteeing constant latency. HNSW graph commit logs and snapshots support graph recovery, while other index types use different storage layouts. Acknowledgment, durable-media synchronization, and vector-search visibility must not be treated as the same event.

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
