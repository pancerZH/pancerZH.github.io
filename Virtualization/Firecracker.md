---
sort: 9
---



# Firecracker: Lightweight Virtualization for Serverless Applications (NSDI'20)

## Summary

This paper introduced Firecracker, a new Virtual Machine Monitor, for serverless workloads. Firecracker focus on Amazon’s business needs: safe isolation, low overhead and high density, good performance, good compatibility, fast switching and soft allocation. It came up with a concept MicroVM, and keeps one function in one MicroVM with one slot, to ensure high utilization of resources and safety. multi-tenancy is also employed to support oversubscription. With all these techniques, by illustrating with experiments, Firecracker has succeeded to achieve Amazon's business goals in AWS Lambda, though it could also be used in other areas in virtualization.

### Q1: What is the benefit of Firecracker over gVisor in terms of the specific goals Amazon has for their cloud production environments?

**A:** Compared to gVisor, Firecracker provides better isolation - because gVisor is a sandboxing, while Firecracker is a VMM, though it manages MicroVMs, instead of traditional VMs. In this view, Firecracker is more security. Besides, Firecracker could obtain high performance, while gVisor captures system calls from applications and replace them, which causes much delay.

### Q2: What mechanism(s) allow Firecracker to run thousands of MicroVMs on the same machine (with 10x-20x oversubscription rate)?

**A:** Firecracker’s block device and network devices offer built-in rate limiters, which allow limits to be set on operations per second and on bandwidth for each device attached to each MicroVM. In this way, the noisy-neighbor effect could be eliminated, and performance isolation could be guaranteed. Besides, soft-allocation and multi-tenancy are key techniques to achieve this goal: only necessary resources are allocated to slots, and multi-tenancy could reduce the no contention ratio.

### Q3: Why do you think Firecracker (when deployed to power AWS Lambda) run one process (one slot) in one MicroVM?

**A:** Firstly, it could gain better isolation to keep each process apart in different MicroVMs. Secondly, this could reduce the duration of initializing time for slots, as well as the boot time, because if we put multiple slots in one MicroVM, they would not start until every one is ready, and each slot would wait that long - but by keeping them in their own MicroVM, they would get to run once they are ready, without waiting for others. Last, keeping one slot in one MicroVM could gain more fine-grained resource allocation and a better resource utilization: each slot just needs to gain the resource they need and reduce idle time for resources, but this could not be avoided when multiple slots reside in a single MicroVM and require different types of resources.
