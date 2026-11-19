---
title: ANN Benchmark
sidebar_position: 1
description: "Approximate Nearest Neighbor performance benchmarks measuring Weaviate's search accuracy and speed in real-world scenarios."
image: og/docs/benchmarks.jpg
# tags: ['Weaviate', 'performance', 'benchmarks', 'ANN benchmarks', 'vector database benchmarks']
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

import useBaseUrl from '@docusaurus/useBaseUrl';
import ThemedImage from '@theme/ThemedImage';

import BenchmarkGrid from '@site/src/components/Documentation/BenchmarkGrid';

# ANN Benchmark

This vector database benchmark is designed to measure and illustrate Weaviate's Approximate Nearest Neighbor (ANN) performance for a range of real-life use cases.

To make the most of this vector database benchmark, you can look at it from different perspectives:

- **Reported performance** – Review the [benchmark results](#benchmark-results) as observations under the documented test conditions, subject to the provenance limitation below.
- **Starting point for your use case** – Find the dataset closest to your production use case, then measure your own workload rather than treating these results as production capacity estimates.
- **Fine Tuning** – If you don't get the results you expect. Find the optimal combinations of the configuration parameters (`efConstruction`, `maxConnections` and `ef`) to achieve the best results for your production configuration. (See [HNSW Configuration Tips](/weaviate/config-refs/indexing/vector-index.mdx#hnsw-configuration-tips))

<!---
:::warning TODO
Add some more info related to our philosophy / [DuckDB blog post](https://motherduck.com/blog/perf-is-not-enough/)
:::
--->

## Measured Metrics

For each benchmark test, we set these HNSW parameters:
- **`efConstruction`** - Controls the search quality at build time.
- **`maxConnections`**	 - The number of outgoing edges a node can have in the HNSW graph.
- **`ef`** - Controls the search quality at query time.

:::info HNSW Parameter Configuration Guide
For good starting point values and performance tuning advice, see [HNSW Configuration Tips](/weaviate/config-refs/indexing/vector-index.mdx#hnsw-configuration-tips).
:::

<!---
:::warning TODO
Add a note about how many runs there were. Or, if there's only one run because of cost, let's add a note about that instead.
:::
--->

For each set of parameters, we've run 10,000 requests, and we measured the following metrics:

- The **Recall@10** and **Recall@100** - by comparing Weaviate's results to the ground truths specified in each dataset.
- **Multi-threaded Queries per Second (QPS)** - The overall throughput you can
  achieve with each configuration.
- **Individual Request Latency (mean)** - The mean latency over all 10,000 requests.
- **P99 Latency** - 99% of all requests (9,900 out of 10,000) have a latency that is lower than or equal to this number
- **Import time** - Since varying build parameters has an effect on import time, the import time is also included.

By request, we mean:
An unfiltered vector search across the entire dataset for the given test. All
latency and throughput results include the request path measured by the benchmark,
not a guarantee of production user latency. In particular:

* Each request time includes the network overhead for sending the results over the
  wire. The methodology records client and server in the same VPC, but their
  placement is not reconciled with the shared-instance description under
  [Hardware](#hardware).
* Each request includes retrieving all the matched objects from disk. This is
  a significant difference from `ann-benchmarks`, where the embedded libraries
  only return the matched IDs.

:::info
This benchmark is [open source](https://github.com/weaviate/weaviate-benchmarking), so you can reproduce the results yourself.
:::

## Benchmark Results
<!---
**Last updated: MM 20YY with Version**
:::warning TODO
Update information
:::
--->

:::caution Result provenance

The hardware descriptions on this page are inconsistent: the setup lists a `n4-highmem-16` instance with 16 vCPUs and 128 GB memory, while the FAQ previously attributed throughput to a 30-core machine and identified a GCP `c2` CPU type. Client placement is also ambiguous. A tested Weaviate release and run date are not identified for each result set.

The rows and charts below are retained as reported observations, not independently reproduced measurements. Until the original run artifacts reconcile these details, do not use them for QPS-per-core normalization, hardware-cost comparisons, or capacity extrapolation. Matching manifests need the server and benchmark commits, run date, machine and CPU topology, RAM and storage, client placement, worker count, cache state, workload settings, and raw result files.

:::

This section contains datasets modeled after the [ANN Benchmarks](https://github.com/erikbern/ann-benchmarks). Pick a dataset that is closest to your production workload:


| **Dataset** | **Number of Objects** | **Vector Dimensions** | **[Distance metric](https://weaviate.io/blog/distance-metrics-in-vector-search)** | **Use case** |
| --- | --- | --- | --- | --- |
| [SIFT1M](http://corpus-texmex.irisa.fr/) | 1 M | 128 | l2-squared | SIFT is a common ANN test dataset generated from image data |
| [DBPedia OpenAI](https://huggingface.co/datasets/mteb/dbpedia) | 1 M | 1536 | cosine | DBPedia dataset from MTEB embedded with OpenAI's ada002 model |
| [MSMARCO Snowflake](https://huggingface.co/datasets/Snowflake/mteb-retrieval-snowflake-arctic-embed-m-v1.5) | 8.8 M | 768 | l2-squared | MSMARCO dataset embedded with Snowflake's Arctic Embed M v1.5 model |
| [Sphere DPR](https://weaviate.io/blog/sphere-dataset-in-weaviate) | 10 M | 768 | dot | Meta's Sphere dataset (10M sample)  |

#### Benchmark Datasets
These are the results for each dataset:

<Tabs groupId="datasets">
<TabItem value="1" label="DBPedia OpenAI">

#### QPS vs Recall for DBPedia OpenAI

<ThemedImage
  alt="DBPedia OpenAI Benchmark results"
  sources={{
    light: useBaseUrl('/img/benchmark/dbpedia-openai-1000k-angular-light.svg'),
    dark: useBaseUrl('/img/benchmark/dbpedia-openai-1000k-angular-dark.svg'),
  }}
/>

<Tabs groupId="limits">
  <TabItem value="10" label="Limit 10">
    <BenchmarkGrid datasetLabel="dbpediaOpenai_10" />
  </TabItem>
  <TabItem value="100" label="Limit 100">
    <BenchmarkGrid datasetLabel="dbpediaOpenai_100" />
  </TabItem>
</Tabs>

<AnnReadResultsTable/>

#### Recommended configuration for DBPedia OpenAI ada002
<RecommendedConfig/>

| `efConstruction` | `maxConnections` | `ef` | **Recall@10** | **QPS (Limit 10)** | **Mean Latency (Limit 10**) | **p99 Latency (Limit 10)** |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 256 | 16 | 96 | 97.24% | 5639 | 2.80ms | 4.43ms |

</TabItem>
<TabItem value="2" label="SIFT1M">

#### QPS vs Recall for SIFT1M

<ThemedImage
  alt="SIFT1M Benchmark results"
  sources={{
    light: useBaseUrl('/img/benchmark/sift-128-euclidean-light.svg'),
    dark: useBaseUrl('/img/benchmark/sift-128-euclidean-dark.svg'),
  }}
/>

<Tabs groupId="limits">
  <TabItem value="10" label="Limit 10">
    <BenchmarkGrid datasetLabel="sift128_10" />
  </TabItem>
  <TabItem value="100" label="Limit 100">
    <BenchmarkGrid datasetLabel="sift128_100" />
  </TabItem>
</Tabs>

import AnnReadResultsTable from '/_includes/ann-read-results-table.mdx';

<AnnReadResultsTable/>

#### Recommended configuration for SIFT1M
import RecommendedConfig from '/_includes/ann-recommended-config.mdx';

<RecommendedConfig/>

| `efConstruction` | `maxConnections` | `ef` | **Recall@10** | **QPS (Limit 10)** | **Mean Latency (Limit 10**) | **p99 Latency (Limit 10)** |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 256 | 32 | 64 | 98.35% | 10940 | 1.44ms | 3.13ms |


</TabItem>
<TabItem value="3" label="MSMARCO Snowflake">

#### QPS vs Recall for MSMARCO Snowflake

<ThemedImage
  alt="MSMARCO Snowflake Benchmark results"
  sources={{
    light: useBaseUrl('/img/benchmark/snowflake-msmarco-arctic-embed-m-v1-light.svg'),
    dark: useBaseUrl('/img/benchmark/snowflake-msmarco-arctic-embed-m-v1-dark.svg'),
  }}
/>

<Tabs groupId="limits">
  <TabItem value="10" label="Limit 10">
    <BenchmarkGrid datasetLabel="msmarcoSnowflake_10" />
  </TabItem>
  <TabItem value="100" label="Limit 100">
    <BenchmarkGrid datasetLabel="msmarcoSnowflake_100" />
  </TabItem>
</Tabs>

<AnnReadResultsTable/>

#### Recommended configuration for MSMARCO Snowflake
<RecommendedConfig/>

| `efConstruction` | `maxConnections` | `ef` | **Recall@10** | **QPS (Limit 10)** | **Mean Latency (Limit 10**) | **p99 Latency (Limit 10)** |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 384 | 32 | 48 | 97.36% | 7363 | 2.15ms | 3.69ms |

</TabItem>
<TabItem value="4" label="Sphere DPR">

#### QPS vs Recall for Sphere DPR

<ThemedImage
  alt="Sphere DPR Benchmark results"
  sources={{
    light: useBaseUrl('/img/benchmark/sphere-10M-meta-dpr-light.svg'),
    dark: useBaseUrl('/img/benchmark/sphere-10M-meta-dpr-dark.svg'),
  }}
/>

<Tabs groupId="limits">
  <TabItem value="10" label="Limit 10">
    <BenchmarkGrid datasetLabel="sphereDpr_10" />
  </TabItem>
  <TabItem value="100" label="Limit 100">
    <BenchmarkGrid datasetLabel="sphereDpr_100" />
  </TabItem>
</Tabs>

<AnnReadResultsTable/>


#### Recommended configuration for Sphere DPR
<RecommendedConfig/>

| `efConstruction` | `maxConnections` | `ef` | **Recall@10** | **QPS (Limit 10)** | **Mean Latency (Limit 10**) | **p99 Latency (Limit 10)** |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 384 | 32 | 96 | 96.06% | 3523 | 4.49ms | 7.73ms |

</TabItem>
</Tabs>


## Benchmark Setup

### Scripts

This benchmark is [open source](https://github.com/weaviate/weaviate-benchmarking), so you can reproduce the results yourself.

### Hardware

![Setup with Weaviate and benchmark machine](/img/docs/weaviate_benchmark_setup.png)

The recorded setup describes one GCP `n4-highmem-16` instance with 16 vCPUs and 128 GB memory running both Weaviate and the benchmark scripts. This conflicts with the other hardware descriptions summarized under [Result provenance](#benchmark-results). The diagram is a setup illustration, not evidence that each result used that topology.

The machine type and client placement for each result set remain unverified. Do not select a production machine by scaling these QPS figures; see the [CPU-scaling FAQ](#what-happens-if-i-run-with-fewer-or-more-cpu-cores-than-on-the-example-test-machine).

### Experiment Setup

We modeled our dataset selection after
[ann-benchmarks](https://github.com/erikbern/ann-benchmarks). The same test
queries are used to test speed, throughput, and recall. The provided ground
truths are used to calculate the recall.

We use Weaviate's Golang client to import data.
We use Go to measure the concurrent (multi-threaded) queries.
Each language has its own performance characteristics.
You may get different results if you use a different language to send your queries.

For maximum throughput, we recommend using the [Go](/weaviate/client-libraries/go/index.md) or
[Java](/weaviate/client-libraries/java/index.mdx) client libraries.

The complete import and test scripts are available [here](https://github.com/weaviate/weaviate-benchmarking).

## Benchmark FAQ

### How can I get the most performance for my use case?
If your use case is similar to one of the benchmark tests, use the recommended HNSW parameter configurations to start tuning.

For more instructions on how to tune your configuration for best performance, see [HNSW Configuration Tips](/weaviate/config-refs/indexing/vector-index.mdx#hnsw-configuration-tips).

### What is the difference between latency and throughput?

The latency refers to the time it takes to complete a single request. This
is typically measured by taking a mean or percentile distribution of all
requests. For example, a mean latency of 5ms means that a single request takes, on average, 5ms to complete. This does not say anything about how many queries
can be answered in a given timeframe.

If Weaviate were single-threaded, the throughput per second would roughly equal
to 1s divided by mean latency. For example, with a mean latency of 5ms, this
would mean that 200 requests can be answered in a second.

However, in reality, you often don't have a single user sending one query after
another. Instead, you have multiple users sending queries. This makes the
querying side concurrent. Similarly, Weaviate can handle concurrent incoming
requests. We can identify how many concurrent requests can be served by measuring
the throughput.

We can take our single-thread calculation from before and multiply it with the
number of server CPU cores. This will give us a rough estimate of what the
server can handle concurrently. However, it would be best never to trust this
calculation alone and continuously measure the actual throughput. This is because
such scaling may not always be linear. For example, there may be synchronization
mechanisms used to make concurrent access safe, such as locks. Not only do
these mechanisms have a cost themselves, but if implemented incorrectly, they
can also lead to congestion, which would further decrease the concurrent
throughput. As a result, you cannot perform a single-threaded benchmark and
extrapolate what the numbers would be like in a multi-threaded setting.

The QPS figures are reported multi-threaded measurements, not estimates. Their machine and worker-count provenance still needs reconciliation as described under [Benchmark Results](#benchmark-results).

### What is a p99 latency?

The mean latency gives you an average value of all requests measured. This is a
good indication of how long a user will have to wait on average for
their request to be completed. Based on this mean value, you cannot make any
promises to your users about wait times. 90 out of 100 users might see a
considerably better time, but the remaining 10 might see a significantly worse
time.

Percentile-based latencies are used to give a more precise indication. A
99th-percentile latency - or "p99 latency" for short - describes a percentile
of the measured request distribution. It provides information about the observed
tail that a mean does not, but a sampled benchmark p99 is not a guarantee that
99% of production users will experience that latency.

In production settings, requirements - as stated in support plans - are often a
combination of throughput and a percentile latency. For example, the statement
"3000 QPS at p95 latency of 20ms" conveys the following meaning.

- 3000 requests need to be successfully completed per second
- 95% of users must see a latency of 20ms or lower.
- There is no assumption about the remaining 5% of users, implicitly tolerating
  that they will experience higher latencies than 20ms.

The higher the percentile (e.g. p99 over p95) the "safer" the quoted
latency becomes. We have thus decided to use p99-latencies instead of
p95-latencies in our measurements.

### What happens if I run with fewer or more CPU cores than on the example test machine?

Do not derive QPS per core from these results while their hardware provenance is unresolved. Even with a verified machine configuration, throughput need not scale linearly with core count. Measure your workload at the required recall, concurrency, and tail latency on the intended hardware.

Adding more CPUs reaches a point of diminishing returns because of synchronization mechanisms, disk, and memory bottlenecks. Beyond that point, you  should scale horizontally instead of vertically. Horizontal scaling with replication is also available.

### What are `ef`, `efConstruction`, and `maxConnections`?

These parameters refer to the [HNSW build and query
parameters](/weaviate/config-refs/indexing/vector-index.mdx#hnsw-index).
They represent a trade-off between recall, latency & throughput, index size, and
memory consumption. This trade-off is highlighted in the benchmark results.

### I can't match the same latencies/throughput in my own setup. How can I debug this?

If you are encountering other numbers in your own dataset, here are a couple of
hints to look at:

* What CPU architecture and instruction sets are you using? CPU type can affect
  distance-computation performance. Record the CPU model and architecture when
  comparing results. This page's conflicting hardware descriptions do not
  establish a verified CPU type to match.

* Are you using an actual dataset or random vectors? HNSW is known to perform
  considerably worse with random vectors than with real-world datasets. This is due
  to the distribution of points in real-world datasets compared to randomly
  generated vectors. If you cannot achieve the performance (or recall)
  outlined above with random vectors, switch to an actual dataset.

* Are your disks fast enough? While the ANN search itself is CPU-bound, the objects
  must be read from disk after the search has been completed. Weaviate
  uses memory-mapped files to speed this process up. However, if not enough
  memory is present or the operating system has allocated the cached pages
  elsewhere, a physical disk read needs to occur. If your disk is slow,
  it could then be that your benchmark is bottlenecked by those disks.

* Are you using more than 2 million vectors? If yes, make sure to set the
  [vector cache large enough](/weaviate/concepts/resources.md#vector-cache)
  for maximum performance.

### Where can I find the scripts to run this benchmark myself?

The [repository is located here](https://github.com/weaviate/weaviate-benchmarking).


## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
