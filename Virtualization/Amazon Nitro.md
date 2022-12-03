---
sort: 14
---

# Amazon Nitro

## Summary

Nitro is a new infrastructure of virtualization. In traditional virtualization, the hypervisor is responsible for virtualizing resources, protecting host OS and hardware, and capture and translate sensitive and privileged instructions. However, in Nitro, these functions and protection are broken apart, and different hardwares and softwares would take a part of them respectively. Important components involve Nitro Cards, which is used to improve I/O performance; Nitro Security Chip, which is the root of trust; and Nitro Hypervisor, a lightweight hypervisor with excellent performance which only does things when required.

### Q1: With Amazon Nitro, virtualization functions are mostly offloaded to hardware. Do we still need a hypervisor (or an OS)? Can everything just run in user space and interact with Nitro cards directly?

**A:** At most time, hypervisor is still needed for a better isolation and security. However, in some cases, like workloads that cannot run in a virtualized environment due to licensing requirements, or those that need specific hardware features only provided through direct access, Nitro offers direct access to the bare metal of the host without a hypervisor involved. In this way, instances could interact with Nitro cards directly.

### Q2: Can you think of a drawback of offloading tasks to hardware (i.e., Nitro's approach)?

**A:** The management of a lot of different cards could be a pain point. When developing and debugging of Nitro functions, cases could be more complicated compared to traditional hypervisor mode. Besides, the communication between different cards also require extra efforts and resources.
