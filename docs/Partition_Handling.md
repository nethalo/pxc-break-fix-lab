# Partition Handling (Network Hiccup)

In Galera, when the network connection between nodes is lost, those who still have a quorum will form a new cluster view. Those who lost a quorum keep trying to re-connect to the primary component. Once the connection is restored, separated nodes will sync back using IST and rejoin the cluster automatically.

In a Galera-based cluster, you are automatically protected from actually perform DDL, and a partitioned node refuses to allow both reads and writes. It throws an error:  ERROR 1047 (08S01): WSREP has notyet prepared node for application use. You can force dirty reads using the [wsrep_dirty_reads](https://www.percona.com/doc/percona-xtradb-cluster/5.7/wsrep-system-index.html#wsrep_dirty_reads) variable.