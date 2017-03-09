# Microservices

- Introduction
  - [Terminology]()
  - [Practice Moderation]()
- Isolation
  - [Share Data, Not a Database]()
  - [Deploy Discretely]()
- Composition
  - [Use Common Protocols and Conventions]()
  - [Enforce Strong Contracts]()
  - [Execute Idempotently]()
  - [Be a Good Citizen]()
  - [Communicate Economically]()
- Resilience
  - [Use Common Patterns for Resiliency]()
  - [Ensure Correctness With Atomicity]()
  - [Stream State Changes]()
  - [Defend In Depth]()
  - [Exercise the Platform]()
- Operation
  - [Instrument, Observe, and Correlate]()
  - [Normalize Practices]()
  - [Use a Platform]()

## Introduction

### Terminology

This guide defines the following terminology for service-oriented architectures:

* A **service** is a software installation that runs and is deployed independently of other parts of the larger system. It is largely interchangeable with "microservice".
* A **component** is a software module with well-defined interface boundaries that could operate as a part of service or be broken out into its own service if it's deemed to be beneficial.
* A **platform** is a greater software system that makes up all of an organization's services and components.

### Practice Moderation

Service-oriented architectures can result in large organizational, logistical, and operational benefits, but have very real drawbacks. Wide reaching changes that touch many services gain significant overhead in orchestration and the system gets harder to reason about as its complexity is fundamentally increased. In many situations, monolithic architectures are more appropriate.

Practice moderation and objective thought when deciding whether to break out a service. Sketch out the API contract between itself and other components in the system. Thoroughly consider how likely those contracts are to stay roughly in the same shape over the long term. If major changes are likely, keep the monolith.

## Share Data, Not a Database

Sharing information between services by plugging a single database into multiple services is a tempting shortcut, but will ultimately lead to an architecture that's fragile and static. Avoid going down this road and share data via well-defined APIs instead.

---

Services with inevitably need to share information. It can occasionally be tempting to have databases communicate with each other by attaching a single database to more than a single service and have them all read and/or write out of it. For example, a billing service could reach into the database of other services to determine how users should be billed.

Although this architecture will probably come with the smallest capital investment, it will be a costly mistake over the long run as an architecture manifests that's fragile and difficult to change.

Take for example a fairly run of the mill migration where we drop a column that was previously used but is now no longer needed:

``` sql
ALTER TABLE users DROP COLUMN password_v1;
```

Normally, this could be accomplished relatively easily by deploying code changes to a service to make sure that it stops referencing the column, then simply running the migration. However, with multiple services connected to the database, care must be taken to deploy code changes to all of them before the migration can be run. Given that there's a reasonable chance that these services are owned across multiple teams, this will probably take much greater coordination overhead, and it be very tempting to just not make the change instead of making the effort.

The most common results will be either slowed product development because changes are difficult, or accumulation of technical debt as nothing gets cleaned up. An even riskier problem is that an unsafe change is deployed because database dependencies are opaque and unintuitive, resulting in the breakage of another service and leading to a production incident.

Services that detect their schema implicitly using a framework like Active Record can be an even bigger problem because they may rely on a certain schema even if it's not obvious from their application code.

Although we've used a migration as an example, it's noteworthy that the same applies to any kind of operational maintenance that's needed by the shared database. Failovers and credential rotations are just as difficult.

Instead, have services communicate with one another over well-defined APIs. Attached databases should be implementation details that are fully abstracted away services to the point where they're completely hidden from view.

## Execute Idempotently

Design API endpoints to be idempotent so that work can be safely retried in the event of a failure. Use idempotency keys where an idempotent design is not possible.

---

Messages between systems can fail at any time, leaving one system unsure of the state of the other.

Prefer idempotent API calls so that in the event of a failure, systems can safely retry operations.

Where idempotency is not possible, use a construct like idempotency keys to ensure exactly one delivery.

## Use Common Protocols and Conventions

Standardize on a single protocol to power all interservice communication and establish usage conventions within that protocol so that service APIs are consistent and predictable.

---

These days there are many plausible alternatives for communication protocols between services. GraphQL and GRPC are two more modern ones, while Thrift has a good track record and will probably always be around, and a strong argument can be made for basic HTTP or HTTP/2 in just how common and widely supported they are.

This guide won't champion a particular protocol, but does recommend that whatever it is, it's made standard across the platform. A common protocol means that APIs can be reused across services and that clients and other toolkits can be refined and shared.

Within the bounds of whatever protocol is selected, establish strong common conventions. For example, if HTTP is used, use something like the [HTTP API style guide]() to make sure that verbs, naming, and idioms are consistent across every service. This might be slightly less critical for an RPC-like protocol, but it's still a good idea to try and maintain consistent naming and structure within endpoints.

## Enforce Strong Contracts

Strongly define every service's interface and make sure that services test their own APIs for regressions. Ensure backwards compatibility to prevent breakages and avoid introducing private APIs.

---

