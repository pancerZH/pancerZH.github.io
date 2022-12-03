---
sort: 4
---

# Fault-Tolerant Virtual Machines

This paper introduces a kind of mechanism of providing fault-tolerant virtual machines for users written by VMWare, based on their own services. Some descriptions involve their identical technology to achieve, and we only need to focus on their ideas to deal with fault-tolerance and split-brain problem solving.

## Introduction

The basic idea is simple: A single virtual machine could crash, could fail, and could lose data. In this way, a natural thought could be use backup virtual machines to provide fault-tolerance ability, but there are some issues to be solved:

1. How to make backups synchronize with primary virtual machine?
2. How to switch to one backup smoothly and quickly after the primary VM fails?
3. How to guarantee there is always one and only one primary VM providing service to external?
4. How to avoid reducing performance significantly?

## Synchronization & Fault-tolerant Design

### Operations to synchronize 

This mechanism does not replicate VMs directly, cause it would consume too much resource and delay the responds. Otherwise, it uses the state-machine approach. Just like **Raft**, it only sends logs (operations) from leader (primary) to followers (backups) in thes same order. The input, or operations, could be deterministic or non-deterministic. The former one means that the very operation could lead to the same state by directly running it; the latter one, however, could lead to diverge by running it directly, like reading CPU clock. There must be a way to deal with non-deterministic operations.

### Hypervisor

> A hypervisor is part of a Virtual Machine system; it's the same as the Virtual Machine Monitor (VMM). The hypervisor emulates a computer, and a guest operating system (and applications) execute inside the emulated computer. The emulation in which the guest runs is often called the virtual machine. In this paper, the primary and backup are guests running inside virtual machines, and FT is part of the hypervisor implementing each virtual machine.

In brief, this paper introduces hypervisor to help to catch operations and synchronize between primary and backup VMs.

### Delay output

It is important to delay any output until all tte backups have received the operation, or a failure of the primary could cause the output information lost. In practice, the primary would send the operation to backups, and the backups would reply to inform the primary operation received. Until then, it is safe to do the output, and even the primary fails, the output information would not be lost but taken over by the next primary, because the necessary information of the output has been received by all backups.

### Quick switch when fails

As you can see, by applying these above requirements, the system could switch to a backup from crashed primary VM quickly without any data loss. But there could also appear some issues. The primary uses heartbeat message to tell backups it is still alive, just as what we did in Raft. When the backup dies not hear from the primary for a while, it then could be ready to take the position. However, what if it is only the network issue, and the primary is still running smoothly in fact? To solve this *split-brain* problem, there must be a third party marker to indicate whether there is a primary working or not. In default implementation of this paper, the marker could be the shared disk, and only after the VM holds the right to write on the disk, it could be recognized as the primary. If there is no shard-disk, just as another alternative described in the paper, there still requires another marker works in a similar way.

## Practical Implementation

### Start and Restart of VM

The designer requires there are two modes for VMs: one is logging mode, another is replay mode. The primary should be in logging mode, it needs to send the operation logs; and all other backups should be in reply mode, they read from logs and replay them, or apply them to themselves. 

### Logging Channel

The hypervisors maintain the logging channels for VMs, and there is a large buffer for logging entries for the primary and backup VMs. Logs are not sent directly from primary to the backups, but they would make use of the buffer as the transit. However, the buffer could be filled up because of the relatively slow rate of consuming. In this case, the designer does not choose to expand the buffer or drop logs, but reduce the processing rate of the primary to have backups to follow up. To be honest, I was a little surprised when I first came to this part, but after consideration, I found it reasonable. Just as explained in the paper, when the backups could not keep pace with the primary, it indicates that the workload is relatively high, and the slow-down rate is then acceptable. In this way, the complexity of the system could be reduced compared with other approaches.

### Disk I/Os

Disk I/O operations could be non-deterministic, and there is a way to guarantee these operations would not lead to diverge. When necessary, the parallel disk I/Os could be forced to run sequentially, and to get rid of disk information changes, a bounce buffer is introduced. When disk read happens, it firstly reads the data from disk to the buffer, and copies it to memory only after I/O completes; when disk write happens, the data is firstly written into the buffer and then applied to the disk. Also, primary failure could cause I/O jam, and to solve this, each time a backup is selected to be the primary, the pending I/Os would be re-issued.

### Network I/O

Similar requirements also apply to network I/Os, because network packets could be asynchronous for optimization, which is a non-deterministic operation needing disabled.

## Conclusion

This mechanism is pretty effective with some optimizations to trade off the extra performance costs caused by the synchronization requirements. It could inspire us to know more about the concept of distributed design and fault-tolerance. 