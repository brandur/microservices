# Microservices

- Introduction
  - [Terminology]()
  - [Practice Moderation]()
- Isolation
  - [Share Data, Not A Database]()
  - [Deploy Discretely]()
- Composition
  - [Enforce Strong Contracts]()
  - [Execute Idempotently]()
  - [Stream State Changes]()
  - [Be A Good Citizen]()
  - [Pool Connections]()
- Resilience
  - [Be Resilient]()
  - [Ensure Correctness With Atomicity]()
  - [Defend In Depth]()
- Operation
  - [Instrument, Observe, and Correlate]()
  - [Normalize Practices]()

## Introduction

### Terminology

This guide defines the following terminology for service-oriented systems:

* A **service** is a software installation that runs and is deployed independently of other parts of the larger system. It is largely interchangeable with "microservice".
* A **component** is a software module with well-defined interface boundaries that could operate as a part of service or be broken out into its own service if it's deemed to be beneficial.
* A **platform** is a greater software system that makes up all of an organization's services and components.

### Practice Moderation

Service-oriented systems can result in large organizational, logistical, and operational benefits, but have very real drawbacks. Wide reaching changes that touch many services gain significant overhead in orchestration and the system gets harder to reason about as its complexity is fundamentally increased. In many situations, monolithic architectures are more appropriate.

Practice moderation and objective thought when deciding whether to break out a service. Sketch out the API contract between itself and other components in the system. Thoroughly consider how likely those contracts are to stay roughly in the same shape over the long term. If major changes are likely, keep the monolith.

## Share Data, Not A Database

Sharing a database between services leads to an architecture that's fragile and immovable. Instead, state should be shared via well-defined public interfaces.

---

Never share an internal data schema between services because it makes changes to that schema slow, difficult, and dangerous.

Instead, have services communicate with one another over well-defined public interfaces.

## Execute Idempotently

Design API endpoints to be idempotent so that work can be safely retried in the event of a failure. Use idempotency keys where an idempotent design is not possible.

---

Messages between systems can fail at any time, leaving one system unsure of the state of the other.

Prefer idempotent API calls so that in the event of a failure, systems can safely retry operations.

Where idempotency is not possible, use a construct like idempotency keys to ensure exactly one delivery.

## Enforce Strong Contracts

Strongly define every service's interface. Ensure backwards compatibility to prevent breakages.

---

Regardless of the technology in use to communicate between services, make sure that interfaces are strongly defined and protected against accidental changes. Regressions in an API can easily break the integration between two components.

Spec APIs out using a format that declaratively describes their characteristics.

Communication via constructs like event streams and other real time APIs counts! Messages that are published into a stream should be as concretely defined as any API.

## Stream State Changes

Streaming changes in state through a log-style stream is efficient provides ordering and inherent error correction. Consider using this technique over sending multitudes of synchronous service requests.

---

Every request between two services is a potential failure.

Where a non-interactive interface is possible, batch up communication by having services stream their state to each over a construct like a Kafka topic. This method is more robust because fewer messages need to be passed, and allows consumers to ingest changes more economically because changes are grouped.

## Ensure Correctness With Atomicity

ACID transactions are the most powerful tool in existence for guaranteeing data integrity. Use them to ensure correctness within the bounds of any particular service.

---

Use them within the bounds of any particular service. Calls to external services can result in inconsistent distributed state, so wherever possible converge state between services asynchronously.

## Defend In Depth

Security should be layered. Use TLS for communication between services even when they're known to be secured within a VPC.

---

Security should be layered. Services should use a secure protocol like TLS to talk to each other, even if they're known to be isolated in something like a VPC.

## Be A Good Citizen

Services should aim to be good citizens within the context of the wider platform by behaving responsibly when making calls out to other services.

---

### Exponential Backoff


### Jitter

When waiting to retry an operation, at some random jitter to the wait time. Other nodes may be waiting to retry the same operation and the added randomness will prevent everyone from doing so in lockstep, which could be dangerous for a target service.

## Pool Connections

Use connection pools for more efficient communication between services.

---

Speed is important and microservice architectures can introduce latency as more information is passed across networks. Use connection pools when communicating between services so that connections that were opened previously can be re-used.

This is especially crucial for secure connections which have traditionally required three round trips to construct, adding a lot of additional latency (although TLS 1.3 is likely to bring this down to one round trip).

## Normalize Practices

Services within a platform all perform very different duties, but treat them the same as possible by standardizing on languages, frameworks, and tooling.

---

Services being deployed separately shouldn't mean that each one should be its own special snowflake.

The more consistent the technology stack and practices between all shared services, the lower the overhead to development and operations will be, and the easier it'll be for engineers to move between services and continue to maintain high levels of productivity. For example:

* A single protocol should be shared for all communication between services (e.g. HTTP, GRPC, ...).
* It takes a lot of experience to learn how to operate a database well, so it's best to use a single one everywhere.
* Standards around services for deployment, logging, reporting, metrics, and CI will significanly ease integration over the long run.
* Standards on languages, frameworks, and even individual libraries are a good idea to reduce cognitive overhead for engineers.
* Universal coding conventions. Use linters in CI for enforcement.

## Deploy Discretely

Deploy services so that they're compatible with the versions of every other running services. Coordinated multi-service deployments should never be a necessity.

---

