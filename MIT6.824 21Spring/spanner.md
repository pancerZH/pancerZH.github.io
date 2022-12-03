---
sort: 10
---

# Spanner

This blog is based on the [paper](https://pdos.csail.mit.edu/6.824/papers/spanner.pdf) required in MIT 6.824 course, which introduced the design and implementation of Spanner, the Google globally distributed database.

## Introduction

Spanner is Google’s scalable, multi-version, globally distributed, and synchronously-replicated database. This shared database is built upon Paxos in datacenters distributed all over the world. It of course uses replication for high availability, and data could be moved across machines for load-balance.

Spanner’s main focus is managing cross-datacenter replicated data, but also implements important database features on top of the distributed-systems infrastructure. Spanner supports general-purpose transactions, and provides a SQL-based query language. It uses timestamp as the version of data, and applications can read data at old timestamps.

Spanner assigns globally-meaningful commit timestamps to transactions, even though transactions may be distributed. In addition, the serialization order satisfies external consistency (or equivalently, linearizability). This could be called cool, because some other distributed systems, like the key-value Raft map-based database does not support this feature. And in fact, it could be pretty hard for distributed system to support external consistency, because though transactions could happen in some order, and be reordered in the level of transaction level.

### Implementation

A Spanner deployment is called a *universe*. Different universes could be used for different uses, like development, test, demo and production. In a universe, there are a lot of sets of deployment of Bigtable servers, which are called *zones*. A zone has one *zonemaster* and between one hundred and several thousand *spanservers*. The former assigns data to spanservers; the latter serve data to clients.

Each spanserver would be responsible for between 100 and 1000 instances of a data structure called a *tablet*, which is similar to Bigtable’s tablet abstraction, that is:

> (key: string, timestamp: int64) → string

Spanner is more like a multi-version database than a key-value store, because tablet’s state is stored in set of B-tree-like files and a write-ahead log. 

To support replication, each spanserver implements a single Paxos state machine on top of each tablet. The Paxos implementation supports long-lived leaders with time-based leader leases.

Concurrency control is implemented by the *lock table*, which contains the state for two-phase locking: it maps ranges of keys to lock states.

At every replica that is a leader, each spanserver also implements a *transaction manager* to support distributed transactions.

On top of the bag of key-value mappings, the Spanner implementation supports a bucketing abstraction called a *directory*, which is a set of contiguous keys that share a common prefix. Directory is the unit of data placement. All data in a directory has the same replication configuration.

The application data model is layered on top of the directory-bucketed key-value mappings supported by the implementation. An application creates one or more *databases* in a universe. 

Spanner’s data model is not purely relational, in that rows must have names. More precisely, every table is required to have an ordered set of one or more primary-key columns. The primary keys form the name for a row, and each table defines a mapping from the primary-key columns to the non-primary-key columns. Here is an example:

```
----------------------
|Directory 1
|-User(1)
|---Albums{1,1)
|---Albums(1,2)
----------------------
|Directory 2
|-User(2)
|---Albums(2,1)
|---Albums(2,2)
----------------------
```

As the above structure shown, each directory could contain a user's photo data, and each album of a certain user share the same prefix across all its siblings and its parent (the user itself).

## TrueTime

The TrueTime API could be the most important foundation for Spanner. In past, the time of distributed system is indeed imprecise. In Java, we read current time by:

```java
var time = LocalDateTime.now();
```

However, the time is imprecise because the system's time is not as precise as we assume. To avoid the uncertainty, the TrueTime API would express time as an interval, instead of a specific time point. In fact, in the design, we could assume there is a precise time for the transaction happening, but we would never know it; and the only thing the system could guarantee is the boundary of the possible time range. In this way, time is expressed as *TTinterval: [earliest, latest]*. At the same time, there are two related APIs to determine a TTinterval is earlier than another or later. 

The TrueTime is maintained and checked by precise tools like GPS, and time across different machines would be synchronized. And if a machine's time is so imprecise that it exceeds the limitation, this very machine would be kicked out of zone.

## Concurrency Control

TrueTime is used to guarantee the correctness properties around concurrency control.

### Timestamp Management

Here is a table to show the types of operations that Spanner supports:

| Operation                                | Concurrency Control | Replica Required                   |
| :--------------------------------------- | ------------------- | ---------------------------------- |
| Read-Write Transaction                   | pessimistic         | leader                             |
| Read-Only Transaction                    | lock-free           | leader for timestamp; any for read |
| Snapshot Read, client-provided timestamp | lock-free           | any                                |
| Snapshot Read, client-provided bound     | lock-free           | any                                |

Standalone writes are implemented as read-write transactions; non-snapshot standalone reads are implemented as read-only transactions.

A read-only transaction must be predeclared as not having any writes; it is not simply a read-write transaction without any writes. Reads in a read-only transaction execute at a system-chosen timestamp without locking, so that incoming writes are not blocked. A snapshot read is a read in the past that executes without locking. A client can either specify a timestamp for a snapshot read, or provide an upper bound on the desired timestamp’s staleness and let Spanner choose a timestamp. For both of them, commit is inevitable once a timestamp has been chosen, unless the data at that timestamp has been garbage collected.

Spanner’s Paxos implementation uses timed leases to make leadership long-lived.

Transactional reads and writes use two-phase locking. As a result, they can be assigned timestamps at any time when all locks have been acquired, but before any locks have been released. Just like I have discussed in last blog. Within each Paxos group, Spanner assigns timestamps to Paxos writes in monotonically increasing order, even across leaders.

The key for Spanner to implement external consistency is the TTinterval. Only after the TrueTime API believes the current time is after the write timestamp (expressed as TTinterval), could the clients see the data committed by the write transaction. In this way, we could also see the importance of preciseness of clock, because the smaller we could make the time range be, the more transactions could be finished in a time unit. This is called **Commit Wait**.

A read-only transaction executes in two phases: assign a timestamp *s_read*, and then execute the transaction’s reads as snapshot reads at the timestamp *s_read*.

### Details

Here are more details.

#### Read-Write Transactions

Writes that occur in a transaction are buffered at the client until commit. As a result, reads in a transaction do not see the effects of the transaction’s writes. Reads within read-write transactions use wound-wait to avoid deadlocks.

A non-coordinator-participant leader first acquires write locks. It then chooses a prepare timestamp that must be larger than any timestamps it has assigned to previous transactions. The coordinator leader also first acquires write locks, but skips the prepare phase. It chooses a timestamp for the entire transaction after hearing from all other participant leaders. The commit timestamp s must be greater or equal to all prepare timestamps.

Before allowing any coordinator replica to apply the commit record, the coordinator leader waits until the current time is truly after the TTinterval, so as to obey the commit-wait rule.

#### Read-Only Transactions

Spanner requires a *scope* expression for every read-only transaction, which is an expression that summarizes the keys that will be read by the entire transaction. Spanner automatically infers the scope for standalone queries.

If the scope’s values are served by a single Paxos group, then the client issues the read-only transaction to that group’s leader. If the scope’s values are served by multiple Paxos groups, the client would have its read operations to execute at the highest boundary of the range of current TTinterval.

## Summary

To summarize, Spanner combines and extends on ideas from two research communities: from the database community, a familiar, easy-to-use, semi-relational interface, transactions, and an SQL-based query language.