The easiest way to break a platform is to have one service break the contract that another serivce is using to communicate with it. This can happen for a number of reasons, but a very common problem is that APIs are implicit and broken accidentally; for example, if an API call's response is a JSON blob and someone changes the type of one field from a string to an integer. Another common pitfall is that semantics around backwards compatibility aren't well understood, and an incompatible change is introduced inadvertently.

### API Specifications

APIs should be strongly defined by giving them a declarative description that dictates exactly what endpoints are available, what parameters they take, and what gets included in their responses. There are a number of competing description formats for API specifications, but a good bet right now might be [OpenAPI]() for example.

However, simply defining a specification isn't enough. Every service must be able to confidently claim that they adhere to their advertised API by ensuring that their implementations are constantly and automatically tested against it on every change.

Building a test suite that uses a tool like [Committee]() to test interfaces and running it in CI is an example of an appropriate solution for ensuring compliance to a specification.

### Backwards Compatibility

Introducing a change in an API that's not backwards compatible is an easy way of breaking integration inside of a platform. For example, one service might drop a field from an API call's response that another service expects to be there.

These are a few examples of rules for maintaining backwards compatibility:

* Don't require new parameters to API calls that weren't required before. Adding a new optional parameter is okay.
* Don't make validations on existing parameters to API calls more strict than they were before. Adding new parameters with stricter validation is okay.
* Don't drop fields from API responses. Adding new fields is okay.
* Don't change the type or formatting of fields in API responses. Adding a new (and nearly identical) field with a different type is okay.

Remember that although these rules are perhaps most application to RPC and RESTful APIs, they still apply to any medium that services use to communicate. For example, the format of messages that are published into something like a streaming API should be treated just as strictly.

### API Versioning

A common scheme for maintaining backwards compatibility is versioning an exposed API so that multiple versions are running simultaneously and services can cut over to an updated version when they're ready to do so.

In practice, versioning can sometimes be a useful technique, but tends to produce a signifcant amount of technical debt because version upgrades are very onerous and not executed in a timely manner. The result is that both the old and new API versions have to run side-by-side in production far longer than had been originally intended. Especially for internal APIs this is more operational overhead than it's worth and should be avoided.

Instead design APIs to be "rolling" so that changes are small and incremental. This has the effect of significantly reducing the upgrade effort for integrations and making them more likely to stay up to date. It also eases operational burden because only a single API version needs to be run and maintained.

### Private APIs

The longer a service is in operation, the more likely it'll be that at some point an involed party will suggest that it should support a one-off experimental API or a private API for limited use by another service.

Despite the best of intentions, these sorts of compromises that make it anywhere near production have a bad habit of becoming permanent and resulting in long-term operational and design debt. This isn't always a fait accompli, but is a frequent enough that is should e considered a probable result. In general, avoid introducing private and one-off APIs and instead work on integrating the necessary changes into the mainline API.

## Ensure Correctness With Atomicity

ACID transactions are the most powerful tool in existence for guaranteeing data integrity. Use an RDMS that provides ACID guarantees to ensure correctness within the service bounds.

---

Use them within the bounds of any particular service. Calls to external services can result in inconsistent distributed state, so wherever possible converge state between services asynchronously.

ACID.

Document Stores vs. RDMS.

## Stream State Changes

Streaming changes in state through a log-style stream is efficient provides ordering and inherent error correction. Consider using this technique over sending multitudes of synchronous service requests.

---

On a large enough scale, even the most reliable of networks will demonstrate a considerable level of background failure. Any API call between two services has the potential to fail, so it follows that the fewer we make, the more robust our system is likely to be.

We can accomplish this by _batching_ API calls so that instead of transferring state on a one-off basis as changes occur, we instead transmit it in bulk by moving a large number of changes in a single call. An effective pattern for this is for an emitting to publish a _log_ of state changes to something like a Kafka topic or Kinesis stream and have other services consume it. This also has the advantages of being very efficient from a resource consumption perspective and guaranteeing that consumers receive those changes in the order which they occurred.

A practical example of this are webhooks. While somewhat of a romantic idea in that they use basic HTTP requests to stream state, they also come with some serious drawbacks:

* There's one HTTP request per state change, and that proposition comes with all the overhead that you'd expect.
* A webhook that doesn't execute successfully needs to be retried by its emitter. This puts a huge burden on it to ensure that all the bookkeeping occurs correctly.
* Webhooks are inherently unordered. Even if a single webhook request starts before another, connection delay or slow responsiveness on the part of the receiver could mean it finishes processing after one that started later.

A streamed log has none of these disadvantages.

### Transactional Checkpoints

Emitting to a stream can be combined with checkpointing to an ACID database for a very powerful mechanism for guaranteed delivery. An emitter can ensure that everything it expects to emit is delivered at least one time:

``` ruby
loop do
  begin
    DB.transaction do
      events = DB[:events_to_emit].all
      emit_to_log(events)
      DB[:events].where('id in (?)', events.map { |e| e.id }).delete
    end
  rescue => e
    log "Error while emitting to stream: #{e}"
  end
end
```

A consumer can then use a similar mechanism to make sure that everything it expects to consume is handled at least one time.

## Defend In Depth

Security should be layered. Host a platform within an isolated VPN, use TLS to communicate between services within the VPN, and lock down access with service-specific credentials.

