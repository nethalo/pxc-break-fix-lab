# WAN

Distributed systems need a reliable network for good performance. When running the cluster over WAN, you may frequently experience transient network connectivity failures. To prevent this from partitioning the cluster, you may want to increase the keepalive timeouts.

## Segments

Galera 3 (available with [PXC 5.6](https://www.percona.com/downloads/Percona-XtraDB-Cluster/)) introduces a new feature called WAN segments that basically implements the relay-slave concept, but in a more elegant way.  To enable this, we simply assign each node in a given data center a common *gmcast.segment* integer in wsrep_provider_options.  Each data center must have a distinct identifier and each node in that data center should have the same segment.