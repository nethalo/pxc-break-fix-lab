# PXC Strict Mode

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