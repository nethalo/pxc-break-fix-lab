# Multi thread

Galera developed its own multi-threaded slave feature, even in 5.5 versions, for workloads that include tables in the same database. It is controlled with the  [wsrep_slave_threads](https://www.percona.com/doc/percona-xtradb-cluster/5.6/wsrep-system-index.html#wsrep_slave_threads) variable. 