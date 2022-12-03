---
sort: 1
---


# A Comparison of Software and Hardware Techniques for x86 Virtualization

## Summary

This paper compared two virtualization technologies - the existing and currently used software virtualization and the newly released hardware virtualization. The author introduced concepts and principles of both, and designed different experiments to compare their performance and explained the reasons. In the end, the author had found that in most aspects, the software virtualization outperformed the hardware virtualization, but the author also pointed out the potential of hardware virtualization as well as its obstacles. This paper helps me a lot to understand the basic concepts of virtualization, classical virtualization and the initial design of hardware virtualization.

### Q1: Why is x86 un-virtualizable with trap-and-emulate? Give one example.

**A:** Because there is lack of traps when privileged instructions run at user-level. In the paper, the author gave an example: For a deprevileged guest, there required a kernel model popf to trap so that the VMM could emulate it against the virtual IF. However, a deprivileged popf, like any user-mode popf, simply suppressed attempts to modify IF. No trap happened and VMM could not emulate the instruction.

### Q2: How are jump instructions translated?

**A:** For jump instructions in direct control flow, they could be mapped from guest address to TC address. But for jump instructions in indirect control flow, They must be computed dtnamically, because they do not go to a fixed target. There requires a hashtable for lookup for the next target.

### Q3: With hardware virtualization extensions (e.g., Intel VT), do we still need binary translation? Why or why not?

**A:** Yes and no. For hardware virtualization itself, there is a complete design for virtualization which does not include BT. It could totally rely on guest mode and host mode conversion (exit) to finish its job. On the other hand, however, with the help of BT, we could improve the performance of hardware virtualization. For example, we could use BT on Intel CPUs and implement a performant VMM protection schema to avoid segmentation. 