---
title: Multi-Data center
sidebar_position: 5
description: "Run a Weaviate cluster across multiple data centers (WAN) for lower latency and cross-region redundancy."
image: og/docs/concepts.jpg
# tags: ['architecture']
---


Multi-Data center (Multi-DC) replication enables you to have multiple copies of the data on multiple servers across more than one data center. Weaviate supports running a single cluster across multiple data centers , so you can place replicas in different regions to improve locality and redundancy, subject to actual shard placement and operation requirements.

Multi-DC replication is beneficial if you have user groups spread over different geographical locations (e.g. Iceland and Australia). When you place nodes in different local regions of user groups, latency can be decreased.
Cross-region replication can provide copies in more than one physical location, but surviving a datacenter loss depends on placement and request requirements.

If all replica nodes are in the same data center, network requests between nodes are cheap and fast, but the whole deployment is at risk if that data center goes down. Spreading nodes across data centers does not itself ensure a cross-region copy of every shard. Verify actual replica placement: each required shard needs enough surviving healthy, eligible replicas for the requested operation, plus a reachable entry point. Metadata progress requires a majority of Raft voting members. A three-voter split of two in one datacenter and one in another cannot retain a majority after losing the two-voter location.

To run a cluster across data centers, tune Weaviate's inter-node networking for the higher-latency, lower-reliability links between regions. See [Running a single cluster across data centers (WAN)](#running-a-single-cluster-across-data-centers-wan) below.

<p align="center"><img src="/img/docs/replication-architecture/replication-regional-proximity-3.png" alt="Replication multi-dc" width="75%"/></p>

## Running a single cluster across data centers (WAN)

Weaviate can operate a single cluster across data centers (a wide-area network, or WAN) by adjusting its inter-node communication for high-latency, lower-reliability links.

### Availability and placement

WAN mode tunes inter-node communication; it does not configure replication factors, automatically balance regions, or guarantee failover. A local entry point can still require remote replica acknowledgments or metadata coordination. See [consistency](./consistency.md#tunable-write-consistency) for operation-scoped QUORUM/ALL behavior and [replica movement](/deploy/configuration/replica-movement.mdx) for deliberate placement changes.

### Enabling WAN mode

Set the `CLUSTER_ADVERTISE_ADDR` environment variable to enable WAN mode. When it is set, Weaviate switches its internal [memberlist](https://github.com/hashicorp/memberlist) configuration to `DefaultWANConfig`, which increases timeouts and relaxes failure-detection thresholds so they are suitable for cross-data center communication.

### Key environment variables

| Variable | Description | Required |
| --- | --- | --- |
| `CLUSTER_ADVERTISE_ADDR` | The public IP address that other nodes should use to reach this node. Setting this enables WAN mode. Must be a valid IP address (hostnames are rejected). | Yes (for WAN) |
| `CLUSTER_ADVERTISE_PORT` | The port to advertise to other nodes. If not set, it defaults to `CLUSTER_GOSSIP_BIND_PORT`. If set, it must be between `1024` and `65535`. | No |
| `CLUSTER_BIND_ADDR` | The local address to bind to. Defaults to `0.0.0.0`. | No |
| `CLUSTER_GOSSIP_BIND_PORT` | The port used for gossip (memberlist) traffic. Defaults to `7946`. | No |
| `CLUSTER_DATA_BIND_PORT` | The port used for data traffic. Defaults to `CLUSTER_GOSSIP_BIND_PORT + 1`. | No |
| `CLUSTER_JOIN` | A comma-separated list of `host:port` addresses of existing cluster members to join. | Yes (for joining) |

### Example configuration

Below is an example environment configuration for a node intended to participate in a cross-data center cluster:

```yaml
environment:
  - CLUSTER_HOSTNAME=weaviate-0
  - CLUSTER_ADVERTISE_ADDR=203.0.113.10 # Public IP of this node
  - CLUSTER_JOIN=203.0.113.20:7946 # Public IP:Port of a node in another DC
  - CLUSTER_BIND_ADDR=0.0.0.0 # Optional; this is already the default bind address
  - CLUSTER_GOSSIP_BIND_PORT=7946
  - CLUSTER_DATA_BIND_PORT=7947
  - RAFT_BOOTSTRAP_EXPECT=3 # Illustrative three-voter bootstrap; majority is two
  - RAFT_JOIN=weaviate-0:8300,weaviate-1:8300,weaviate-2:8300 # Cluster node names, not IP addresses
```

The three `RAFT_JOIN` names must match the nodes' `CLUSTER_HOSTNAME` values and resolve through cluster membership. Configure each node's own advertised address. Permit peer traffic on the configured gossip/data ports, Raft transport (default `8300`), and internal Raft RPC (default `8301`). An odd voter count does not by itself guarantee datacenter-loss tolerance or faster elections.

### Notes

- **Latency**: Cross-data center operations inherently have higher latency. Make sure your application timeouts account for this.
- **Security**: Gossip and data traffic are not encrypted by default. For cross-data center communication over the public internet, use a VPN or other secure tunnel, or make sure `CLUSTER_ADVERTISE_ADDR` is only reachable over a private network (for example, VPC peering).

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
