# PXC Break/Fix Tutorial



Based on Percona XtraDB Cluster 5.7.25

3-node cluster

Running on CentOS 7



## Flow control

A technique to throttle writes when nodes are slow in applying. Since on PXC each node applies changes async and it prevents to fall too behind. 

If the number of write-sets waiting in the queue to be applied (`wsrep_local_recv_queue`) grows too large (than `gcs.fc_limit`), the node informs other nodes with a Flow Control message, to pause replication while it works on the processing events from a queue.

When `gcs.fc_master_slave` is disabled, limit of the queue is increased automatically based on a number of nodes. The theory behind this is that, for multi-master clusters, the larger cluster gets (and presumably busier with more writes coming from more nodes), the more leeway each node will get to be a bit further behind applying. PXC uses the following algorithm for dynamically resizing fc_limit:

```
LET fn = sqrt( numberOfNodes )
LET fc_limit = base_fc_limit * fn + .5
```

The following table presents an ultimate value of fc_limit depending on cluster size, with the base:

```
 fc_limit = 100
```

| Number of nodes | Calculated fc_limit |
| --------------- | ------------------- |
| 3               | 173                 |
| 4               | 200                 |
| 5               | 224                 |
| 6               | 245                 |
| 7               | 265                 |

### Key variables

- **gcs.fc_limit:** the threshold.  if the *wsrep_local_recv_queue* exceeds this size on a given node, a pausing flow control message will be sent. 
- **gcs.fc_master_slave**: should galera modify dinamycally the limit or not
- **gcs.fc_factor:** once flow control kicks in, when it should be released. The factor is a number between 0.0 and 1.0, which is multiplied by the current *fc_limit*

### Monitor FC

On 5.7: show status like '**wsrep_flow_control_status**'; This boolean status variable tells the user if the node is in FLOW_CONTROL or not.

the **wsrep_flow_control_sent/recv** counter can be used to track FLOW_CONTROL status. 

Better check them all: `show status like 'w%flow%';`

### Simulate FC

- Disable PXC STRICT MODE: `set global pxc_strict_mode='DISABLED';`
- Lock a table: `use percona; lock tables sbtest9 write;`
- You will see messages like this in the error log: `2019-05-17T13:47:47.773417Z 8 [Note] WSREP: MDL conflict db=percona table=sbtest9 ticket=MDL_SHARED_NO_READ_WRITE solved by abort`
- Remove lock: `unlock tables;`
- Set back strict mode: `set global pxc_strict_mode='ENFORCING';`

## PXC Strict Mode

PXC Strict Mode is designed to avoid the use of experimental and unsupported features in Percona XtraDB Cluster. It performs a number of validations at startup and during runtime.

