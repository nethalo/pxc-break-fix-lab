# PXC Break/Fix Tutorial

In this tutorial, we'll walk through over several scenarios where things can get complex with your Percona XtraDB Cluster (PXC)/Galera installation. We will break things in these scenarios and then you'll have a chance to diagnose and fix them. Learn some of the most complex troubleshooting techniques. 

This tutorial assume that you are somehow familar with the Percona XtraDB Cluster.

## Galera focus points

When talking about Galera, the main key points are the following:

- State Transfer (Syncing the nodes): Boostraping, Node joining/leaving. Data replication.
- Network connection (Maintain Cluster as a whole): Quorum, Primary Component, Cluster Stability.
- Data consistency (Guarantee data is the same): Certification, Flow Control.

Besides the omnipresent topic on databases, and our reason to live: **Performance.**

# Environment

All the scenarios will take place on a basic 3-node PXC Cluster

<img src="cluster.png" alt="cluster" width="400px"/>

Creation of the VMs and the cluster are available at the Environment link: [here](docs/environment.md)

# Cases

## State Transfer

### [Ports](docs/Ports.md)

### [State Transfers](docs/State_Transfers.md)

### [Group communication](docs/Group_communication.md)

## Quorum

### [Quorum](docs/Quorum.md)

### [WAN](docs/WAN.md)

### [Partition Handling (Network Hiccup)](docs/Partition_Handling.md)

### [GTID](docs/GTID.md)

## Data Consistency

### [Flow control](docs/Flow_control.MD)

### [PXC Strict Mode](docs/PXC_Strict_Mode.md)

### [GRA Files](docs/GRA_Files.md)

### [Auto increment](docs/Auto_increment.md)

### [Data Consistency](docs/Data_Consistency.md)

### [Multi Master](docs/Multi_Master.md)

### [Schema Requirements](docs/Schema_Requirements.md)

### [Multi thread](docs/Multi_thread.md)

## 

