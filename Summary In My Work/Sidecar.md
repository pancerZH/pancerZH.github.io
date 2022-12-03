---
sort: 5
---

# Sidecar pattern in microservices

## Background

In my team, there are always scenarios like this: someone implemented a complex logic, and the similar requirement would also apply for others. In this way, the developer would encapsulate these details into a JDK, or, a JAR package, and provide to others in the team appending with our architecture. Also, it could be the same when we deal with some common operations, like recording logs and dealing with business objects. Providing a package in our architecture is common, and mostly works fine. 

Our team is developing cloud native products, and thet are supported by lots of microservices. Our programming language is mainly Java, which makes the above operation acceptable. But there are still some flaws:

1. There are also some microservices programmed by other languages, like Go. The JDK would not be applicable for them.
2. Developers still need some to pay some attention to learning how to use these packages. Also, some operations and codes are redundant in microservices, which are repeated and repeated again in them.

In this way, we introduced a pattern to solve these trouble: this is the **sidecar pattern**.

## Usage

In my opinion, the key point of sidecar pattern is *decoupling*, which is everywhere in our designs. For example, we extract some shared operations into the sidecar implementation to reduce the working load of developers. In our services, we need to create, edit or delete ConfigMap in Kubernetes clusters. This work, however, can be redundant for each programmer to do the same thing in their own code. We use some REST APIs to send requests to the sidecar and let it finish the operations of ConfigMap.

Just as I described above, sidecar is not only for Java programmed microservices, but also for all mocroservices. This is because of the communication method: HTTP request. Instead of using lanaguage-specific packages, HTTP request is of course applicable for everyone. However, this does not come without any prices. The communication can be somewhat expensive, compared to direct operations by JDK, though not significant. There must be a consideration and comprise.

In summary, the sidecar pattern would be suitable for some shared operations, basic operations (in fact they could also be recognized as shared operations) and complex operations. The decoupling involves basic functions and business functions, functions between different domains (microservices) and even services and platforms.

## Concept

Why could sidecar work? The first and most basic reason is that in Kubernetes, each pod could include more than one containers (every container could contain one service), and all the containers in the same pod would share the same resources. In other words, by putting the sidecar and main application in the same pod, the sidecar container would share the same database and network resources, as well as the same life cycle with the main application. This indicates we do not pay attention to these implementation details, and the call to the sidecar could ba as simple as a local HTTP call.

![sidecar](./img/sidecar.png)

By putting them into the same pod with a shared host, the independency of serives could be guaranteed, and all the above altogether would provide the whole function.

## Benefits

1. Reduces the complexity in the microservice code by abstracting the common infrastructure-related functionalities to a different layer.
2. Reduces code duplication in a microservice architecture since you do not need to write configuration code inside each microservice.
3. Provide loose coupling between application code and the underlying platform.