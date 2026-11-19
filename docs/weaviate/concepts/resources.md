---
title: Resource Planning
sidebar_position: 90
description: "CPU, memory, and GPU resource planning guidelines for optimal Weaviate performance at scale."
image: og/docs/concepts.jpg
# tags: ['architecture', 'resource', 'cpu', 'memory', 'gpu']
---

Plan resources from vector count and dimensions, index type and compression, object and property indexes, workload, and replication—not object count alone. Even small collections can require substantial resources with high-dimensional or multiple vectors. Include CPU, memory, and storage capacity and I/O in your planning. Depending on the modules you use, GPUs may also play a role.


## Limit available resources

You can set [environment variables](/deploy/configuration/env-vars/index.md) to manage Weaviate's resource usage, as to prevent Weaviate from using all available resources. The following environment variables are available:

- [`LIMIT_RESOURCES`](/deploy/configuration/env-vars/index.md#LIMIT_RESOURCES): When set to true, Weaviate automatically limits its resource usage. It sets memory usage to 80% of the total memory and uses all but one CPU core. It overrides any `GOMEMLIMIT` values but respects `GOMAXPROCS` settings.

- [`GOMEMLIMIT`](/deploy/configuration/env-vars/index.md#GOMEMLIMIT): This sets the memory limit for the Go runtime, which should be around 80-90% of the total memory available for Weaviate. It controls the aggressiveness of the Garbage Collector as memory usage approaches this limit.

- [`GOMAXPROCS`](/deploy/configuration/env-vars/index.md#GOMAXPROCS): This sets the maximum number of threads for concurrent execution. If set, it's respected by `LIMIT_RESOURCES`, allowing users to specify the exact number of CPU cores Weaviate should use.

These settings help in optimizing Weaviate's performance by balancing resource utilization with the available system resources.


## The role of CPUs

:::tip Rule of thumb
The CPU has a direct effect on query and import speed, but does not affect dataset size.
:::

HNSW vector search and insertion can be CPU-intensive when the required vectors and graph are resident in memory. Complete requests can also be limited by vector-cache misses, object retrieval, storage I/O, or indexing backlog. Other index types, such as HFresh, have different memory and I/O trade-offs. You can [tune the HNSW index](../config-refs/indexing/vector-index.mdx) on a per collection basis for your primary use case.

To use multiple CPUs efficiently, create multiple shards for your collection. For the fastest imports, create multiple shards even on a single node.

HNSW graph traversal is largely single-threaded, but a complete request can include parallel work, such as candidate rescoring. Concurrent searches and inserts can also use multiple threads. [Batch inserts](/weaviate/manage-objects/import) use multiple threads to process data in parallel.

### When to add more CPUs

Consider adding CPUs when measurements show that CPU is the bottleneck during import or search. Check cache behavior, storage I/O, and indexing queues before scaling: adding CPUs does not resolve every throughput limit.

## The role of memory

:::tip Rule of thumb
Memory affects resident-index capacity, cache hit rates, and garbage collection. These can affect both latency and throughput.
:::

The HNSW graph is stored in memory; vector residency depends on compression and cache settings. Dataset size and query concurrency both matter for memory planning. You can use [`product quantization (PQ)`](/weaviate/concepts/vector-quantization#product-quantization) to reduce vector memory, but compression does not shrink the graph. If memory is your main constraint, the disk-based [HFresh index](/weaviate/concepts/vector-index#hfresh-index) is another option.

[`vectorCacheMaxObjects`](../config-refs/indexing/vector-index.mdx) configures an entry-count threshold for each applicable vector-index cache instance. It is not a byte limit or a guarantee against Out-of-Memory ("OOM") situations. Account for separate shard and named-vector index instances and their replicas, along with graph memory, other indexes, and runtime allocations.

Weaviate also uses [memory-mapped files](https://en.wikipedia.org/wiki/Memory-mapped_file) for data stored on disks. Memory-mapped files are efficient, but disk storage is much slower than in-memory storage.

### Which factors drive memory usage?

For HNSW deployments, include these components when planning memory:

- **Vectors and their representations**. Count dimensions, data type, compression, and the number of vectors per object. Cache settings determine which applicable representations remain resident.
- **The HNSW graph**. [`maxConnections`](../config-refs/indexing/vector-index.mdx) limits connections per upper layer; the base layer allows `2 * maxConnections`. Resident graph memory also includes nodes, layer metadata, and allocation overhead, so a fixed bytes-per-connection multiplier is not a complete estimate.
- **Objects and property indexes**. Objects need not all reside in the Go heap, but their size and the indexes used for filtering and keyword search affect storage and the memory working set.
- **Caches, runtime, and temporary work**. Include operating-system file cache, query concurrency, indexing buffers and queues, and import or recovery peaks.

### An example calculation

:::note
This example covers one logical copy of one uncompressed HNSW vector index, with one vector per object and all vectors cached. It is not total process or cluster memory sizing. For cache trade-offs, see [Vector Cache](#vector-cache) below.
:::

A rough starting heuristic for an uncompressed HNSW index is:

`Index planning heuristic = 2 * (the raw float32 vector bytes)`

For one million 384-dimensional vectors of type `float32`:

- A single vector contains `384 * 4 B = 1536 B` of raw values.
- The raw arrays contain `1e6 * 1536 B = 1.536 GB` (about `1.431 GiB`).
- Doubling gives `3.072 GB` (about `2.861 GiB`) as a starting heuristic, not a measured resident-memory requirement or peak bound.

Here, GB means decimal gigabytes and GiB means binary gibibytes. The heuristic does not separately size graph topology and allocations, object and property indexes, file cache, runtime overhead, or transient work. It does not apply to compressed HNSW, flat, or HFresh indexes. Measure these components and peak headroom for your workload before selecting capacity.

For a deployment estimate, account for each shard and vector index and the replicas placed on each node. Do not treat the memory summed across all copies as a per-node requirement.

## Effects of garbage collection

Weaviate is written in Go, which is a garbage-collected language. This means some memory is not immediately available for reuse when it is no longer needed. The application has to wait for an asynchronous process, the garbage collector, to free up the memory. This has two distinct effects on memory use:

- [Memory overhead](#memory-overhead-for-the-garbage-collector)
- [Out-of-memory issues](#out-of-memory-issues-due-to-garbage-collection)

### Memory overhead for the garbage collector
During imports and queries, Weaviate allocates temporary memory that is eventually freed by the garbage collector. Since garbage collection is asynchronous, leave headroom above the measured resident working set. The amount depends on allocation rate, concurrency, and runtime settings; the 2× index heuristic above does not guarantee sufficient garbage-collection or peak headroom.

### Out-of-Memory issues due to garbage collection
In rare situations - typically on large machines with very high import speeds - Weaviate can allocate memory faster than the garbage collector can free it. When this happens, the system kernel can trigger an `out of memory kill (OOM-Kill)`. This is a known issue that Weaviate is actively working on.

### Data import
To manage Go runtime memory pressure during imports, set `LIMIT_RESOURCES` to `True` or configure `GOMEMLIMIT`. These settings do not replace workload sizing or guarantee that the process cannot run out of memory. For details, see [Environment variables](/deploy/configuration/env-vars/index.md).

## Strategies to reduce memory usage

The following tactics can help to reduce Weaviate's memory usage:

- **Use vector compression**. Product quantization (PQ) is a technique that reduces the size of vectors. Vector compression impacts recall performance, so we recommend testing PQ on your dataset before using it in production. <br/><br/> For more information, see [Product Quantization](/weaviate/concepts/vector-quantization). <br/> To configure PQ, see [Compression](../configuration/compression/pq-compression.md).

- **Reduce the dimensionality of your vectors.** The most effective approach to reducing memory size, is to reduce the number of dimensions per vector. If you have high dimension vectors, consider using a model that uses fewer dimensions. For example, a model that has 384 dimensions uses far less memory than a model with 1536 dimensions.

- **Reduce [`maxConnections`](../config-refs/indexing/vector-index.mdx) in your HNSW index settings**. This reduces the connection budget per layer, including the base-layer budget of `2 * maxConnections`, and can reduce graph memory. The actual saving depends on graph occupancy and representation.

Reducing `maxConnections` adversely affects HNSW recall performance. To mitigate this effect, increase one or both of the `efConstruction` and `ef` parameters.

- Increasing `efConstruction` increases import time without affecting query times.
- Increasing `ef` increases query times without affecting import times.

- **Use a vector cache that is smaller than the total amount of your vectors (not recommended)**. This strategy is described under [Vector Cache](#vector-cache) below. It has a significant performance impact, and is only recommended in specific, limited situations.

## Vector Cache

For HNSW workloads, keeping the searched vector representations in memory avoids storage lookups. The [`vectorCacheMaxObjects`](../config-refs/indexing/vector-index.mdx) parameter in the collection definition sets an entry-count threshold for each applicable index cache, defaulting to one trillion (`1e12`). Separate shards and named-vector indexes have separate instances, and replicas add copies. This is not a collection-wide or process-wide memory cap, and cache maintenance is not an instantaneous allocation limit.

Reducing `vectorCacheMaxObjects` can introduce storage lookups and cache churn. The performance impact depends on the working set, file-cache state, and storage. Test changes under representative import and query loads.

Generally we recommend that:
- During import set `vectorCacheMaxObjects` high enough that all vectors can be held in memory. Each import requires multiple searches. Import performance drop drastically when there isn't enough memory to hold all of the vectors in the cache.

- After import, when your workload is mostly querying, experiment with vector cache limits that are less than your total dataset size.

  Vectors that aren't currently in cache are added to the cache if there is still room. If the cache fills, Weaviate drops the whole cache. All future vectors have to be read from disk for the first time. Then, subsequent queries runs against the cache, until it fills again and the procedure repeats. Note that the cache can be a very valuable tool if you have a large dataset, and a large percentage of users only query a specific subset of vectors. In this case you might be able to serve the largest user group from cache while requiring disk lookups for "irregular" queries.

### When to add more Memory to your Weaviate machine or cluster

Consider adding more memory if:
- You want to import a larger dataset (more common).
- Exact lookups are disk-bound and more memory will improve page-caching (less common).

## The role of GPUs in Weaviate

Weaviate Database itself does not make use of GPUs. However, some of the models that Weaviate includes as modules are meant to run with GPUs, for example `text2vec-transformers`, `qna-transformers`, and `ner-transformers`. These modules run in isolated containers, so you can run the module containers on GPU-accelerated hardware while running Weaviate Database on low-cost CPU-only hardware.

## Disks: SSD vs Spinning Disk

Weaviate is optimized to work with Solid-State Disks (SSDs). However, spinning hard-disks can also be used with some performance penalties.

## File system

For optimal performance and reliability, avoid using `NFS` or similar file systems for the Weaviate persistent volume ([`PERSISTENCE_DATA_PATH`](/deploy/configuration/env-vars/index.md)). 

Instead, use file systems like `Ext4` or `XFS` in combination with SAN storage (e.g. `EBS`) to ensure the best performance.

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
