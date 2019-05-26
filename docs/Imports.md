# Imports under load

Sometimes a situation may arise where you must complete an unusual task that will put a large load on a PXC cluster (in this case we'll assume a database import).  Because the cluster must support its normal amount of writes and also the import set of writes, chances are much higher that flow control may bring the cluster to a halt. 

One option is to throttle an import so that it will only load data in a slow enough way to protect the cluster from flow control but this may not be an option. 

So how do we import bulk data while "under load"?



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



If we cannot Desync multiple nodes at once, how do we handle this situation?  We desync the node that will be used as the LOCAL write node for the data import.  This will prevent it from emiting any flow control messages to the rest of the cluster. 



## Protecting other nodes: 

To protect the other nodes in the cluster from becoming overwhelmed by the added write set load, we will temporarily increase their flow control limits to an arbitrary large value: 

```
#note the current value of fc_limit: 
show global variables like "%wsrep_provider%";
#Increase to high value
SET GLOBAL wsrep_provider_options = "gcs.fc_limit = 512"; 
```

This will allow the other nodes in the cluster to tolerate a higher queue before pausing while we complete our import operations.  

After the import is done, return the cluster to its normal fc_limit values and then turn off desync on the node that was used to import the data. 



## Other notes

If the data being imported is into a new database or place that isn't being written to by the current cluster, there is less likelihood of certification conflicts.  If this is the case, use one node that is sync'd for the cluster wide "master" or "writer" and the desync'd node to load the data. 

Attempt to avoid writing to all nodes in the cluster at once. 



## Alternative option

If a cluster is under so much pressure that an import is impossible, you should initiate other plans to accomplish this task.  One way is to make a second cluster that serves as an asynchronous slave to the primary cluster, import the data there, and then failover to it.  The secondary cluster can tolerate bad behavoir such as super high fc_limits much longer than the primary.  

