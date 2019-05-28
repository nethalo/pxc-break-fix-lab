# Sysbench

To simulate traffic to the cluster, we will use sybench from the app-pxc machine. 

## Install sysbench

Go to the app server (vagrant ssh app-pxc) and install the percona repo && the sysbench package:

- yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
- yum install -y sysbench.x86_64

## Prepare sysbench tables

Before running any sysbench command, create the database "percona" in the cluster. Go to any pxc node and run: 

```
create database percona;
```

## Create MySQL User for sysbench to use

We just need a "app" like user. We will call it "percona"

```
CREATE USER 'percona'@'%' IDENTIFIED BY 'perconarocks'; 
GRANT all privileges on *.* to 'percona'@'%'; 
FLUSH PRIVILEGES;
```

### Create the tables

```
sysbench /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua --db-driver=mysql \
--mysql-host=mysql1-T21 \
--mysql-user=percona --mysql-password=perconarocks \
--mysql-db=percona --mysql-table-engine=innodb \
--threads=2 --events=100000000 \
--oltp-tables-count=10 --oltp-table-size=10000 \
prepare
```

### And now, run it to send traffic

```
sysbench /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua --db-driver=mysql \
--mysql-host=mysql1-T21 \
--mysql-user=percona --mysql-password=perconarocks \
--mysql-db=percona --mysql-table-engine=innodb \
--threads=2 --events=100000000 \
--oltp-tables-count=10 --oltp-table-size=10000 \
--time=1000000000 --report-interval=1 \
run
```

## 

