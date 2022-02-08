---
Sort: 7
---



# Pocket: Elastic Ephemeral Storage for Serverless Analytics (OSDI'18)

## Summary

The author of this paper indicated a difficulty: It’s hard to communicate directly between serverless tasks (functions), which put an obstacle for exchanging intermediate data between execution stages in analytic jobs. In this way, the author presented a new data storage resolution (Pocket) for elastic, distributed data store that automatically scales. A detailed illustration and explanation have been given in the paper, as well as necessary experiments to show the high performance and low cost features of Pocket.

### Q1: Why isn't using existing in-memory key-value stores such as Redis and Memcached a good option for storing ephemeral data in serverless computing?

**A:** Because these systems require high cost of DRAM, as well as require users to manage the storage cluster scale and configuration, which includes a bunch of resources to provision.

### Q2: How does Pocket balance storage load?

**A:** Pocket focuses on steering data for incoming jobs across active and new storage servers joining the cluster. Because Pocket could assign specific weights for storage servers in each job’s weight map, the controller would assign higher weights to under-utilized storage servers to lead jobs assigned to them for load balance.

### Q3: Do you think Pocket solve all the problems of managing states in serverless computing? If not, what do you think are the remaining problems?

**A:** It’s not that easy to solve all the problems of state management, and Pocket mainly focus on solving the ephemeral data sharing difficulties in serverless analytics. However, for a general stateful application, state storage could be long-term and each time it finishes its job, its state would be stored in whatever storage, and the next time it is invoked would cause state reload and cold start problem. So for communication, Pocket could be a good solution, but for application state management, there could still be a gap.
