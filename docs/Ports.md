# Ports

- 3306 is used for MySQL client connections and [SST](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-sst) (State Snapshot Transfer) via `mysqldump`.
- 4444 is used for [SST](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-sst) via `rsync` and [Percona XtraBackup](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/manual/xtrabackup_sst.html#xtrabackup-sst).
- 4567 is used for write-set replication traffic (over TCP) and multicast replication (over TCP and UDP).
- 4568 is used for [IST](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-ist) (Incremental State Transfer).

