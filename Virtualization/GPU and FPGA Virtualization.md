---
sort: 13
---

# GPU Virtualization on VMware’s Hosted I/O Architecture and Do OS abstractions make sense on FPGAs? (OSDI'20)

## Summary

The first paper introduces the strategy for GPU virtualization and details about a specific GPU virtualization architecture in VMWare. This paper points out the difficulties of virtualizing GPUs, and comes up with the paravirtualization idea. Two categories of GPU virtualization are given, the first is front-end virtualization, which is more portable and enables virtualized application to run interactively; the latter one is back-end virtualization, which could gain better performance and ease of driver maintenance. Then the paper gives the VMWare’s virtual GPU solution based on front-end virtualization, and some experiments to show its features and performance.

The second paper mainly introduces the virtualization of FPGA, including some specific features of FPGA and related challenges of virtualization, by comparing to a common OS. The author also comes up with Coyote, a hybrid computing system, provides a complete minimal core set of essential features above which other services could be based. Functionality and details are given then, as well as some experiments to show advantages of Coyote.

### Q1: More than ten years later, GPUs are used more for running machine-learning workloads and in the cloud environments. For such uses, which virtualization techniques outlined in Section 3 do you think work better?

**A:** I think the back-end virtualization could be better for machine-learning workloads, because the back-end virtualization gains better performance, and machine-learning is extremely needy of performance; also the back-end virtualization supports more GPU features, which could help to accelerate training processes. Ease of driver maintenance could also reduce load of researchers to set up the environment.

### Q2: Is space sharing or time sharing harder (e.g., bigger performance overhead, harder to implement, etc.) on FPGA? Why

**A:** Time sharing is harder for FPGA, because the context switch time is much higher in FPGA, and preemption introduces great implementation difficulties. On the other hand, space sharing is much easier, because static region could be used to switch applications, or we could partition the FPGA resources statically between applications.

### Q3: Does the idea of "API remoting" apply to FPGA? For example, can we let VMs call APIs that have their implementation in an FPGA? Is that a good idea, and how's it different from API remoting on GPU?

**A:** Yes, FPGAs could be used called through APIs, but this does not work well for hybrid FPGA-based systems, and in this way I don’t think it a good idea. In fact, GPUs could be viewed as pure accelerators, and thus they could be called via APIs directly and finish their work. But for non pure computational devices like hybrid FPGA-based systems, runtime interfaces are needed, and the functionality could not be finished by a compiler solely.
