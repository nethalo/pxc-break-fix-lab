# Backups Simulation Lab

Before getting started, ensure there are some RW threads running against the cluster. 



## Simulating production load 

Our lab environment is not configured in such a way to simulate the complex and demanding load that a cluster node might experience in a production environment.  To help us simulate a node that is being pushed to "near its limits" we'll lower the flow control threshold on the node we're going to use for backups: 

```
SET GLOBAL wsrep_provider_options = "gcs.fc_limit = 1";
```



At this point, using PMM or the status variables, ensure there is a very slow increment of flow control messages being sent.  If not slowly increase the threads of the load until it starts to happen but very slowly: 

```show global status like '%flow%sent%'\G```



## Backup: 

Now lets begin a backup on the node: 

```mysqldump --single-transaction --all-databases```



Again observe the flow control.  The node will be emitting FC a lot and it will cripple the entire cluster.  You will be able to see the tps drop off from the sysbench threads.  Cancel the backup and lets do it correctly now. 



## Desync the node

First, make sure we aren't sending writes to the node we're going to use as a backup.  

If we're thinking this backup is going to take a very long time and we have a good amount of disk space, now is the time that we'd tell mysql to allow gcache purging to stop, in anticipation that we'll need a larger than normal IST after we're done.  To do this, pick your DONOR node for the IST and: 

```SET GLOBAL wsrep_provider_options = "gcache.freeze_purge_at_seqno=now";```



Now, back to our backup node.  Lets make sure it won't impact the cluster by desync: 

```SET GLOBAL wsrep_desync='ON'; ```



Begin our backup the same as before: 

```mysqldump --single-transaction --all-databases```



Observe the cluster for FC and you'll see that the node carries on the backup without impacting the cluster now.



## Cleanup

Be sure to set the default fc_limit, turn desync=OFF and if you set the ```SET GLOBAL wsrep_provider_options = "gcache.freeze_purge_at_seqno=now";``` option you will want to return it to the default value of -1.

