# Microservices

## Practice Moderation

Before breaking out a new service, sketch out its API and the contract between itself and other components in the system. Thoroughly consider how likely those contracts are to stay roughly in the same shape over the long term. If major changes are likely, keep the monolith.

## Share Data, Not A Database

Never share an internal data schema between services because it makes changes to that schema slow, difficult, and dangerous.

Instead, have services communicate with one another over well-defined public interfaces.

## Idempotency Is the Key

Messages between systems can fail at any time, leaving one system unsure of the state of the other.

Prefer idempotent API calls so that in the event of a failure, systems can safely retry operations.

Where idempotency is not possible, use a construct like idempotency keys to ensure exactly one delivery.

## Strong Contracts

Regardless of the technology in use to communicate between services, make sure that interfaces are strongly defined and protected against accidental changes. Regressions in an API can easily break the integration between two components.

Communication via constructs like event streams and other real time APIs counts! Messages that are published into a stream should be as concretely defined as any API.

## The Log

Every request between two services is a potential failure.

Where a non-interactive interface is possible, batch up communication by having services stream their state to each over a construct like a Kafka topic. This method is more robust because fewer messages need to be passed, and allows consumers to ingest changes more economically because changes are grouped.

## In-service Atomicity to Ensure Correctness

ACID transactions are the most powerful tool in existence for guaranteeing data integrity.

Use them within the bounds of any particular service. Calls to external services can result in inconsistent distributed state, so wherever possible converge state between services asynchronously.

## Defense In Depth

Security should be layered. Services should use a secure protocol like TLS to talk to each other, even if they're known to be isolated in something like a VPC.

## Connection Pooling

Speed is important and microservice architectures can introduce latency as more information is passed across networks. Use connection pools when communicating between services so that connections that were opened previously can be re-used.

This is especially crucial for secure connections which have traditionally required three round trips to construct, adding a lot of additional latency (although TLS 1.3 is likely to bring this down to one round trip).

## Consistent Tooling

Services being deployed separately shouldn't mean that each one should be its own special snowflake.

Use a single protocol for intercommunication. Standardize on languages, frameworks, and even libraries. Use the same database (Postgres) everywhere. Use common services for purposes of deployment, logging, error reporting, metrics, and CI everywhere. Use linters to enforce conventions.

# Zero Downtime Deployments

Like the deployment of a single component should maintain compatibility with the code that is already running, the deployment of any component should be compatible with the overall service and not rely on lockstep deployments.
