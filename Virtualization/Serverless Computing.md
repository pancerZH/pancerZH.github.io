---
sort: 6
---



# Cloud Programming Simplified: A Berkeley View on Serverless Computing & Serverless in the Wild: Characterizing and Optimizing the Serverless Workload at a Large Cloud Provider (ATC'20)

## Summary

The Berkeley View mainly introduced key ideas in FaaS (Function as a Service), including its history, concepts (serverless, elastic, charging and etc.), and key differences when compared with serverful computing. This view also indicates its role in virtualization by comparing serverless computing with VM-level isolation and OS-level isolation.

The second paper introduced a resource management policy aiming at solving cold-start problem of functions. The author firstly characterized main features of Function workload on Azure, and summarized some points to start with. The author then came up with their own policy and compared it with state-of-practice policy by conducting experiments.

### Q1: Today's serverless functions are stateless. How do you think different functions can share data and communicate?

**A:** 

1. Network communication
2. Remote memory sharing
3. External storage system

### Q2: Can you think of any security threats of serverless computing? Bonus points if you can outline a real threat/attack.

**A:** Considering the ware start policy in the paper: The current policy is to give a fix time of resources remaining in memory after execution. However, if an attacker periodically triggers (by any means) the function, these resources would be always remaining in memory, which would cause extra cost and resource waste. 

Permission policy could also be a weak point: if a malicious user was granted permission to change other user’s permission, bad things would happen. Sometimes it’s even not about a malicious user, a developer with improper permission could also perform operations harmful by accident - just like dropping a database accidentally.

A real attack could be:

1. An attacker injects shell command in function with high permissions
2. The attacker gets CLI Tool
3. By checking roles and policies, the attacker could modify function configuration leading to extra cost or service unavailable.

### Q3: Can you think of any other ways to reduce or avoid cold start for serverless computing (other than what the ATC'20 paper talks about).

**A:** Statically typed languages, such as Java and C#, start much slower compared to scripting languages like Python and Ruby. In this way, sticking on scripting languages when developing functions could be a good way to reduce cold start. 

Another way is to keep necessary code and libraries only, which would reduce uncompressing time of uploaded compressed package with the function.