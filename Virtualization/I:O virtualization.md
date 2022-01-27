# virtio: Towards a De-Facto Standard For Virtual I/O Devices & High Performance Network Virtualization with SR-IOV & Network Virtualization Overview

## Summary

The first paper introduced the concepts, some basic ideas, and APIs of virtio, indicating its usage for I/O device virtualization, as well as its benefits.

The second paper introduced the single-root I/O virtualization (SR-IOV), a standard to allow an I/O device to be shared among VMs. This paper also mentioned the architecture of SR-IOV, including management and configuration of physical devices, VFs, PFs, and IOVM.

The last one covers the whole view of network virtualization, starting from requirements and challenges, this chapter introduced the architecture of virtual networks (Management, Control and Data Planes), the encapsulation of virtual network (outer - inner headers), hardware involvement and also an example to illustrate all.

### Q1: Is virtio a full virtualization or a paravirtualization technique? What's its main benefit?

**A:** Virtio is a ~~full virtualization, though those I/O devices know they are running on VMs, OSes are not modified.~~ paravirtualization, because the guest OSes are modified with some front-end drivers (new drivers run in guest OSes). The main benefit should be this mechanism is both efficient (gaining some benefits from paravirtualization because these devices know the fact. ~~e.g. data movement~~) ~~and unified which is compatible with multiple hypervisors and platforms~~. *High performance*

### Q2: List at least one limitation of SR-IOV

**A:** It requires specific hardwares. For example, VF requires SR-IOV-capable devices to be enabled; It also requires modern NICs which provide the capability to quiesce interrupt firing for a certain time to moderate the interrupt frequency (though maybe not necessary, it would lead to overhead without this kind of support).

### Q3: What are the similarities and differences between network virtualization and traditional server virtualization?

**A:** 

 - Similarities
   - They both provide a full feature set of a physical network and a physical server respectively.
   - They both can be moved around and keep working correctly.
 - Differences
   - There is a middle layer (network hypervisor) between software and hardware, and all instructions and communications between software and hardware would be captured and processed by the middle layer. In traditional server virtualization, some instructions (non-privileged or insensitive instructions) would not be captured by VMM and could be run directly.