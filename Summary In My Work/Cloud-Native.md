---
sort: 2
---

# Cloud Native - Introduction Note

## Advantages of cloud-native application architectures

### Speed

Deploy more frequently.

- Recover from mistakes quickly
- Deploy easily
- Apply changes quickly

### Safety

Stability, availability and durability.

Recovering fast instead of mistake prevention.

- **Visibility** - Find failures easily and quickly. Feature-rich metrics, monitoring, alerting and data visualization frameworks and tools.
- **Fault isolation** - Limit the scope of components or features that could be affected by a failure. Employ microservices and fault tolerance.
- **Fault tolerance** - Prevent cascading failure.
- **Automated recovery** - Restart or redeploy the service in question can sometimes be helpful. Cloud-native applications can do it automatically.

### Scale

- Consumer: Horizontally scale application instances across multi cheap machines.
- Cloud Provider: Virtualizing several smaller servers in the same large server and assign isolated workloads to them.

Stateless applications can be created and destroyed quickly.

## Cloud-native architectures

### Twelve-factor applications

This is a collection of patterns for cloud-native application architectures.

**Codebase** - Each deployable app is tracked as one codebase tracked in revision control.

**Dependencies** - An app explicitly declares and isolates dependencies via appropriate tooling (e.g., Maven) rather than depending on implicitly realized dependencies in its deployment environment.

**Config** - Configuration, or anything that is likely differ between deployment environments (e.g., development, staging) is injected via operating system-level environment variables.

**Backing services** - Like database or message broker, are treated as attached resources and consumed identically across all environments.

**Build, release, run** - Different stages are strictly separated.

**Process** - The app executes as one or more stateless processes that share nothing. State is externalized to backing services.

**Port binding** - The app is self-contained and exports any/all services via port binding.

**Concurrency** - Accomplished by scaling out app processes horizontally.

**Disposability** - Robustness is maximized via processes that start up quickly and shut doe gracefully.

**Dev/prod parity** - Continuous delivery and deployment are enabled by keeping development, staging, and production environments as similar as possible.

**Logs** - Treat logs as event streams. Use centralized services to collect, aggregate, index, and analyze the events in the execution environment.

**Admin processes** - Administrative or managements tasks are executed as one-off processes in environments identical to the app's long-running processes.

### Microservices

Microservices are independently deployed services that do one thing well each.

- We decouple the change cycles when we decouple the business domain into independently deployable bounded contexts of capabilities.
- Development can be accelerated by scaling the development organization.
- The new developers can become productive more rapidly due to reduced cognitive load od learning.
- Adoption of new technology can be accelerated.
- Microservices offer independent, efficient scaling of services.

### Self-service agile infrastructure

Successful adopters of cloud-native applications have empowered teams with self-service platforms. So that we need a team to provide a platform on which to deploy and operate these microservices.

1. The application code would be pushed in the form of pre-built artifacts or raw source code to a Git remote.
2. The platform would then build the application artifact, construct an application environment, deploy the application and start the necessary processes.

Teams should not take care of where and how their applications run, but the platform does.

This model is also for backing services. These service instances could be bound to the application and automatically injected into the environment to be consumed.

These platforms also provide:

- Automated and on-demand scaling of application instances
- Application health management
- Dynamic routing and load balancing of requests to and across application instances
- Aggregation of logs and metrics

### API-based collaboration

The sole mode of interaction between services in a cloud-native application architecture is via published and versioned APIs, which are typically RESY-style with JSON serialization. It is generally the same for the self-service infrastructure platform.

Service consumers are not allowed to gain access to private implementation details of their dependencies or directly access their dependencies' data stores. Only one service is allowed to gain direct access to any data store. This forced decoupling directly supports the cloud-native goal of speed.

### Antifragility

The opposite of fragility is antifragility, or the quality of a system that gets stronger when subjected to stressors. We can build this kind of system by injecting random failures into production components to identify and eliminate weaknesses in the architecture.

## Changes needed

To adopt cloud-native application architectures, the overall theme is one of decentralization and autonomy.

- DevOps - Decentralization of skill sets into cross-functional teams
- Continuous delivery - Decentralization of the release schedule and process
- Autonomy - Decentralization of decision making

We codify this decentralization into two primary team structures:

- Business capability teams - Cross-functional teams that make their own decisions about design, process, and release schedule.
- Platform operations teams - Teams that provide the cross-functional teams with the platform they need to operate.

And technically, we also decentralize control:

- Monoliths to microservices - Control of individual business capabilities is distributed to individual autonomous services.
- Bounded contexts - Controls of internally consistent subsets of the business domain model is distributed to microservices.
- Containerization - Control of application packaging is distributed to the service endpoints.
- Choreography - Controls of service integration is distributed to the service endpoints.

All of these changes create autonomous units that are able to safely move at the desired rate of innovation.

## How to move toward a cloud-native application architecture

Two recipes:

- Decomposition - We break down monolithic application by:
  1. Building all new features as microservices.
  2. Integrating new microservices with the monolith via anti-corruption layers.
  3. Strangling the monolith by identifying bounded contexts and extracting services.
- Distributed systems - We comprise distributed systems by:
  1. Versioning, distributing, and refreshing configuration via a configuration server and management bus.
  2. Dynamically discovering remote dependencies.
  3. Decentralizing load balancing divisions.
  4. Preventing cascading failures through circuit breakers and bulkheads.
  5. Integrating on the behalf of specific clients via API Gateways.