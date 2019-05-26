# Import Simulation Lab

Before getting started, ensure there are some RW threads running against the cluster.  



## Generate an import file

First lets generate a file to simulate a large data dump: 

```
mysqldump --single-transaction sbtest > import_simulation.sql
```



## Simulating production load

Our lab environment is not configured in such a way to simulate the  complex and demanding load that a cluster node might experience in a  production environment.  To help us simulate a node that is being pushed  to "near its limits" we'll lower the flow control threshold on the node(s)  we're going to use for testing:

```
SET GLOBAL wsrep_provider_options = "gcs.fc_limit = 1";
```

At this point, using PMM or the status variables, ensure there is a  very slow increment of flow control messages being sent.  If not slowly  increase the threads of the load until it starts to happen but very  slowly:

```
show global status like '%flow%sent%'\G
```



## Initial attempts to import

First attempts to just "import this file" will fail: 
```[root@pxc3 backups]# mysql import_simulation < import_simulation.sql
ERROR 1105 (HY000) at line 48: Percona-XtraDB-Cluster prohibits use of LOCK TABLE/FLUSH TABLE <table> WITH READ LOCK/FOR EXPORT with pxc_strict_mode = ENFORCING```
```

Lets bypass this for now: 

```mysql -e "SET GLOBAL pxc_strict_mode='PERMISSIVE';"``` 

And finally simulate an import: 

```
mysql -e "CREATE DATABASE import_simulation;"
mysql import_simulation < import_simulation.sql
```



You should witness the cluster struggling as FC messages begin being emitted: 

```show global status where variable_name in ('wsrep_flow_control_sent','wsrep_flow_control_recv');```



Check a few nodes and you'll see different nodes are all emitting FC messages: 

```mysql> show global status where variable_name in ('wsrep_flow_control_sent','wsrep_flow_control_recv');
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| wsrep_flow_control_sent | 39    |
| wsrep_flow_control_recv | 296   |
+-------------------------+-------+
```



## First try - desync - just like a backup right?: 

Just like in the backup phase, lets desync the node we're pushing the import on: 

```SET GLOBAL wsrep_desync='ON'; ```

House keeping to try again: 

```mysql -e "DROP DATABASE import_simulation; CREATE DATABASE import_simulation;"```

Start the import again: 

```mysql import_simulation < import_simulation.sql```

Observe the FC results. Multiple nodes might now be complaining as the load is too much for all the writes.  So how can we handle this? 



## Increase the fc_limit on all the nodes of the "other" nodes

On the nodes not being used for the import: 

```SET GLOBAL wsrep_provider_options = "gcs.fc_limit = 512";```

Observe the FC conditions of the cluster. 



## Cleanup

Be sure to set the default fc_limit, turn desync=OFF and if you set pxc_strict_mode back to default. 

