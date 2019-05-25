## Data Inconsistencies in PXC

As Galera replication is designed for zero data inconsistency tolerance, this problem is even more important than in traditional replication. And although it is much less likely to end up with data being different between PXC cluster nodes, still such problem may occur.

The usual error seen after a data inconsistency problem, may look like this:

```
pxc1:

mysql> show session variables like "wsrep_on";
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wsrep_on      | ON    |
+---------------+-------+
1 row in set (0.16 sec)
mysql> set session wsrep_on=0;
Query OK, 0 rows affected (0.09 sec)

mysql> show session variables like "wsrep_on";
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wsrep_on      | OFF   |
+---------------+-------+
1 row in set (0.13 sec)

pxc1:mysql>select max(id) from sbtest.sbtest1;
+---------+
| max(id) |
+---------+
| 3000000 |
+---------+
1 row in set (0.03 sec)

pxc1:mysql>delete from sbtest.sbtest1 where id=3000000;
Query OK, 1 row affected (0.16 sec)

pxc1:mysql>set session wsrep_on=1;
Query OK, 0 rows affected (0.06 sec)


pxc2:mysql>select max(id) from sbtest.sbtest1;
+---------+
| max(id) |
+---------+
| 3000000 |
+---------+
1 row in set (0.02 sec)
pxc1 log:

2019-05-22T17:17:21.649623Z 14 [Note] WSREP: resuming provider at 48091
2019-05-22T17:17:21.650006Z 14 [Note] WSREP: Provider resumed.
2019-05-22T17:17:21.653964Z 0 [Note] WSREP: Member 2.0 (pxc1) resyncs itself to group
2019-05-22T17:17:21.654001Z 0 [Note] WSREP: Shifting DONOR/DESYNCED -> JOINED (TO: 715957)
2019-05-22T17:17:21.712542Z 0 [Note] WSREP: Member 2.0 (pxc1) synced with group.
2019-05-22T17:17:21.712722Z 0 [Note] WSREP: Shifting JOINED -> SYNCED (TO: 715957)
2019-05-22T17:17:21.713817Z 10 [Note] WSREP: Synchronized with group, ready for connections
2019-05-22T17:17:21.714784Z 10 [Note] WSREP: Setting wsrep_ready to true

pxc2:mysql>delete from sbtest.sbtest1 where id=3000000;
Query OK, 1 row affected (0.09 sec)

pxc1 log:
2019-05-22T17:30:18.539589Z 5 [ERROR] WSREP: Failed to apply trx 746293 4 times
2019-05-22T17:30:18.539601Z 5 [ERROR] WSREP: Node consistency compromised, aborting...
.
..
...
2019-05-22T17:30:59.991784Z 0 [Note] WSREP: GCache history reset: bcf6669e-6db5-11e9-9815-0b05174fd831:0 -> 00000000-0000-0000-0000-000000000000:-1
2019-05-22T17:30:59.993338Z 0 [Note] WSREP: Assign initial position for certification: -1, protocol version: -1
2019-05-22T17:30:59.993652Z 0 [Note] WSREP: Preparing to initiate SST/IST
2019-05-22T17:30:59.993677Z 0 [Note] WSREP: Starting replication
2019-05-22T17:30:59.993704Z 0 [Note] WSREP: Setting initial position to 00000000-0000-0000-0000-000000000000:-1
.
..
...
2019-05-22T17:32:51.781238Z 0 [Note] WSREP: SST leaving flow control
2019-05-22T17:32:51.781244Z 0 [Note] WSREP: Shifting JOINER -> JOINED (TO: 749616)
2019-05-22T17:32:53.529636Z 0 [Note] WSREP: Member 0.0 (pxc1) synced with group.
2019-05-22T17:32:53.529669Z 0 [Note] WSREP: Shifting JOINED -> SYNCED (TO: 749696)
2019-05-22T17:32:54.848298Z 11 [Note] WSREP: Synchronized with group, ready for connections
2019-05-22T17:32:54.848334Z 11 [Note] WSREP: Setting wsrep_ready to true
2019-05-22T17:32:54.848508Z 11 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.

```

So what could be the reason for data differences between cluster nodes, in **Galera based** **replication** environment?

