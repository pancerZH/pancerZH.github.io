---
sort: 12
---

# When Virtual is Harder than Real: Security Challenges in Virtual Machine Based Computing Environments (HotOS'05) and Hey, You, Get Off of My Cloud: Exploring Information Leakage in Third-Party Compute Clouds (CCS'09)

## Summary

The first paper introduced the security challenges when VM were widely used in computing environments. Given the fast growth and flexibility natures of VM, traditional security mechanism would not work as before. This paper explained and analyzed these concepts and reasons in detail, and then came up an idea to add security mechanisms at VMM level. Analysis of advantages was given then to support the idea.

The second paper mainly introduced the corss-VM side-channel attacks. In this paper, the author firstly introduced threat models and the platform they used to perform their experimental attacks. By describing the underlying mechanism of the cloud provider, they figured out some ways to determine co-residence. Then some attack strategies were given, and their performance was compared. Also some potential data leakage was shown after attackers had succeeded. Finally, a suggestion of defending the co-location attack was given.

### Q1: Can you think of some drawback of enforcing security mechanisms at the hypervisor level (compared to at the guest OS or above)?

**A:** I believe there could exist some performance overhead by implementing this mechanism, because VMM would definitely do more work than its original design, and in a way trying to understand what the guest OS’ purposes, which is not that straightforward compared to what could have been done in the guest OS. Then the ability of understanding guest OS could also be a burden.

### Q2: When a zone and/or an instance type are used more frequently (i.e., having higher loads from more tenants), do you think the co-location attack would be come easier or harder? Why?

**A:** I think it would be harder, because with higher usage of certain zones or types, the ability of attackers would be diluted. For example, it would be harder for attackers to locate their targets with the same number of VMs used for attack, or it would cost them more to achieve their original success rate. 

### Q3: Do you think a similar co-location attack exist with serverless computing (i.e., one function attacking another function on the same physical machine)? Does serverless computing make such attacks harder or easier and why?

**A:** Yes. A function is located in a container or a VM(MicroVM), and functions from different tenants could also run on the same metal machine. But it would be harder to achieve the co-location attack, because the existence time of functions would be much shorter compared to VMs, and then the footprints could be collected and destroyed shortly after startup. This would be much harder for attackers to notice a function has started and start their co-location attack before everything is collected.
