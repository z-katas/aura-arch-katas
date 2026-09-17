# ADR: Event-Driven Architecture Style for the Analytics Quantum

## Status
Accepted

## Context
Per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md), the Analytics quantum's top driving characteristics are **Data Integrity**, **Interoperability**, and **Adaptability**. Analytics is not a single service with a fixed set of callers — it ingests from every physical data source on the estate (ticketing, ride/animal sensors, feedback) and is read by multiple independent consumers (Estate Owner Dashboard, Staff Dashboard, Personalization Recommender, and — per our footfall/staff-deployment use case — the Staffing quantum itself). New sensor types, new dashboards, and new downstream consumers are expected to be added over the estate's 3-year growth window without requiring changes to Analytics' internals.

We considered three alternatives:

1. **Synchronous request/response APIs** — each producer (ticketing, sensors, feedback) calls Analytics directly, and each consumer calls Analytics' API to read insights. Simple to reason about for a single call, but every new producer or consumer becomes a point-to-point integration; a producer's write path now depends on Analytics' availability, and a slow consumer (e.g. a dashboard doing a heavy query) can back-pressure the producers feeding it — the opposite of the Adaptability and Interoperability this quantum is scored on.
2. **Shared database access** — producers write directly to Analytics' database, and consumers query it directly rather than going through an API or event stream. Removes a layer of indirection, but couples every producer and consumer to Analytics' internal schema; a schema change to support a new sensor type or a new insight field risks breaking every existing consumer silently, since there's no published contract between them.
3. **Event-driven via a Central Message Broker** — producers publish events, consumers subscribe to them; neither side calls the other directly or shares a schema beyond the published event contract.

## Decision
We adopt an **event-driven architecture style** for the Analytics quantum (Central Message Broker + Stream Processor + AI Analytics Agent, all communicating via published events rather than direct synchronous calls or shared database access).

Specifically:
- All raw telemetry (footfall, ticketing, feedback) is published as events to the Central Message Broker, not written directly to a shared database by producers.
- The AI Analytics Agent and Stream Processor are independent **consumers** of these events; neither is a required dependency of the other.
- Downstream consumers (dashboards, Personalization Recommender, Staffing quantum) subscribe to published insight/forecast events rather than querying Analytics' internal stores directly.

## Consequences

**Positive:**
- **Interoperability** — new consumers (e.g. a future BI tool, or a new AI use case) can subscribe to existing event streams without any change to producers or to Analytics' internals.
- **Adaptability** — new physical data sources (e.g. a new sensor type) only need to publish to the broker in the agreed event schema; no coordination needed with every downstream consumer.
- **Data integrity** — the event log is the durable source of truth; a consumer outage (e.g. a dashboard being down) cannot cause data loss, since events remain on the broker until consumed.

**Negative / trade-offs:**
- Eventual consistency — dashboards and downstream consumers see analytics with some lag relative to when an event was produced, which is acceptable for popularity/trend reporting but must be kept in mind if a future use case needs near-instant consistency.
- Operational complexity — running and monitoring a message broker is more infrastructure than a simple request/response API, and event schema changes need versioning discipline to avoid breaking existing consumers.
- Debugging a multi-hop event flow (sensor → broker → stream processor → AI agent → consumer) is harder than tracing a single synchronous call chain; this is mitigated by the request/response logging already required in [ADR](ADR-001-ai-vendor-risk-and-monitoring.md).
