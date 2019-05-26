# SST Simulations Lab

We will simulate several different situations in which we break the lab, attempt a SST and then diagnose the issue, correct it, and successfully perform the SST.

To begin, lets add some configuration to our cluster: 

#### START UP TASKS (ON JOINER): 

```
echo "wsrep-sst-donor=\"pxc2\"" >> /etc/my.cnf
```



## Bad Authentication

#### Break It

- On the Donor Node: 

```
  sed -i 's/root:cocacola/badguy:pepsi/g' /etc/my.cnf
  service mysql stop 
  service mysql start
```

#### Attempt SST

- On the Joiner Node: 

```
  service mysql stop 
  rm -rf /var/lib/mysql/*
  service mysql start
```

```  less /var/log/mysqld.log```

#### FIX IT 

```
  sed -i 's/badguy:pepsi/root:cocacola/g' /etc/my.cnf
  service mysql stop 
  service mysql start
```



## Blocked by security

#### Break It

- On the Donor Node: 

```
 setenforce 1
```

#### Attempt SST

- On the Joiner Node: 

```
  service mysql stop 
  rm -rf /var/lib/mysql/*
  service mysql start
```

```  less /var/log/mysqld.log```

#### FIX IT 

```
 setenforce 0 
```



## Blocked by ports

#### Break It

- On the Joiner Node: 

```
  iptables -A INPUT -p tcp --destination-port 4444 -j DROP
```

#### Attempt SST

- On the Joiner Node: 

```
  service mysql stop 
  rm -rf /var/lib/mysql/*
  service mysql start
```

```  less /var/log/mysqld.log```

#### FIX IT 

```
iptables -D INPUT -p tcp --destination-port 4444 -j DROP
```



## Use a different SST method

#### Use It

- On the Joiner Node: 

```
 service mysql stop 
 echo "wsrep_sst_method = "rsync"" >> /etc/my.cnf
```

#### Attempt SST

- On the Joiner Node: 

```
  rm -rf /var/lib/mysql/*
  service mysql start
```

```  less /var/log/mysqld.log```

#### 

## Use a different SST method

#### Use It

- On the Joiner Node: 

```
 service mysql stop 
 echo "wsrep_sst_method = "rsync"" >> /etc/my.cnf
```

#### Attempt SST

- On the Joiner Node: 

```
  rm -rf /var/lib/mysql/*
  service mysql start
```

```  less /var/log/mysqld.log```

