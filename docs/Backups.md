# Backups under load

Backups that take place in a Galera cluster can induce flow control messages and block writes in the cluster in two ways. 

- The backup creates enough "load" on the node that it cannot keep up with applying the write sets from the cluster and it emits a flow control message. 
- The backup creates locks which prevent writes from being applied so that the queue grows and the node will emit a flow control message. 

Both cases will cause a cluster wide outage in production if writes are halted due to a backup, even on a node that wasn't being used for anything else. 



## Desynchronization 

Galera has built in states designed to "desync" a node from its cluster so that it is assumed to be non-functional as far as synchronous replication is concerned, but it shall remain part of the cluster with the idea that in the future it will resume normal operations. 

```
SHOW VARIABLES LIKE 'wsrep_desync';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wsrep_desync  | OFF   |
+---------------+-------+
SET GLOBAL wsrep_desync='ON'; 
```

You can also desync a node using the `/*! WSREP_DESYNC */` query comment.



This state prevents the node from emitting flow control messages, but it still participates in other cluster wide operations such as quorum. 



** Do not desync multiple nodes of a cluster at the same time 



## Backup: 

```
mysql -e "SET GLOBAL wsrep_desync='ON';"

mydumper -h localhost ... 

mysql -e "SET GLOBAL wsrep_desync='OFF';
```



## From Desync -> Sync

Once a node is expected to resume synchronous replication with the cluster it will perform a IST to rejoin or SST if IST is not possible. 

Avoiding SST during resync: 

- Let the node catch up before turning off desync
  - `show status where variable_name in ('wsrep_local_recv_queue_max','wsrep_local_recv_queue_min');`
- Increase the size of gcache on the node you're using to backup



## Alternative option

If a cluster is in a state where a backup would induce flow control and no single node can be put into a desync'd state, the alternative is to create an asynchronous slave or use one if it already exists for the purpose of backups. 