Like the deployment of a single component should maintain compatibility with the code that is already running, the deployment of any component should be compatible with the overall service and not rely on lockstep deployments.

## Instrument, Observe, and Correlate

Make services and the greater system observable by instrumenting communication between services and by using tracing techniques.

---

When a service calls into another service it should offer a `Request-Id` and identify itself with a `User-Agent` where possible. An identifier should be available to trace a request's path through the entire platform and separate identifiers should be able to isolate activity within any given service.

Services should instrument the time that it takes them to run and the time they spend calling into other services. If service time for the platform is degraded, it should be possible to tell whether time is being lost to a particular service or to degenerate network conditions.

If an integration test is running against the platform and fails, it's not enough to know just that result; the test should be able to identify exactly which service caused its failure so that fixing it is immediately actionable.

## Be Resilient

Harden services by considering when their dependencies might become unavailable and using circuit breakers to protect themselves against degraded external conditions. Use rate limiting and control rods to build layers of defense.

---

Identify single points of failure and compensate for their unavailability where possible. For example, a service might not be able to make much progress anyway if it's database becomes totally unavailable, but should be able to stay online if its ephemeral Redis gets knocked offline.

### Circuit Breakers

The _circuit breaker_ pattern identifies a certain number of failures from an external service within a configured period of time. During such an event the circuit breaker "opens" so that subsequent requests to that service fail automatically and cheaply without having to wait on an expensive timeout. An external service in this sense might be another service with the platform, or an attached resource like Redis.

After a grace period, the circuit breaker re-engages and more requests are allowed through. If the external service has since recovered, operations continue normally; if not, the circuit breaker will break open again and the cycle will repeat.

The important function that circuit breakers provide is to help keep a service up even if some of its critical dependencies have gone down thus protecting against _cascading failure_. Without circuit breakers, it's easy for a service to start timing out requests as its own internal requests are timing out.

### Rate Limiting

Use rate limiting liberally to ensure that the rate of incoming requests stay within the maximum bounds of what a service is able to fulfill with its currently available resources.

In a distributed environment where thousands of servers are talking to each other, an external event can easily trigger a _thundering herd_ problem where many nodes of a service trigger requests simultaneously and accidentally DDOS another service.

This is especially important for faster services like Go or Rust where a backing database or other single point of failure is likely to degrade to degrade before its fleet of application workers. Although rate limits are often implemented on a per-user basis, it's advisable in this case to implement a service-wide rate limit that roughly represents the floor of its maximum capacity and the maximum capacity of all its critical dependencies.

### Control Rods

Implement control rods that can be manually enabled or disabled by a human operator during emergencies.

The most simple control rod is one that enables full maintenance lockdown by having the service respond only with 503s ("service unavailable") instead of fulfilling any normal requests. This is of course highly disruptive to services that may depend on it, so more granular control rods can be implemented to help keep the platform healthy:

* One that allows a service's maximum rate limit to be scaled to a fraction of normal (e.g. 0.1x). This allows some traffic to succeed even while the platform as a whole continues to be generally degraded.
* One that allows particular IPs or user agents to be banned. Useful for temporarily disallowing other services that may be behaving pathologically due to a bug.
* A set that allow individual external service dependencies to be disabled. This can be used to keep some requests succeeding by disabling those that rely on an external service that's known to be completely degraded.

## Publish Playbooks

Write playbooks for services that are simple enough operators who aren't core contributors to use successfully. Keep them succinct so that they're easy to reference and resistant to bit rot.

---

Services should ship with playbooks that describe the basic structure of deployments along with the most important operations that may need to be run during maintenance or an emergency. It's useful to imagine a playbook's reader as someone who isn't a core contributor to the project so that in a particularly rough spot, the service can still be operated even if none of its normal shepherds are present.

Examples of what might be included in a playbook:

* The service's deployment structure: which servers it resides on, what its critical dependencies are, instructions for carrying out a deployment, etc.
* Links to important dashboards and other visibility tools.
* The names of any alarms that might be triggered by the service, and what a human operator should do to resolve them.
* A list of available control rods, a basic description of each, and how to activate them.

## Succinctness

Brevity is important for a few reasons:

* A production incident is a very stressful time during which it can be difficult to think clearly or deeply. Documents that are amenable to being skimmed are particularly valuable during such a time. Commands in them should be "clipboard friendly" so that users can copy and run them easily and safely.
* Documents that aren't in constant use tend to be susceptible to bit rot and quickly fall out of date. A succinct document is easier to update, and therefore less prone to this problem.

### Executable Playbooks

Playbooks are good, but executable playbooks are even better. Unfortunately, by definition playbooks are only referenced in unusual situations, and documentation that's not in constant use is incredibly prone to falling out-of-date.

Mature services should consider shipping their own set of operator tools that empower even a naive operator to resolve all but the most unusual problems. Unlike documentation, programmatic tools can be compiled and tested in CI which is an incredibly powerful tool for ensuring their correctness.

Some good practices:

* Make operator tools trivial to install so that an operator responding to an incident can access them as soon as possible. A package repository that's already widely distributed internally is a good option.
* Write good `--help` documentation that's useful even to a first time user. Include examples.
* Write tests and run them in CI. There's nothing worse than a tool that fails as soon as it's needed.
* Where possible, include tab completion and other niceties. The more discoverable and the _more quickly_ discoverable an interface is, the better.
