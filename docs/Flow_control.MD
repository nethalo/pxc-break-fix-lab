# Flow control

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

## Key variables

- **gcs.fc_limit:** the threshold.  if the *wsrep_local_recv_queue* exceeds this size on a given node, a pausing flow control message will be sent. 
- **gcs.fc_master_slave**: should galera modify dinamycally the limit or not
- **gcs.fc_factor:** once flow control kicks in, when it should be released. The factor is a number between 0.0 and 1.0, which is multiplied by the current *fc_limit*

## Monitor FC

On 5.7: show status like '**wsrep_flow_control_status**'; This boolean status variable tells the user if the node is in FLOW_CONTROL or not.

the **wsrep_flow_control_sent/recv** counter can be used to track FLOW_CONTROL status. 

Better check them all: `show status like 'w%flow%';`

## Simulate FC

- Disable PXC STRICT MODE: `set global pxc_strict_mode='DISABLED';`

- Lock a table: `use percona; lock tables sbtest9 write;`

- You will see messages like this in the error log:

  ```
   2019-05-17T13:47:47.773417Z 8 [Note] WSREP: MDL conflict db=percona table=sbtest9 ticket=MDL_SHARED_NO_READ_WRITE solved by abort
  ```

- Remove lock: `unlock tables;`

- Set back strict mode: `set global pxc_strict_mode='ENFORCING';`