---
sort: 2
---

# GFS: The Google File System

This part is all based on the paper [GFS](https://pdos.csail.mit.edu/6.824/papers/gfs.pdf), which is absolutely worth carefully reading.

## What is GFS

GFS is a scalable distributed file system for large distributed data-intensive applications. With the same goals with previous distributed file system, GFS focus on:

1. **Component failures are norm rather than the exception**: do not try to avoid failures, but accept them and solve them
2. **Files are huge**: I/O must be treated carefully
3. **Most files are mutated by appending new data rather than overwriting**: appending is the focus of performance optimization and atomicity guarantees
4. **Co-designing the applications and the file system API**: increase flexibility

GFS is not built for experiments, but based on very specific goals. It does not pay equal attentions to different kinds of file operations, but according to Google's scenario, focus on large streaming reads. Some designs may not meet requirements out of Google's scope, but it does not mean GFS has a bad design.

## Structures about GFS

1. **Single master**: GFS has one single master, which maintains file metadata, and is obviously a centric component. This single master simplifies the whole system, but GFS's developers deal with it carefully. Only control flow is passed to and from this master, while data flow, on the other hand, is not. This design decreases the workload of I/O and network bandwidth, and it has been proved a delightful design.

2. **Multi chunkservers**: They are used to store files and their replicas. Files are stored in chunks, which are fixed-size buckets, or blocks. GFS treats it carefully when writing contents into these buckets,  because it has to care about the both sides of the file, they may be stored in different buckets, and these details must be recorded and tracked for reading.
3. **Multi clients**: Clients would simply send requests to the master, and the master would check its records and find out which chunkservers contain the file each client wants and their locations. These information would be sent back to clients, and clients would communicate with chunkservers directly for the data.

GFS also maintains necessary logs to track each operation, and creates checkpoints based on these logs. They are powerful to be used for recovering the file system by replaying them.

After a sequence of successful mutations, the mutated file region is guaranteed to be defined and contain the data written by the last mutation. GFS achieves this by using:

1. the same order applied of mutations
2. chunk version number

GFS also uses checksums to verify the validity of records.

## How GFS runs

### Lease

Cause each operation would finally be applied to all replicas, GFS uses leases to maintain the consistency across replicas. This mechanism is designed to minimize management overhead at the master. There are some steps:

1. The client asks the master for the location of the chunkserver holding the lease of the chunk and locations of other replicas
2. The master replies
3. The client pushes the data to all replicas
4. Once receiving all confirmed, the client sends a write request to the primary. The primary applied the mutations
5. The primary forwards the request to all secondary replicas
6. The secondaries all reply to the primary 
7. The primary replies to the client

### Data Flow

GFS decouples the flow of data from the flow of control to use the network efficiently.

- Control Flow: Client -> Primary -> Secondary
- Data Flow: between chunkservers (and clients)

The key is that each machine forwards the data to the closest machine. Once a machine receives some data, it starts forwarding immediately.

### Atomic Record Appends

GFS guarantees to append data to the file at least once atomically at an offset of GFS’s choosing and returns that offset to the client. GFS does not guarantee that all replicas are bytewise identical. It only guarantees that the data is written at least once as an atomic unit. All replicas are at least as long as the end of record and therefore any future record will be assigned a higher offset or a different chunk even if a different replica later becomes the primary.

### Snapshot

The snapshot operation makes a copy of a file or a directory tree. There are some steps:

1. Revoke current leases
2. Log the snapshot operation
3. All following write operations would be applied to a copied new chunk, instead of the snapshotted original one

### Mater operation

The master executes all namespace operations. In addition, it manages chunk replicas throughout the system:

1. **Namespace Management and Locking**: allow multiple operations to be active and use locks over regions of the namespace to ensure proper serialization.
2. **Replica Placement**: maximize data reliability and availability, and maximize network bandwidth utilization.
3. **Creation, Re-replication, Rebalancing**: re-replication happens when the number of replicas is not enough; rebalancing happens for better disk space usage and load balancing.
4. **Garbage Collection**: After a file is deleted, GFS does not immediately reclaim the available physical storage. It does so only lazily during regular garbage collection at both the file and chunk levels.
5. **Stale Replica Detection**: the master uses version number to detect stale replicates and remove them.

## Fault tolerance

### High availability

GFS guarantees high availability mainly by **Fast Recovery** and **chunk replications**

### Data Integrity

GFS does not guarantee identical replicas. Therefore, each chunkserver must independently verify the integrity of its own copy by maintaining checksums.

For reads, the chunkserver verifies the checksum of data blocks that overlap the read range before returning any data.

For writes, GFS reads and verifies the first and last blocks of the range being overwritten, then performs the write.