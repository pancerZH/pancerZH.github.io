---
sort: 4
---


# Understanding and Hardening Linux Containers

## Summary

This whitepaper introduces the basic ideas of Linux containers, as well as their main components (5 in total, and 3 are touched in ch3,4,5). The 3 main components covered are Kernel Namespaces, which are used for application isolation; Control Groups, which are used for host hardware resource limitation; and Root Capabilities, which are used to limit the power of root users in container and also make them be able to run some privileged operations. Besides, this whitepaper touches a lot of details of them, also some examples are given. Though they look pretty different, the key idea behind Linux containers is the same with traditional server virtualization (resource virtualization), as well as some challenges (privilege control, isolation and e.g. ).

### Q1: What types of isolations does Linux containers achieve?

**A:** They achieve OS-level virtualization, involving process isolation, namespace isolation and hardware resource isolation (performance isolation).

### Q2: Can one Linux container affect the performance of another Linux container on the same machine (i.e., performance isolation)? Why or why not?

**A:** LXC could achieve performance isolation with the assistance of Control Groups (cgroups) in theory, which are used to limit host hardware resource usage of each container. However, it is hard to say a LXC would **never** affect the performance of another LXC on the same machine. I read a [paper](https://repositorio.pucrs.br/dspace/bitstream/10923/13227/2/A_Performance_Isolation_Analysis_of_Disk_Intensive_workloads_on_Container_Based_Clouds.pdf) and its author mentioned disk-incentive workloads would bring negative influences to other containers. Though the reason was not mentioned in that paper, I would assume that the performance isolation provided by cgroups is not so precise and some overheads could not be avoided.

### Q3: Why do you think containers are less "secure" than virtual machines?

**A:** I believe the main reason comes from the architecture: container virtualization (or OS-level virtualization) does not involve hypervisor, which could be used to capture privileged or sensitive instructions. Containers share the same kernel with the host, and this would bring tons of attacking surfaces including syscalls, network, hardware resources and information leakage - so many interfaces exposed to untrusted entities.
