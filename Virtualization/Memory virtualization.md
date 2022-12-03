---
sort: 2
---


# Performance Evaluation of Intel EPT Hardware Assist & Memory Resource Management in VMware ESX Server (OSDI'02)

## Summary

The first paper introduced the virtualization of MMU, and the architecture of MMU, as well as the concepts and details of software MMU and hardware MMU.

The second paper mainly introduced 2 techniques supporting overcommitment: ballooning technique used to reclaim pages and content-based page sharing technique used to share pages among VM. This paper also represented the obstacles and difficulties for implementing these techniques, as well as related experiments to show the excellent performance of both techniques.

### Q1: List at least one pro and one con for software MMU and for hardware MMU.

**A:** 

- software MMU
  - Pro: It could make use shadow page table to do mapping of LPN -> MPN directly, which could avoid LPN -> PPN -> MPN overhead.
  - Con: The VMM has to keep these shadow page tables synchronized to the guest page tables, and this synchronization introduced virtualization overhead when the guest updates its page tables.

- hardware MMU
  - Pro: During the guest page walk, the hardware also walks the nested page tables to determine the corresponding MPN. This eliminates the need to maintain shadow page tables and synchronize them with the guest page tables.
  - Con: The extra operation also increases the cost of a page walk, thereby impacting the performance of applications that stress the TLB.

### Q2: What is the double paging problem and what caused it?

**A:** When a page on hardware has been selected to reclaim and paged out, and the guest OS happens to determine to page the very same page out for memory pressure, this page would be faulted in from the system paging device, and then written to the virtual paging device, which would cause great overhead.

The reason is that the paging on hardware is transparent to the guest OS, and the guest OS has no idea that some pages have already been paged out. In this way, the guest OS could select these pages to perform its own redundant page-out operation.

### Q3: What is the benefit of keeping a "hint" entry for each scanned (but unshared) page (as compared to not maintaining anything for the page)

**A:** If there is no hint, each scanned page would be treated COW, which would incur unnecessary overhead on subsequent writes, because this very page is unshared in fact, and of course it does not require COW. But with the assistance of hint, the contents of the hint page could be rehashed when compared with other pages later, and it would not trigger COW when it is not shared.