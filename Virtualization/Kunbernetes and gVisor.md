---
sort: 5
---


# Kubernetes and gVisor

## Summary

The Kubernetes document mainly introduces the concepts, architecture(including components) and its usage. Kubernetes mainly focus on container management, and is a framework to run distributed systems resiliently. It takes care of scaling and failover for your application, provides deployment patterns and so on.

The gVisor document illustrates the main idea of gVisor: a layer between applications and host kernel for a better isolation. The document also simply introduces main components of gVisor, as well as how they work. There is also a comparsion between gVisor and other two approaches to achieve better container isolation with their hosts.

### Q1: What is a Kubernetes Pod? How do you think it is useful in container orchestration?

**A:** Pods are the smallest deployable units of computing that you can create and manage in Kubernetes, also a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers. For me, it is like the process in OS. 

Kubernetes Pod is a universal standard for container runtime, which could hold different containers, so that Kubernetes could make use of the universal model, instead of handling different containers separately. Also the Pod configuration could be useful for K8s to construct and schedule related containers automatically, instead of manually.

### Q2: What does Kubernetes use etcd for? Why is having a consistent, atomic key-value store important for Kubernetes' control plane?

**A:** K8s stores configuration data into etcd for service discovery and cluster management, also used as K8s’ backing store for all cluster data. 

etcd’s consistency is crucial for correctly scheduling and operating services, which is K8s’ control plane’s job. The K8s API server persists cluster state into etcd. It uses etcd’s watch API to monitor the cluster and roll out critical configuration changes.

### Q3: Vulnerabilities in the Linux kernel makes it unsafe for containers to call Linux system calls. How does gVisor solve this problem?

**A:** For each Linux system calls from applications within containers, gVisor intercepts it and acts as the guest kernel, without the need for translation through virtualized hardware, like the VMM. gVisor uses Sentry to finish the work, and Sentry would make some host system calls for application system calls passed in as a userspace application. Also, gVisor is written in Go in order to avoid security pitfalls that can plague kernels. 
