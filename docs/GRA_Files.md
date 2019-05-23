# GRA Files

Those files correspond to a replication failure. That means the slave thread was not able to apply one transaction. For each of those file, a corresponding warning or error message is present in the mysql error log file.

```
2019-05-17T13:56:35.525364Z 8 [ERROR] Slave SQL: Error 'Incorrect table definition; there can be only one auto column and it must be defined as a key' on query. Default database: 'percona'. Query: 'CREATE TABLE sbtest100 (

  id int(10) unsigned NOT NULL AUTO_INCREMENT,

  k int(10) unsigned NOT NULL DEFAULT '0',

  c char(120) NOT NULL DEFAULT '',

  pad char(60) NOT NULL DEFAULT '',

  KEY k_1 (k)

) ENGINE=InnoDB', Error_code: 1075
2019-05-17T13:56:35.525848Z 8 [Warning] WSREP: RBR event 1 Query apply warning: 1, 36304
2019-05-17T13:56:35.571531Z 8 [Warning] WSREP: Ignoring error for TO isolated action: source: b7c194e5-7846-11e9-b6c4-2afd2e47cbf0 version: 4 local: 0 state: APPLYING flags: 65 conn_id: 19 trx_id: -1 seqnos (l: 36935, g: 36304, s: 36303, d: 36303, ts: 60090963212074)
```

Files are stored on the datadir:

```
[root@pxc2 mysql]# strings /var/lib/mysql/GRA_8_36304.log
percona
CREATE TABLE sbtest100 (
  id int(10) unsigned NOT NULL AUTO_INCREMENT,
  k int(10) unsigned NOT NULL DEFAULT '0',
  c char(120) NOT NULL DEFAULT '',
  pad char(60) NOT NULL DEFAULT '',
  KEY k_1 (k)
) ENGINE=InnoDB
```

<https://www.percona.com/blog/2012/12/19/percona-xtradb-cluster-pxc-what-about-gra_-log-files/>

## Primary Keys 

Let's create (properly) the table:

```
CREATE TABLE `sbtest100` (
  `id` int(10) unsigned NOT NULL,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  KEY `k_1` (`k`)
) ENGINE=InnoDB;
```

And now let's populate it with 1 single big TRX:

`INSERT INTO sbtest100 SELECT * from sbtest1;`

The strict mode will prevent to do such a crazy query. Let's override it:

```
set global pxc_strict_mode='DISABLED'; INSERT INTO sbtest100 SELECT * from sbtest1; set global pxc_strict_mode='ENFORCING';
```

## 