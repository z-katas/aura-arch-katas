# ADR: Ride and Enclosure Event Delivery Strategy

## Status
Accepted

## Context
[ADR: Ride and Enclosure Feed Ingestion Strategy](ADR-010-maintenance-feed-ingestion.md) decided **what** leaves the estate: compact anomaly payloads and health / count summaries, not raw video or waveforms. This ADR decides **how those events reach the cloud** when Wi-Fi drops — the normal condition on the grounds, not an edge case.

Maintenance is driven by **Data Integrity** first ([architecture characteristics](../design_docs/architecture-characteristics-styles.md)). A ride-failure or animal-health event that exists only in a failed in-flight publish is a silent "healthy." That is worse than the cloud seeing the same event a few minutes late. The [Maintenance quantum architecture](../assets/maintenance-quantum-architecture.png) already shows an MQTT gateway on the edge and a Central MQTT Broker in the cloud; the [sequence](../assets/maintenance-quantum-sequence.png) is the outage path: queue on disk, flush on reconnect.

This is the same estate constraint as [ADR: Field Staff App Connectivity Strategy](ADR-014-staffing-field-app-connectivity.md), on a different hop: here the publisher is a ride or enclosure gateway, not a staff handheld.

We considered three alternatives:

1. **Fire-and-forget publish** — the gateway sends MQTT QoS 0 (or HTTP) and drops the payload if the central broker is unreachable. Simplest client. Any outage deletes the exact moments we instrumented the estate to catch.
2. **In-memory retry only** — the gateway keeps unacked messages in RAM and resends when the socket returns. Survives a short blip, loses the queue on a power cycle or a long dead zone, which is common around older ride structures and wet enclosures.
3. **Disk-backed store-and-forward at the edge gateway** — every accepted estate event is written to local durable storage first, then forwarded to the Central MQTT Broker when the uplink is up. Cloud engines see events in arrival order after an outage, but they do not lose them.

## Decision
We adopt a **disk-backed MQTT store-and-forward gateway** on the estate edge.

Specifically:
- Edge inference outputs are written to the gateway's local queue **before** they are considered published.
- The gateway forwards to the **Central MQTT Broker** using **QoS 1** (at-least-once) and drains the queue asynchronously when Wi-Fi returns.
- Cloud consumers (predictive wear, multimodal animal health, RAG maintenance copilot) subscribe only to the central broker. They never poll a ride or enclosure device.
- Duplicate delivery is possible (QoS 1). Consumers must be idempotent on event id.

## Consequences

**Positive:**
- **Data integrity** — a connectivity gap cannot erase a ride-failure or animal-health event. The payload waits on disk at the gateway.
- **Fits the MQTT-only constraint** — no second estate-wide radio, no "upload when convenient" side channel. The same path used while online is the path used after an outage.
- **Small queues stay drainable** — because ingestion already shrunk the payload, a multi-hour outage is a backlog of compact events, not hours of video.
- Completes the architecture picture: edge inference → local queue → central broker → cloud engines.

**Negative / trade-offs:**
- **Eventual consistency** — during an outage the Predictive ML and Health Engines work from the last synced events, not from "now." Mitigation: every payload carries an **event time** (when the edge model fired), not only an arrival time; cloud models order and window by event time. Operator surfaces show **last successful sync** so a keeper does not treat a stale "healthy" as current.
- **Burst on reconnect** — a long dead zone then a flood of queued events can look like a sudden outbreak. Mitigation: drain **severity-first** (failure / population anomaly ahead of routine summaries) and let consumers dedupe on event id.
- **Gateway disk and operations** — another thing to monitor (queue depth, disk full, poison messages). Accepted: that is cheaper than a silent miss on a heritage ride or a piranha enclosure.
- We trade a live, always-current cloud view for a durable one. For this quantum, late is acceptable; lost is not.
