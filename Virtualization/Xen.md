---
sort: 10
---

# Xen and the Art of Virtualization (SOSP'03)

## Summary

To allow multiple OSes to share hardware in a safe and resource managed fashion while without sacrificing performance or functionality, this paper came up with Xen, a paravirtualization VMM. Xen modifies guest OSes a bit to enable necessary traps on X86, on which some sensitive operations could not be trapped. Also, Xen introduced some techniques to improve performance, isolation and security, like direct access to guest OS system call handlers, lower privilege level of guest OSes and asynchronous event notifications. The author then introduced the design of Xen in detail, and presented a lot of experiments to prove its good performance and how well their goals are achieved.

### Q1: Why can Xen allow guest OS system call handlers to be accessed directly (without any ring-0 Xen involvement) but not guest page fault handler?

**A:** For system calls, guest OS is allowed to install a fast exception handler for being accessed directly by the processor without indirecting via ring 0, while only code executing in ring 0 can read the faulting address from register CR2, which makes page faults must always be delivered via Xen so that the register value could then be saved for access via ring 1.

### Q2: What's the benefit of using asynchronous event notifications from Xen to a VM?

**A:** For notifications, they are made by updating a bitmap of pending event types and calling an event handler. These callbacks could be held off at the discretion of the guest OS, which is asynchronous and avoids extra costs incurred by frequent wake-up notifications.

### Q3: What goals of Xen are not valid or less valid in today's cloud environments? 

**A:** 

1. Xen is a paravirtualization VMM, which requires modification on guest OSes, while no matter how minor modifications required, it violates the needs of common users to run their applications on original (also the most universal) OSes. Besides, once OS distributions have been updated, these modifications should also be checked and maintained, and trust between users and these modified OSes is also an issue. 
2. Nowadays, cloud applications like microservices or even functions, they do not require complex configurations, or supports for multi-application running. They could be as simple as running one single application(service or function). 
