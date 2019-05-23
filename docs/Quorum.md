# Quorum

A majority (> 50%) of nodes. In the event of a network partition, only the cluster partition that retains a quorum (if any) will remain Primary by default.

The size of the cluster is used to determine the required votes to achieve [quorum](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-quorum).

A quorum vote is done when a node or nodes are suspected to no longer be part of the cluster (they do not respond). 

This no response timeout is the [`evs.suspect_timeout`](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-provider-index.html#evs.suspect_timeout) setting in the [`wsrep_provider_options`](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_provider_options) (default 5 sec), and when a node goes down ungracefully, write operations will be blocked on the cluster for slightly longer than that timeout.

Cluster membership is determined simply by which nodes are connected to the rest of the cluster; there is no configuration setting explicitly defining the list of all possible cluster nodes. Therefore, every time a node joins the cluster, the total size of the cluster is increased and when a node leaves (gracefully) the size is decreased