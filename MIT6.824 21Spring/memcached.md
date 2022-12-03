---
sort: 13
---

# Memcached

This blog is based on this [paper](https://pdos.csail.mit.edu/6.824/papers/memcache-fb.pdf) introduced in MIT 6.824, which is worth reading. 

## Introduction

Memcached is a in-memory caching solution to accelerate querying. Facebook makes use of native memcached to construct and scale a distributed key-value store for its business requirements. The target is to provide low latency access to a shared storage pool at low cost. To achieve this goal, the designers have made trade-off on these and consistency. 

Facebook relies on memcache to lighten the read load on database. There are two scenarios for read and write:

1. **Read**: a web server firstly try to get data according to the key in cache, if the data is available, then everything is fine, but if not, it would query the data in database and update in the cache.
2. **Write**: a web server firstly updates the record in database, and then deletes the related record in cache to avoid inconsistency.

Note that the operation in writing is to delete the related record in cache, that is because the delete operation is idempotent, which would reduce a lot of trouble.

As is, memcached provides no server-to-server coordination; it is an in-memory hash table running on a single server.

## Latency and Load

This part focus on reducing either the latency of fetching cached data or the load imposed due to a cache miss. 

### Reducing Latency

To reduce latency, Items are distributed across the memcached servers through consistent hashing, focusing on the memcache client, which runs on each web server. This client serves a range of functions, including serialization, compression, request routing, error handling, and request batching.

On the other hand, minimizing the number of network round trips for responses is also necessary. To achieve this, the designers introduce a directed acyclic graph (DAG) representing the dependencies between data, to maximize the number of items that can be fetched concurrently.

Client-server communication is another point needing attention. Memcached servers do not communicate with each other, and complexity of the system should be embed into a stateless client rather than in the memcached servers. Clients use UDP and TCP to communicate with memcached servers. And UDP is good for get requests to reduce latency and overhead, because TCP naturally requires more CPU and other resource load. For consistency, each UDP request should contain a sequence number for failure and order detection. But for reliability, clients perform set and delete operations over TCP, and a retry mechanism is also introduced.

Memcache clients implement flow- control mechanisms to limit incast congestion, and clients therefore use a sliding window mechanism to control the number of outstanding requests. There is a balance between these extremes where unnecessary latency can be avoided and incast congestion can be minimized.

### Reducing Load

We use memcache to reduce the frequency of fetching data along more expensive paths. There are two issues requiring solved: stale sets and thundering herds. The designers use a mechanism called **lease** to solve them both.

A stale set occurs when a web server sets a value in memcache that does not reflect the latest value that should be cached. This can occur when concurrent updates to memcache get reordered. Here's an example of the "stale set" problem:

> ```
> 1. Client C1 asks memcache for k; memcache says k doesn't exist.
> 2. C1 asks MySQL for k, MySQL replies with value 1.
>    C1 is slow at this point for some reason...
> 3. Someone updates k's value in MySQL to 2.
> 4. MySQL/mcsqueal/mcrouter send an invalidate for k to memcache,
>    though memcache is not caching k, so there's nothing to invalidate.
> 5. C2 asks memcache for k; memcache says k doesn't exist.
> 6. C2 asks MySQL for k, mySQL replies with value 2.
> 7. C2 installs k=2 in memcache.
> 8. C1 installs k=1 in memcache.
> ```

A thundering herd happens when a specific key undergoes heavy read and write activity. As the write activity repeatedly invalidates the recently set values, many reads default to the more costly path. Here is a detailed description:

> ```
> * key k is very popular -- lots of clients read it.
> * ordinarily clients read k from memcache, which is fast.
> * but suppose someone writes k, causing it to be invalidated in memcache.
> * for a while, every client that tries to read k will miss in memcache.
> * they will all ask MySQL for k.
> * MySQL may be overloaded with too many simultaneous requests.
> ```

A memcached instance gives a lease to a client to set data back into the cache when that client experiences a cache miss. Memcached can verify and determine whether the data should be stored and thus arbitrate concurrent writes. Verification can fail if memcached has invalidated the lease token due to receiving a delete request for that item.

In this way, the stale set issue could be fixed like this:

> ```
> 1. Client C1 asks memcache for k; memcache says k doesn't exist,
>    and returns lease L1 to C1.
> 2. C1 asks MySQL for k, MySQL replies with value 1.
>    C1 is slow at this point for some reason...
> 3. Someone updates k's value in MySQL to 2.
> 4. MySQL/mcsqueal/mcrouter send an invalidate for k to memcache,
>    though memcache is not caching k, so there's nothing to invalidate.
>    But memcache does invalidate C1's lease L1 (deletes L1 from its set
>    of valid leases).
> 5. C2 asks memcache for k; memcache says k doesn't exist,
>    and returns lease L2 to C2 (since there was no current lease for k).
> 6. C2 asks MySQL for k, mySQL replies with value 2.
> 7. C2 installs k=2 in memcache, supplying valid lease L2.
> 8. C1 installs k=1 in memcache, supplying invalid lease L1,
>    so memcache ignores C1.
> ```

And the thunder herd issue could also be fixed to require only the first client that misses to ask MySQL for the latest data, and other queries should wait for the results to appear in cache.

Another mechanism is memcache pool. Different applications’ workloads can produce negative interference resulting in decreased hit rates. So the designers partition a cluster’s memcached servers into separate pools, and replication could be used to improve the latency and efficiency of memcached servers.

To insulate backend services from failures, designers dedicate a small set of machines, named *Gutter*, to take over the responsibilities of a few failed servers. By using Gutter to store these results, a substantial fraction of these failures are converted into hits in the gutter pool thereby reducing load on the backing store. And Gutter mainly focus on performance, instead of consistency. In this way, a client that writes data does not delete the corresponding key from the Gutter servers, even though the client does try to delete the key from the ordinary Memcached servers, or the Gutter solution would have no difference from rehashing the request to another ordinary server. And entries in Gutter expire quickly, which is enough for obviating Gutter invalidation.

## In a Region: Replication

Designers split our web and memcached servers into multiple *frontend clusters*. These clusters, along with a storage cluster that contain the databases, define a *region*. This region architecture also allows for smaller failure domains and a tractable network configuration. The storage cluster is responsible for invalidating cached data to keep frontend clusters consistent with the authoritative versions.

They also deploy invalidation daemons (named mcsqueal) on every database. Each daemon inspects the SQL statements that its database commits, **extracts** any deletes, and broadcasts these deletes to the memcache deployment in every frontend cluster in that region. 

To reduce packet rate, the invalidation daemons **batch deletes** into fewer packets and send them. 

Each cluster independently caches data depending on the mix of the user requests, and we can reduce the number of replicas by
having multiple frontend clusters share the same set of memcached servers, which is called *regional pool*. Replication trades more memcached servers for less inter-cluster bandwidth, lower latency, and better fault tolerance.

A system called *Cold Cluster Warmup* mitigates this by allowing clients in the “cold cluster” (i.e. the frontend cluster that has an empty cache) to retrieve data from the “warm cluster” (i.e. a cluster that has caches with normal hit rates) rather than the persistent storage. To avoid inconsistencies, all deletes to the cold cluster are issued with a two second hold-off. When a miss is detected in the cold cluster, the client re-requests the key from the warm cluster and adds it into the cold cluster. The failure of the add indicates that newer data is available.

## Across Regions: Consistency

The design is that one region to hold the master databases and the other regions to contain read-only replicas, relying on MySQL’s replication mechanism to keep replica databases up-to-date. But replica databases may lag behind the master database.

The final decision is to sacrifice consistency by a little for better performance and availability. Because for Facebook, it is acceptable for uses to see inconsistent data in a short period, it could be easily ignored, or be solved by refreshing the web browser. While designers still employ a remote marker mechanism to minimize the probability of reading stale data, indicating that data in the local replica database are potentially stale and the query should be redirected to the master region.

By sharing the same channel of communication for the delete stream as the database replication we could gain network efficiency on lower bandwidth connections. Databases and mcrouters buffer deletes when downstream components become unresponsive. A failure or delay in any of the components results in an increased probability of reading stale data.

## Single Server Improvement

The *all-to-all* communication pattern implies that a single server can become a bottleneck for a cluster. There are some optimizations:

1. allow automatic expansion of the hash table to avoid look-up times drifting to *O(n)*
2. make the server multi-threaded using a global lock to protect multiple data structures
3. giving each thread its own UDP port to reduce contention when sending replies and later spreading interrupt processing overhead

Memcached employs a slab allocator to manage memory. The allocator organizes memory into slab classes, each of which contains pre-allocated, uniformly sized chunks of memory. Once a memcached server can no longer allocate free memory, storage for new items is done by evicting the least recently used (LRU) item within that slab class.

Memcached lazily evicts such entries by checking expiration times when serving a get request for that item or when they reach the end of the LRU. So the designers introduce a hybrid scheme that relies on lazy eviction for most keys and proactively evicts short-lived keys when they expire. They place short-lived items into a circular buffer of linked lists (indexed by seconds until expiration) – called the *Transient Item Cache* – based on the expiration time of the item.

## Summary

Some lessons:

1. Separating cache and persistent storage systems allows us to **independently scale** them.
2. Features that improve **monitoring**, **debugging** and **operational efficiency** are as important as performance. 
3. Managing stateful components is operationally more complex than stateless ones, **keeping logic in a stateless client**.
4. The system must support **gradual rollout** and **rollback** of new features.
5. **Simplicity** is vital. 