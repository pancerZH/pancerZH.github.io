---
sort: 11
---

# kvm: the Linux Virtual Machine Monitor, and QEMU, a Fast and Portable Dynamic Translator

## Summary

KVM is a Linux subsystem which leverages virtualization extensions to add a VMM capability to Linux. Under KVM, VMs are created by opening a device node, which has its own virtual memory and CPU. Guest codes are run under guest mode, while guest mode exists are handled under kernel mode, and I/O instructions are handled under user mode. This paper also introduced how KVM virtualized memory and I/O, as well as how to integrate with Linux and live migration process.

QEMU is a fast machine emulator using an original portable dynamic translator. It is used to emulate CPUs on hosts, or a hardware virtualization solution. In this paper, the author introduced how QEMU performs the dynamic translation of CPU instructions to micro operations, as well as an example and the implementation details, which also involves memory management for emulated CPUs. QEMU also supports user mode emulation.

### Q1: What is the implication of KVM forwarding I/O requests to the user space?

**A:** Userspace could feed all pio and mmio accesses into a device model in order to simulate their behavior, and possibly trigger real I/O. KVM also provides a mechanism for userspace to inject interrupts into the guest, to determine when the guest is ready to accept an interrupt.

### Q2: What is the benefit of QEMU first translating the source instructions (guest) into micro-operations implemented in C and their compiled object files and then translating the object files into the target instructions (host)?

**A:** Because direct translation could be much harder with a lot of code to be rewritten on different hosts. But by firstly translating the source instructions into small micro-operations, whose number is smaller and easier to perform compared to the whole target instruction set. The first step is done at compile time, and the next step is done at runtime, which would gain a better performance.

### Q3: Can you think of some good use cases for QEMU+KVM?

**A:** I used to work on a project involving QEMU/KVM: OPNFV, which is based on OpenStack for network functions virtualization. In this project, KVM works as the hypervisor, and multiple virtual machines run on the same host machine. QEMU is used to emulate different hardwares. Besides, as far as I know, QEMU/KVM is widely used in data center and many cloud infrastructure, like OpenStack. Because KVM could be integrated with Linux very well with good performance, and QEMU provides the necessary ability of hardware emulation.