- `DISABLED`: Do not perform strict mode validations and run as normal.
- `PERMISSIVE`: If a vaidation fails, log a warning and continue running as normal.
- `ENFORCING`: If a validation fails during startup, halt the server and throw an error. If a validation fails during runtime, deny the operation and throw an error.
- `MASTER`: The same as `ENFORCING` except that the validation of [explicit table locking](https://www.percona.com/doc/percona-xtradb-cluster/5.7/features/pxc-strict-mode.html#explicit-table-locking) is not performed. This mode can be used with clusters in which write operations are isolated to a single node.

By default, PXC Strict Mode is set to `ENFORCING`, except if the node is acting as a standalone server or the node is bootstrapping, then PXC Strict Mode defaults to `DISABLED`.

https://www.percona.com/doc/percona-xtradb-cluster/5.7/features/pxc-strict-mode.html#validations

```
 `CREATE TABLE sbtest100 (`
  id int(10) unsigned NOT NULL AUTO_INCREMENT,
  k int(10) unsigned NOT NULL DEFAULT '0',
  c char(120) NOT NULL DEFAULT '',
  pad char(60) NOT NULL DEFAULT '',
  KEY k_1 (k)
`) ENGINE=InnoDB;`
```



## GRA Files

Those files correspond to a replication failure. That means the slave thread was not able to apply one transaction. For each of those file, a corresponding warning or error message is present in the mysql error log file.

```
2019-05-17T13:56:35.525364Z 8 [ERROR] Slave SQL: Error 'Incorrect table definition; there can be only one auto column and it must be defined as a key' on query. Default database: 'percona'. Query: 'CREATE TABLE sbtest100 (

  id int(10) unsigned NOT NULL AUTO_INCREMENT,

  k int(10) unsigned NOT NULL DEFAULT '0',

  c char(120) NOT NULL DEFAULT '',

  pad char(60) NOT NULL DEFAULT '',

  KEY k_1 (k)

) ENGINE=InnoDB', Error_code: 1075
2019-05-17T13:56:35.525848Z 8 [Warning] WSREP: RBR event 1 Query apply warning: 1, 36304
2019-05-17T13:56:35.571531Z 8 [Warning] WSREP: Ignoring error for TO isolated action: source: b7c194e5-7846-11e9-b6c4-2afd2e47cbf0 version: 4 local: 0 state: APPLYING flags: 65 conn_id: 19 trx_id: -1 seqnos (l: 36935, g: 36304, s: 36303, d: 36303, ts: 60090963212074)
```

Files are stored on the datadir:

```
[root@pxc2 mysql]# strings /var/lib/mysql/GRA_8_36304.log
percona
CREATE TABLE sbtest100 (
  id int(10) unsigned NOT NULL AUTO_INCREMENT,
  k int(10) unsigned NOT NULL DEFAULT '0',
  c char(120) NOT NULL DEFAULT '',
  pad char(60) NOT NULL DEFAULT '',
  KEY k_1 (k)
) ENGINE=InnoDB
```

<https://www.percona.com/blog/2012/12/19/percona-xtradb-cluster-pxc-what-about-gra_-log-files/>

Issues when no PK is available

Let's create (properly) the table:

CREATE TABLE `sbtest100` (
  `id` int(10) unsigned NOT NULL,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  KEY `k_1` (`k`)
) ENGINE=InnoDB;

And now let's populate it with 1 single big TRX:

INSERT INTO sbtest100 SELECT * from sbtest1;

The strict mode will prevent to do such a crazy query. Let's override it:

set global pxc_strict_mode='DISABLED'; INSERT INTO sbtest100 SELECT * from sbtest1; set global pxc_strict_mode='ENFORCING';

## Auto increment

Galera adjusts the auto_increment_increment and auto_increment_offset values according to the number of members in a cluster. So, for a 3-node cluster, auto_increment_increment  will be “3” and auto_increment_offset  from “1” to “3” (depending on the node). If a number of nodes change later, these are updated immediately. This feature can be disabled using the wsrep_auto_increment_control setting. If needed, these settings can be set manually.

## State Transfers

Galera has two types of state transfers that allow syncing data to nodes when needed: incremental (IST) and full (SST). Incremental is used when a node has been out of a cluster for some time, and once it rejoins the other nodes has the missing write sets still in Galera cache. Full SST is helpful if incremental is not possible, especially when a new node is added to the cluster. SST automatically provisions the node with fresh data taken as a snapshot from one of the running nodes (donor). The most common SST method is using Percona XtraBackup, which takes a fast and non-blocking binary data snapshot (hot backup).

## Multi Master

The more masters you have in the cluster the higher the probability of certification conflicts. This can lead to undesirable rollbacks and performance degradation.

If you find you experience frequent certification conflicts, consider reducing the number of nodes your cluster uses as masters.

## Group communication

Galera is using a proprietary group communication system layer, which implements a virtual synchrony QoS which is based on the Totem Single-ring Ordering protocol.

## Ports

- 3306 is used for MySQL client connections and [SST](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-sst) (State Snapshot Transfer) via `mysqldump`.
- 4444 is used for [SST](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-sst) via `rsync` and [Percona XtraBackup](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/manual/xtrabackup_sst.html#xtrabackup-sst).
- 4567 is used for write-set replication traffic (over TCP) and multicast replication (over TCP and UDP).
- 4568 is used for [IST](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-ist) (Incremental State Transfer).

## Quorum 

A majority (> 50%) of nodes. In the event of a network partition, only the cluster partition that retains a quorum (if any) will remain Primary by default.

The size of the cluster is used to determine the required votes to achieve [quorum](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-quorum).

A quorum vote is done when a node or nodes are suspected to no longer be part of the cluster (they do not respond). 

This no response timeout is the [`evs.suspect_timeout`](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-provider-index.html#evs.suspect_timeout) setting in the [`wsrep_provider_options`](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_provider_options) (default 5 sec), and when a node goes down ungracefully, write operations will be blocked on the cluster for slightly longer than that timeout.

Cluster membership is determined simply by which nodes are connected to the rest of the cluster; there is no configuration setting explicitly defining the list of all possible cluster nodes. Therefore, every time a node joins the cluster, the total size of the cluster is increased and when a node leaves (gracefully) the size is decreased

## WAN

Distributed systems need a reliable network for good performance. When running the cluster over WAN, you may frequently experience transient network connectivity failures. To prevent this from partitioning the cluster, you may want to increase the keepalive timeouts.

### Segments

Galera 3 (available with [PXC 5.6](https://www.percona.com/downloads/Percona-XtraDB-Cluster/)) introduces a new feature called WAN segments that basically implements the relay-slave concept, but in a more elegant way.  To enable this, we simply assign each node in a given data center a common *gmcast.segment* integer in wsrep_provider_options.  Each data center must have a distinct identifier and each node in that data center should have the same segment.

## GTID

Galera has it’s [own Global Transaction ID](http://galeracluster.com/documentation-webpages/architecture.html#global-transaction-id), which has existed since MySQL 5.5, and is independent from MySQL’s GTID feature introduced in MySQL 5.6.

To keep the state identical on all nodes, the [wsrep API](http://galeracluster.com/documentation-webpages/glossary.html#term-wsrep-api) uses global transaction IDs (GTID), which are used to both:

> - Identify the state change
> - Identify the state itself by the ID of the last state change

The GTID consists of:

> - A state UUID, which uniquely identifies the state and the sequence of changes it undergoes
> - An ordinal sequence number (seqno, a 64-bit signed integer) to denote the position of the change in the sequence

The Global Transaction ID allows you to compare the application state and establish the order of state changes. You can use it to determine whether or not a change was applied and whether the change is applicable at all to a given state.

## Schema Requirements

one of the requirements is that all tables must be InnoDB and have a primary key

## Schema upgrades

TOI RSU

## Multi thread

Galera developed its own multi-threaded slave feature, even in 5.5 versions, for workloads that include tables in the same database. It is controlled with the  [wsrep_slave_threads](https://www.percona.com/doc/percona-xtradb-cluster/5.6/wsrep-system-index.html#wsrep_slave_threads) variable. 

## Partition Handling (Network Hiccup)

In Galera, when the network connection between nodes is lost, those who still have a quorum will form a new cluster view. Those who lost a quorum keep trying to re-connect to the primary component. Once the connection is restored, separated nodes will sync back using IST and rejoin the cluster automatically.

In a Galera-based cluster, you are automatically protected from actually perform DDL, and a partitioned node refuses to allow both reads and writes. It throws an error:  ERROR 1047 (08S01): WSREP has notyet prepared node for application use. You can force dirty reads using the [wsrep_dirty_reads](https://www.percona.com/doc/percona-xtradb-cluster/5.7/wsrep-system-index.html#wsrep_dirty_reads) variable.
