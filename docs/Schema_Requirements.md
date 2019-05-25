# Schema Requirements

One of the requirements is that all tables must be InnoDB and have a primary key

## Schema Changes

Schema changes can be quite a challenge in PXC replication. This is due to the multi-master nature of this replication topology, where data consistency must be preserved when changes can be done in the same time on many nodes. And while conflict resolution is pretty effective for DML updates, it cannot be done the same way for DDL queries as they are not transactional in MySQL.

In this article you will find the available methods for executing DDL queries in PXC with some guide on when to use each of them.

### Total Order Isolation (TOI)

This is the default Online Schema Upgrade method, defined with [wsrep_OSU_method](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_OSU_method) variable. It guarantees data consistency as DDL statements are executed in exactly the same order on all the nodes with respect to all other transactions executed on cluster nodes. This also means such upgrade will have to block writes across the whole cluster for it's duration. For a very short alters, this usually is not a problem, but expensive ones may render cluster unusable for often too long. Worth mentioning is that Online DDL feature introduced in MySQL 5.6 will not work as expected in Galera, due to the TOI nature.

This mode is the fastest and most safe one though, so recommended anytime when you can plan for downtime or when short lock time is not a problem.

### pt-online-schema-change

Anytime when you want to avoid potential long downtime caused by DDL query, you may use this tool, which was originally invented when there was no Online DDL feature in MySQL, but still has many use cases these days. It is important to understand though, that pt-osc always re-creates the whole table by copying the existing data (and new incoming during the process) to a table with new schema definition, regardless of what the difference is. This implicates the need to reserve at least same amount of free disk space as the table currently occupies to allow the copy to succeed. It has though some limitations, like for example when there are any triggers on the table already.

Even though this method is not blocking for most of the alter duration, it will still need to execute some (fast) TOI statements, like create needed triggers, create new table as well as final RENAME. It may matter especially in case it will not be able to get the MDL lock due to long ongoing transactions affecting the table you are going to alter.

The pt-online-schema-change is especially useful for long, schema incompatible changes, like adding or removing columns, changing data types and so on, but also for the OPTIMIZE/noop alter query. But for alterations that are very fast and light by nature, like dropping or adding secondary keys, renaming columns or similar, it will usually NOT make sense at all.

The tool is also Galera aware thanks to the `--max-flow-ctl `option, which can adjust copy process speed to the cluster's replication delays.

You should also check this article addressing potential problem with the tool in PXC: <https://customers.percona.com/hc/en-us/articles/360000367649-pt-online-schema-change-Error-5-During-Commit-PXC>

### Rolling Schema Upgrade (RSU)

This method may provide the least blocking schema change scenario for Galera based clusters, however it is also one that must be used with the greatest care.

It is actually very similar to a workaround being used sometimes for asynchronous replication, where schema is changed first on the slave, which is then promoted so that the master can do same alter without interrupting the service. In PXC/Galera, this method practically means that when you change the OSU method like this in your session:

```
 mysql> set session wsrep_OSU_method=RSU;
Query OK, 0 rows affected (0.01 sec)

  mysql> show global  variables like "wsrep_OSU_method";
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| wsrep_OSU_method | RSU   |
+------------------+-------+
1 row in set (0.01 sec)

2019-05-22T17:16:27.631556Z 0 [Note] WSREP: Member 2.0 (pxc1) desyncs itself from group
2019-05-22T17:16:27.645724Z 0 [Note] WSREP: Shifting SYNCED -> DONOR/DESYNCED (TO: 715957)
2019-05-22T17:16:27.683374Z 14 [Note] WSREP: Provider paused at bcf6669e-6db5-11e9-9815-0b05174fd831:715957 (48091)


mysql> show create table sbtest.sbtest1\G
*************************** 1. row ***************************
       Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB AUTO_INCREMENT=3000003 DEFAULT CHARSET=latin1 MAX_ROWS=1000000
1 row in set (0.00 sec)

mysql> ALTER TABLE sbtest.sbtest1 ADD COLUMN d CHAR(100) NOT NULL DEFAULT '';                                                                                        Query OK, 0 rows affected (54.17 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show create table sbtest.sbtest1\G
*************************** 1. row ***************************
       Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  `d` char(100) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB AUTO_INCREMENT=3000003 DEFAULT CHARSET=latin1 MAX_ROWS=1000000
1 row in set (0.06 sec)

mysql> set session wsrep_OSU_method=TOI;
Query OK, 0 rows affected (0.01 sec)

pxc2:

mysql> show create table sbtest.sbtest1\G
*************************** 1. row ***************************
       Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB AUTO_INCREMENT=3000001 DEFAULT CHARSET=latin1 MAX_ROWS=1000000
1 row in set (0.03 sec)



```

