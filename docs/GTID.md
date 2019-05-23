# GTID

Galera has it’s [own Global Transaction ID](http://galeracluster.com/documentation-webpages/architecture.html#global-transaction-id), which has existed since MySQL 5.5, and is independent from MySQL’s GTID feature introduced in MySQL 5.6.

To keep the state identical on all nodes, the [wsrep API](http://galeracluster.com/documentation-webpages/glossary.html#term-wsrep-api) uses global transaction IDs (GTID), which are used to both:

> - Identify the state change
> - Identify the state itself by the ID of the last state change

The GTID consists of:

> - A state UUID, which uniquely identifies the state and the sequence of changes it undergoes
> - An ordinal sequence number (seqno, a 64-bit signed integer) to denote the position of the change in the sequence

The Global Transaction ID allows you to compare the application state and establish the order of state changes. You can use it to determine whether or not a change was applied and whether the change is applicable at all to a given state.

## 