## List of the most typical culprits

- Write explicitly set to not be replicated (by users having SUPER privilege), with:
  - set [session] [sql_log_bin](https://dev.mysql.com/doc/refman/5.7/en/set-sql-log-bin.html)=0
  - set [session] wsrep_on=0
- Tables having an unsupported storage engine
- Incompatible ALTER executed in RSU
- log_slave_updates not enabled when PXC node acts as an async slave
- Write [non-deterministic](https://dev.mysql.com/doc/refman/5.7/en/replication-rbr-safe-unsafe.html) in binlog_format=STATEMENT
- [replication filters](https://www.percona.com/blog/2007/11/07/filtered-mysql-replication/) used
- Node [re-]started with wsrep_provider not set, while still allowing connections (like during an upgrade step)
- Faulty SST
- [wsrep_preordered](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_preordered) misused on async slave node
- Writeset (internally binary log event) corrupted, due to bug: https://bugs.mysql.com/bug.php?id=72457
- Split brain scenario, like when two partitioned cluster parts have Primary state at the same time, due to human error
- Wrong node bootstrapped after [whole cluster down scenario](https://www.percona.com/blog/2014/09/01/galera-replication-how-to-recover-a-pxc-cluster/)
- A node bootstrapped when other nodes already running
- [pxc_strict_mode not same on all nodes](https://bugs.launchpad.net/percona-xtradb-cluster/+bug/1663759)
- [tablespace encryption](https://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace-encryption.html) not set equally on all nodes
- Other [bugs](https://jira.percona.com/projects/PXC/issues/PXC-2039)

## How to avoid data inconsistencies

As you can see, there are many possible scenarios that can lead to this problem. And most of the cases can be avoided by few simple things:

- use **PXC Strict Mode** enforced
- avoid giving too much privileges to users, especially SUPER one
- respect safe_to_bootstrap information from grastate.dat and update it very carefully when needed
- avoid unsafe settings
- be careful with maintenance

## How to investigate

To investigate a particular case in more detail, we will usually need to check:

- full error logs from all nodes, even old ones with possible crash information
- configuration files (my.cnf etc) plus show variables output, to get actual values if my.cnf modified after server startup
- GRA* files created on aborted nodes - every failed write set will be logged in GRA_x_y.dat log file, where x=thread ID and y=seqno
- if binlogs are enabled, check each node's history - a node who originated failed update, will have failed

Please note that data differences may get undetected (no cluster aborts) for a long time or not at all, if affected data is not updated later. Or in case data differences do not span to primary/unique keys. Let's take such situation for instance:

```
CREATE TABLE `t2` (
`id` int(11) NOT NULL,
`a` int(11) DEFAULT '2',
PRIMARY KEY (`id`)
) ENGINE=InnoDB

pxc-cluster-node-1> select * from test.t2;
+----+------+
| id | a |
+----+------+
| 3  | 3 |
+----+------+
1 row in set (0.00 sec)

pxc-cluster-node-3> select * from test.t2;
+----+------+
| id | a |
+----+------+
| 3  | 5 |
+----+------+
1 row in set (0.00 sec)

pxc-cluster-node-1> update test.t2 set a=10 where id=3;
Query OK, 1 row affected (0.45 sec)
Rows matched: 1 Changed: 1 Warnings: 0

pxc-cluster-node-1> select * from test.t2;
+----+------+
| id | a |
+----+------+
| 3  | 10 |
+----+------+
1 row in set (0.00 sec)

pxc-cluster-node-3> select * from test.t2;
+----+------+
| id | a |
+----+------+
| 3  | 10 |
+----+------+
1 row in set (0.00 sec)
```

As seen above, initial data inconsistency in column a, was completely undetected and overwritten by later update. It happened as PK values were same, and row identification used PK only.

For these reasons, finding a root cause for replication error may be extremely difficult, if not impossible some times.

We encourage to check cluster data consistency regularly with pt-table-checksum tool or similar.

- https://www.percona.com/live/17/sessions/galera-cluster-data-consistency
- https://www.percona.com/blog/2014/07/21/a-schema-change-inconsistency-with-galera-cluster-for-mysql/

## 
