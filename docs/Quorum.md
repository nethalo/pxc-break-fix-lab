# Quorum

A majority (> 50%) of nodes. In the event of a network partition, only the cluster partition that retains a quorum (if any) will remain Primary by default.

The size of the cluster is used to determine the required votes to achieve [quorum](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/glossary.html#term-quorum).

A quorum vote is done when a node or nodes are suspected to no longer be part of the cluster (they do not respond). 

This no response timeout is the [`evs.suspect_timeout`](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-provider-index.html#evs.suspect_timeout) setting in the [`wsrep_provider_options`](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/wsrep-system-index.html#wsrep_provider_options) (default 5 sec), and when a node goes down ungracefully, write operations will be blocked on the cluster for slightly longer than that timeout.

Cluster membership is determined simply by which nodes are connected to the rest of the cluster; there is no configuration setting explicitly defining the list of all possible cluster nodes. Therefore, every time a node joins the cluster, the total size of the cluster is increased and when a node leaves (gracefully) the size is decreased

## Primary Component 

PXC has ability auto-bootstrap after all-nodes-down situation. Previously we had to bootsrap a node manually. 

The way it works is if all the nodes are able to resrtart and communicate each other the cluster will make automatic decision to recovery Primary state as a whole. 

Possible scenarios are:

- All nodes went down hard — that is; a kill -9, kernel panic, server power failure, or similar event
- All nodes from the last PRIMARY component are restarted and are able to see each other again.

Simulation:

```
[root@pxc1 ~]# killall5 -9 mysqld
Shared connection to 127.0.0.1 closed.`

`root@pxc2 ~]# killall5 -9 mysqld`
`Shared connection to 127.0.0.1 closed.`

`[root@pxc3 ~]# killall5 -9 mysqld`
`Shared connection to 127.0.0.1 closed.
```



```
[root@pxc1 mysql]# cat gvwstate.dat
my_uuid: 5d6f51af-7cb7-11e9-93b5-472bfe14ee56
#vwbeg
view_id: 3 5d6f51af-7cb7-11e9-93b5-472bfe14ee56 12
bootstrap: 0
member: 5d6f51af-7cb7-11e9-93b5-472bfe14ee56 0
member: 715bd53e-7c80-11e9-999c-ee1e835824a9 0
member: a5109f43-7c80-11e9-b10d-8ee17b77392d 0
#vwend
[root@pxc1 mysql]# cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    bcf6669e-6db5-11e9-9815-0b05174fd831
seqno:   -1
safe_to_bootstrap: 0
```

```
[root@pxc2 ~]# cat /var/lib/mysql/gvwstate.dat
my_uuid: 715bd53e-7c80-11e9-999c-ee1e835824a9
#vwbeg
view_id: 3 715bd53e-7c80-11e9-999c-ee1e835824a9 13
bootstrap: 0
member: 715bd53e-7c80-11e9-999c-ee1e835824a9 0
member: a5109f43-7c80-11e9-b10d-8ee17b77392d 0
#vwend
[root@pxc2 ~]# cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    bcf6669e-6db5-11e9-9815-0b05174fd831
seqno:   -1
safe_to_bootstrap: 0
    [root@pxc2 ~]#
```

```
[root@pxc3 ~]# cat /var/lib/mysql/gvwstate.dat
my_uuid: a5109f43-7c80-11e9-b10d-8ee17b77392d
#vwbeg
view_id: 3 715bd53e-7c80-11e9-999c-ee1e835824a9 13
bootstrap: 0
member: 715bd53e-7c80-11e9-999c-ee1e835824a9 0
member: a5109f43-7c80-11e9-b10d-8ee17b77392d 0
#vwend
[root@pxc3 ~]# cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    bcf6669e-6db5-11e9-9815-0b05174fd831
seqno:   -1
safe_to_bootstrap: 0
    [root@pxc3 ~]#
```

This file will not exist on a node if it was shutdown cleanly, only if the mysqld was uncleanly terminated. This file should exist and be the same on all the nodes for the auto-recovery to work.

As you can see cluster sets safe_to_bootstrap: 0 . 

Start one of the nodes:

```
    [root@pxc1 mysql]# service mysql start
Redirecting to /bin/systemctl start mysql.service

2019-05-22T18:38:00.896804Z 0 [Note] WSREP: Restoring primary-component from disk successful
2019-05-22T18:38:00.897002Z 0 [Note] WSREP: GMCast version 0
2019-05-22T18:38:00.897277Z 0 [Note] WSREP: (5d6f51af, 'tcp://0.0.0.0:4567') listening at tcp://0.0.0.0:4567
2019-05-22T18:38:00.897371Z 0 [Note] WSREP: (5d6f51af, 'tcp://0.0.0.0:4567') multicast: , ttl: 1
2019-05-22T18:38:00.897762Z 0 [Note] WSREP: EVS version 0
2019-05-22T18:38:00.897873Z 0 [Note] WSREP: gcomm: connecting to group 'dani-cluster', peer '192.168.80.10:,192.168.80.20:,192.168.80.30:'
2019-05-22T18:38:00.898647Z 0 [Note] WSREP: (5d6f51af, 'tcp://0.0.0.0:4567') connection established to 5d6f51af tcp://192.168.80.10:4567
2019-05-22T18:38:00.898678Z 0 [Warning] WSREP: (5d6f51af, 'tcp://0.0.0.0:4567') address 'tcp://192.168.80.10:4567' points to own listening address, blacklisting
2019-05-22T18:38:03.970438Z 0 [Note] WSREP: announce period timed out (pc.announce_timeout)
2019-05-22T18:38:03.970845Z 0 [Warning] WSREP: no nodes coming from prim view, prim not possible
2019-05-22T18:38:03.970891Z 0 [Note] WSREP: Current view of cluster as seen by this node
view (view_id(NON_PRIM,5d6f51af,14)
memb {
        5d6f51af,0
        }
joined {
        }
left {
        }
partitioned {
        }
)
2019-05-22T18:38:03.970929Z 0 [Note] WSREP: gcomm: connected
2019-05-22T18:38:03.971403Z 0 [Note] WSREP: Shifting CLOSED -> OPEN (TO: 0)
2019-05-22T18:38:03.971531Z 0 [Note] WSREP: Waiting for SST/IST to complete.
2019-05-22T18:38:03.972735Z 0 [Note] WSREP: New COMPONENT: primary = no, bootstrap = no, my_idx = 0, memb_num = 1
2019-05-22T18:38:03.972784Z 0 [Note] WSREP: Flow-control interval: [100, 100]
2019-05-22T18:38:03.972791Z 0 [Note] WSREP: Trying to continue unpaused monitor
2019-05-22T18:38:03.972798Z 0 [Note] WSREP: Received NON-PRIMARY.
2019-05-22T18:38:03.973161Z 1 [Note] WSREP: New cluster view: global state: bcf6669e-6db5-11e9-9815-0b05174fd831:866061, view# -1: non-Primary, number of nodes: 1, my index: 0, protocol version -1
2019-05-22T18:38:03.973190Z 1 [Note] WSREP: Setting wsrep_ready to false
2019-05-22T18:38:03.973210Z 1 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
2019-05-22T18:38:04.471066Z 0 [Warning] WSREP: last inactive check more than PT1.5S (3*evs.inactive_check_period) ago (PT3.5733S), skipping check

```

At this point pxc1 is awaiting other nodes to come up. 

```
[root@pxc2 ~]# service mysql start
Redirecting to /bin/systemctl start mysql.service

2019-05-22T18:40:52.744308Z 0 [Note] WSREP: Restoring primary-component from disk successful
2019-05-22T18:40:52.745069Z 0 [Note] WSREP: GMCast version 0
2019-05-22T18:40:52.746366Z 0 [Note] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') listening at tcp://0.0.0.0:4567
2019-05-22T18:40:52.746489Z 0 [Note] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') multicast: , ttl: 1
2019-05-22T18:40:52.747349Z 0 [Note] WSREP: EVS version 0
2019-05-22T18:40:52.747535Z 0 [Note] WSREP: gcomm: connecting to group 'dani-cluster', peer '192.168.80.10:,192.168.80.20:,192.168.80.30:'
2019-05-22T18:40:52.749328Z 0 [Note] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') connection established to 715bd53e tcp://192.168.80.20:4567
2019-05-22T18:40:52.749374Z 0 [Warning] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') address 'tcp://192.168.80.20:4567' points to own listening address, blacklisting
2019-05-22T18:40:52.753520Z 0 [Note] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') connection established to 5d6f51af tcp://192.168.80.10:4567
2019-05-22T18:40:52.753627Z 0 [Note] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') turning message relay requesting on, nonlive peers:
2019-05-22T18:40:53.264377Z 0 [Note] WSREP: declaring 5d6f51af at tcp://192.168.80.10:4567 stable
2019-05-22T18:40:53.268559Z 0 [Warning] WSREP: no nodes coming from prim view, prim not possible
2019-05-22T18:40:53.268660Z 0 [Note] WSREP: Current view of cluster as seen by this node
view (view_id(NON_PRIM,5d6f51af,15)
memb {
        5d6f51af,0
        715bd53e,0
        }
joined {
        }
left {
        }
partitioned {
        }
)
2019-05-22T18:40:53.759268Z 0 [Note] WSREP: gcomm: connected
2019-05-22T18:40:53.759367Z 0 [Note] WSREP: Shifting CLOSED -> OPEN (TO: 0)
2019-05-22T18:40:53.759539Z 0 [Note] WSREP: Waiting for SST/IST to complete.
2019-05-22T18:40:53.760686Z 0 [Note] WSREP: New COMPONENT: primary = no, bootstrap = no, my_idx = 1, memb_num = 2
2019-05-22T18:40:53.760759Z 0 [Note] WSREP: Flow-control interval: [141, 141]
2019-05-22T18:40:53.760767Z 0 [Note] WSREP: Trying to continue unpaused monitor
2019-05-22T18:40:53.760776Z 0 [Note] WSREP: Received NON-PRIMARY.
2019-05-22T18:40:53.761202Z 2 [Note] WSREP: New cluster view: global state: bcf6669e-6db5-11e9-9815-0b05174fd831:868357, view# -1: non-Primary, number of nodes: 2, my index: 1, protocol version -1
2019-05-22T18:40:53.761315Z 2 [Note] WSREP: Setting wsrep_ready to false
2019-05-22T18:40:53.761378Z 2 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
2019-05-22T18:40:55.763932Z 0 [Note] WSREP: (715bd53e, 'tcp://0.0.0.0:4567') turning message relay requesting off
```



pxc3 also fails to start with same output. 

```
[root@pxc3 ~]# clustercheck
HTTP/1.1 503 Service Unavailable
Content-Type: text/plain
Connection: close
Content-Length: 57

Percona XtraDB Cluster Node is not synced or non-PRIM.
```



None of the nodes have sequence number.

seqno:   -1

We'll try to find most advanced node in cluster to bootstrap. 

```
[root@pxc1 mysql]# mysqld_safe --wsrep-recover
2019-05-22T20:00:40.283638Z mysqld_safe Logging to '/var/log/mysqld.log'.
2019-05-22T20:00:40.288320Z mysqld_safe Logging to '/var/log/mysqld.log'.
2019-05-22T20:00:40.329758Z mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
2019-05-22T20:00:40.344083Z mysqld_safe Skipping wsrep-recover for bcf6669e-6db5-11e9-9815-0b05174fd831:866061 pair
2019-05-22T20:00:40.347172Z mysqld_safe Assigning bcf6669e-6db5-11e9-9815-0b05174fd831:866061 to wsrep_start_position
2019-05-22T20:00:43.346962Z mysqld_safe mysqld from pid file /var/run/mysqld/mysqld.pid ended

[root@pxc2 ~]# mysqld_safe --wsrep-recover
2019-05-22T20:02:02.790197Z mysqld_safe Logging to '/var/log/mysqld.log'.
2019-05-22T20:02:02.794042Z mysqld_safe Logging to '/var/log/mysqld.log'.
2019-05-22T20:02:02.833923Z mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
2019-05-22T20:02:02.873733Z mysqld_safe WSREP: Running position recovery with --log_error='/var/lib/mysql/wsrep_recovery.fC3Gxj' --pid-file='/var/lib/mysql/pxc2-recover.pid'
2019-05-22T20:02:06.917687Z mysqld_safe WSREP: Recovered position 00000000-0000-0000-0000-000000000000:-1
2019-05-22T20:02:10.069667Z mysqld_safe mysqld from pid file /var/run/mysqld/mysqld.pid ended

[root@pxc3 ~]# mysqld_safe --wsrep-recover
2019-05-22T20:01:55.464404Z mysqld_safe Logging to '/var/log/mysqld.log'.
2019-05-22T20:01:55.468699Z mysqld_safe Logging to '/var/log/mysqld.log'.
2019-05-22T20:01:55.512446Z mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
2019-05-22T20:01:55.529229Z mysqld_safe Skipping wsrep-recover for bcf6669e-6db5-11e9-9815-0b05174fd831:868358 pair
2019-05-22T20:01:55.531752Z mysqld_safe Assigning bcf6669e-6db5-11e9-9815-0b05174fd831:868358 to wsrep_start_position
2019-05-22T20:01:59.295247Z mysqld_safe mysqld from pid file /var/run/mysqld/mysqld.pid ended
```

In this case  pxc3 has the latest seqno hence we bootstrap that node. 



```
[root@pxc3 mysql]# systemctl start mysql@bootstrap

pxc3:mysql>show global status where variable_name IN ('wsrep_local_state','wsrep_local_state_comment','wsrep_local_commits','wsrep_received','wsrep_cluster_size','wsrep_cluster_status','wsrep_connected','wsrep_ready');
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| wsrep_received            | 2       |
| wsrep_local_commits       | 0       |
| wsrep_local_state         | 4       |
| wsrep_local_state_comment | Synced  |
| wsrep_cluster_size        | 1       |
| wsrep_cluster_status      | Primary |
| wsrep_connected           | ON      |
| wsrep_ready               | ON      |
+---------------------------+---------+
8 rows in set (0.00 sec)

2019-05-22T20:16:50.958522Z 0 [Note] WSREP: Shifting JOINED -> SYNCED (TO: 868358)
2019-05-22T20:16:50.958637Z 2 [Note] WSREP: REPL Protocols: 9 (4, 2)
2019-05-22T20:16:50.958657Z 2 [Note] WSREP: New cluster view: global state: bcf6669e-6db5-11e9-9815-0b05174fd831:868358, view# 1: Primary, number of nodes: 1, my index: 0, protocol version 3
```

 In this case node 2 and 1 will join via SST

```
2019-05-22T20:20:31.678543Z 2 [Note] WSREP: GCache history reset: bcf6669e-6db5-11e9-9815-0b05174fd831:0 -> bcf6669e-6db5-11e9-9815-0b05174fd831:868358
        2019-05-22T20:20:32.732277Z WSREP_SST: [INFO] Proceeding with SST.........
        2019-05-22T20:20:32.803369Z WSREP_SST: [INFO] ............Waiting for SST streaming to complete!
2019-05-22T20:20:33.277798Z 0 [Note] WSREP: (5d6f51af, 'tcp://0.0.0.0:4567') turning message relay requesting off

mysql> show global status where variable_name IN ('wsrep_local_state','wsrep_local_state_comment','wsrep_local_commits','wsrep_received','wsrep_cluster_size','wsrep_cluster_status','wsrep_connected','wsrep_ready');
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| wsrep_received            | 7       |
| wsrep_local_commits       | 0       |
| wsrep_local_state         | 4       |
| wsrep_local_state_comment | Synced  |
| wsrep_cluster_size        | 3       |
| wsrep_cluster_status      | Primary |
| wsrep_connected           | ON      |
| wsrep_ready               | ON      |
+---------------------------+---------+
8 rows in set (0.00 sec)
```

