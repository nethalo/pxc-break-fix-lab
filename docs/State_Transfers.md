# State Transfers

Galera has two types of state transfers that allow syncing data to nodes when needed: 

- Incremental (IST) 
- Full (SST). 

Incremental is used when a node has been out of a cluster for some time, and once it rejoins the other nodes has the missing write sets still in Galera cache. 

Full SST is helpful if incremental is not possible, especially when a new node is added to the cluster. SST automatically provisions the node with fresh data taken as a snapshot from one of the running nodes (donor).

## [SST - State Snapshot Transfer](SST-State_Transfers.md)