from now on, any DDL statement executed WILL NOT BE REPLICATED. Therefore, it will not block the other cluster nodes, but in order to keep the cluster in sync, you will have to repeat the same ALTER on each of them separately. In addition, RSU method automatically puts the node in [DESYNCED state](https://customers.percona.com/hc/en-us/articles/115005359909-How-the-Desynced-Node-Works), which means it will not emit any Flow Control pause if the alter blocks it's own replication queue applying process.

With that said, **RSU method should be really avoided for any schema incompatible changes**, as those may quickly destroy your cluster introducing [data inconsistency](https://customers.percona.com/hc/en-us/articles/115004449985-Possible-reasons-for-data-differences-in-PXC), where nodes which cannot apply row changes due to incompatible schema will have to abort and shutdown. So, if you are not 100% sure if schema change is safe in that method, please consider TOI or pt-online-schema-change or just file a ticket to Percona Support for assistance.

For example, adding or changing a column is surely not compatible change, while adding or removing secondary keys is compatible, as it does not affect the data itself. However, changing primary key properties or even adding or removing unique key, may be very well harmful here. A very good example of proper use case is when you are testing new indexes during query optimization attempts. It is just important to make sure that finally all nodes have the same indexes in order to avoid inconsistent query performance. You may even run the [**mysqldbcompare** ](https://dev.mysql.com/doc/mysql-utilities/1.5/en/mysqldbcompare.html)(with --skip-row-count and --skip-data-check options) tool occasionally to make sure schemas are consistent between the nodes.

### Aborting DDL

One important fact to mention here, very different from traditional replication, is that once a DDL is started, whether is was in TOI or RSU mode, **there is no way to cancel or abort it**. Even within the alter's owner session you cannot kill the query, as seen on example below:

```
pxc03 > alter table tb1 add key(a);
^C^C -- query aborted
Query OK, 0 rows affected (2 min 10.64 sec)
Records: 0  Duplicates: 0  Warnings: 0

-- second session:
pxc03 > select * from information_schema.processlist where info like 'alter%';
+----+------+-----------+------+---------+------+----------------+----------------------------+---------+-----------+---------------+
| ID | USER | HOST      | DB   | COMMAND | TIME | STATE          | INFO                       | TIME_MS | ROWS_SENT | ROWS_EXAMINED |
+----+------+-----------+------+---------+------+----------------+----------------------------+---------+-----------+---------------+
| 70 | root | localhost | test | Query   |    9 | altering table | alter table tb1 add key(a) |    8868 |         0 |             0 |
+----+------+-----------+------+---------+------+----------------+----------------------------+---------+-----------+---------------+
1 row in set (0.01 sec)

pxc03 > kill query 70;
ERROR 1095 (HY000): You are not owner of thread 70
pxc03 > kill 70;
ERROR 1095 (HY000): You are not owner of thread 70

```

Of course, in case of long TOI alter, unexpectedly long one, it may cause serious issues with the cluster availability, while for RSU, at least other nodes should stay pretty much unaffected.

### References:

[https://www.percona.com/blog/2015/10/09/online-ddl-percona-xtradb-cluster-5-6/ https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_OSU_method](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_OSU_method)<http://galeracluster.com/documentation-webpages/schemaupgrades.html> <https://www.percona.com/blog/2015/07/06/toi-wsrep_rsu_method-pxc-5-6-24/>

This article applies to the following versions of technologies:

| Percona XtraDB Cluster Versions | 5.5.x, 5.6.x, 5.7.x |
| ------------------------------- | ------------------- |
|                                 |                     |

