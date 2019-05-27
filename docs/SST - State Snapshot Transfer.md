# SST - State Snapshot Transfer

State Snapshot Transfer is a full data copy from one node (donor) to the joining node (joiner)

- Can happen when a new node joins the cluster
- Can happen if IST is not possible (JOINER is too far behind)
- Can be forced if we remove meta data related to determining the state of the JOINER (grastate.dat file)



## Requirements

- SST utililzes port 4444 so it must be open between the DONOR and JOINER nodes. For a full list of ports, go to [PORTS](Ports.md)



## Key variables

- **wsrep_sst_method**: Determines the method of the backup/restore operations to be executed on the DONOR and JOINER
- **wsrep_sst_donor:** The preference for DONOR node, under normal circumstances the cluster will elect a DONOR, but with this, you can specify.
- **wsrep_sst_auth**:  The credentials used for the backup - must be declared on the DONOR node!  (And the JOINER if using mysqldump)

# SST Methods

There are three options* for a SST operation in PXC 5.7

## Logical 

### mysqldump - `wsrep_sst_method = "mysqldump"`

- Slowest method
- Blocks donor node
- Allows JOINER to remain operational during SST
- Has benefits of a logical backup - like resetting the ib_ file sizes
- Needs all privs for db access

## Physical

### rsync - `wsrep_sst_method = "rsync"`

- Fastest method
- Blocks donor node
- No DB credentials required

### xtrabackup-v2 - `wsrep_sst_method = "xtrabackup-v2"`

- Default SST method in PXC 5.7 
- Fast
- Fast meta data lock on donor node
- Needs DB creds on DONOR only 
- Resilent to situations where data files aren't in declared datadir
  - `CREATE TABLE test ... DATA DIRECTORY = '/alternative/directory';`
- Has many configuration options (such as network transfer method and encryption options) 

# Force SST 

- Remove the datadir: `rm -rf <datadir>/*`
- Remove the metadata used to calculate IST vs SST: `rm -f <datadir>/grastate.dat`

# [SST Simulations Lab](SST Simulations Lab.md)