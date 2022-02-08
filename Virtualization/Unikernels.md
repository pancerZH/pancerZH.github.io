---
sort: 8
---



# Unikernels: Library Operating Systems for the Cloud (ASPLOS'13)

## Summary

This paper mainly introduced efforts on applying LibOS on cloud service management by comping up with Unikernels, which solves the difficulty of compatibility between different hadrwares by making use of hypervisor platform. The author talked about some critical features of Unikernel, as well as related concepts, to show how they designed the architecture to achieve their goals without sacrificing performance (in fact, most performances have been improved) and security. Detailed experiments were also given to illustrate the high performance and low latency.

### Q1: Name one benefit and one drawback of compiling a single-image VM.

**A:** Single-image VMs require static configuration generated in compile-time, which eliminate unnecessary features from final VMs and reduce the remote attack surface . However, given these static configurations, VMs could not be cloned by taking a COW snapshot of an existing image.

### Q2: Unikernel runs in a single address space. Give one example of how this design helps improve performance. 

**A:** In Unikernel, access control within the appliance does not require userspace processes, but depending on the language’s type-safety to enforce restrictions, which makes the single-address space model possible. In this way, the userspace/kernel boundary has been eliminated, and the Linux's syscall overhead has been reduced, which brings lower and more predictable latency when waking up parallel threads.

### Q3: Comparing gVisor and Unikernels, which one do you think is more secure and which is more lightweight?

**A:** I believe Unikernels is more secure, because by generating static-configuration in compile-time, unnecessary functions could be eliminated from the very beginning, which reduces the remote attack surface greatly. Besides, Unikernels runs on hypervisor while gVisor just provides its own system call APIs - the isolation level is different and Unikernels wins with no surprise.

As for the topic which is more lightweight I would say gVisor is more lightweight, because gVisor could be viewed as a middle layer between LXC and host kernel by providing its own system calls. But for Unikernels, developers had to re-implement a lot of basic protocols firstly to make it able to generate a single-image VM with all needed functions. The development workload is much greater than the that of gVisor. Besides, gVisor only intercepts system calls and acts as the guest kernel, but Unikernels has to compile code and generate VMs.