---

Platforms should be comprehensively secured by layering security so that even if one layer is compromised, the platform stays safe from attack.

Some examples:

* A platform should be isolated into its own VPN with a minimal set of rules for ingress and egress to represent how it needs to talk to users in the outside world.
* Services within the VPN should use TLS to talk to each other in case an attacker has compromised the VPN and is able to man in the middle some connections.
* Services should require that other services talking to them authenticate when they do so. This prevents an attack who has compromised the VPN from connecting to services within it.
* Secrets like database credentials should be encrypted at rest within a service. This prevents an attack who has compromised a single node from getting access to other critical services.

A good example of defense in depth is when [Gmail started encrypting email even when moving it within its own data centers](https://gmail.googleblog.com/2014/03/staying-at-forefront-of-email-security.html) after the Snowden revelations suggested that state-level actors may have compromised its perimeter.

## Exercise the Platform

Make sure that a platform is active at all times so that problems can be detected even if they came in on recent changes. If this is a non-production installations, use integration testing to verify it's working as expected.

---

Any system that's never used is likely to not work as expected when it's needed. It's a good idea to make sure that a platform has enough activity happening in it so to be reasonably confident that its working as intended.

A production platform might have enough baseline load on it to give reasonable reassurance, but it's very common when it comes to QA and staging platforms to see them essentially idle. It's there especially that it becomes very important to make sure that common paths are exercised automatically with simulated traffic or broad integration testing.

### Integration Testing

Put an integration testing framework into place that runs operations against a platform in a way that closely simulates what real users would do. By forcing end-to-end integration between all the relevant services within a platform, these sorts of tests allow regressions to be detected quickly and hopefully before users see them.

Integration tests are often so general that just knowing about a failure isn't very useful. It's key to make sure that your testing framework is smart enough not to just identify when a failure occurs, but be able to precisely identify the service causing the failure. By investing in this sort of accuracy an organization can save countless hours of engineering time because instead of pulling in all hands on deck when a failure occurs, only the relevant teams need be notified.

### Chaos Monkey

Exercising the normal operation of a platform is of paramount importance, but it's often worthwhile taking it a step further by also exercising its ability to tolerate fault and recover from errors.

Netflix coined the idea of a [Chaos Monkey](https://github.com/Netflix/SimianArmy/wiki/Chaos-Monkey) whose job it is to go through a platform and randomly introduce fault by terminating systems and performing other acts of mischief. Doing so allows backup systems a chance to come online and attempt to restore the platform's health.

The principle here is identical to the idea of putting standard load on a platform. Just as with the operations of normal paths, systems for error recovery are also unlikely to work unless they're tested regularly.

## Be a Good Citizen

Services should aim to be good citizens within the context of the wider platform by behaving responsibly with respect to degraded conditions. When observing failures in API calls to other services, use exponential backoff and random jitter to help with expedient recovery.

---

A baseline level of intermittent failure is expected in any kind of distributed system, so when an API call to another service fails, it's appropriate to retry after only a short waiting period in case it's a transitory problem. However, after observing multiple failures, the service should use _exponential backoff_ and _random jitter_ to ensure that it's not hammering a service that's already in trouble, because doing so could contribute to further degradation.

### Exponential Backoff

As API operations continue to fail, a service should use [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) by waiting an amount of time proportional to 2^_n_ before retrying them, where _n_ is the number of failures that have occurred. This helps to reduce load on the remote service to give it some breathing room to recover.

After an API call succeeds, the counter is reset and normal operations can continue.

### Jitter

Exponential backoff on its own does a lot to make a platform safer, but it's also a good idea to take the idea a step further by introducing some random "jitter" to any wait times.

If an abrupt problem in a service causes a large number of clients that are communicating with it to all fail close to the same instant, then even though they're practicing exponential backoff, their scheduled wait times might be close enough that each subsequent retry puts significant load on the degraded service. This is known as [the thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem).

An easy solution is to introduce randomness to those wait times. This causes the schedules of each client retrying to drift apart and space out the load on the degraded service, which will help contribute to its recovery.

## Communicate Economically

Communicate efficiency between services to keep the platform fast. Practice austerity in the total number of interservice calls made, pool connections to minimize overhead, and use efficient protcols and data transfer formats.

---

Speed is important and microservice architectures can introduce latency as more information is passed across networks. Use connection pools when communicating between services so that connections that were opened previously can be re-used.

Use connection pools for more efficient communication between services.

Efficient protocols.

GraphQL

### Connection Pooling

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

## Use Common Patterns for Resiliency

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

### Succinctness

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

## Use a Platform

Make the lives of service owners easier by having a delivery team provide the plumbing in the form of an easily deployable and fully featured platform.

---

Some features that a platform should provide:

* Abstraction from node-specific configuration like Puppet.
* Bootstraps for commonly deployed languages and frameworks like Ruby, Python, etc.
* Easy deployments.
* Secret management.
* Log rotation and unification.
* Asset compilation, distribution to CDN, and versioning.

<!--
> vim: set tw=20000:
-->
