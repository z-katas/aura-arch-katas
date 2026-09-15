# ADR: Field Staff App Connectivity Strategy

## Status
Accepted

## Context
Per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md), the Staffing quantum's top driving characteristics are **Fault Tolerance**, **Availability**, and **Responsiveness**. Field staff work across a sprawling estate (40 vintage rides and 55 animal enclosures) with **patchy Wi-Fi**. They must still receive real-time dispatch orders (for example, dynamic crowd-control nudges) and log incident reports (animal-health anomalies, ride safety hazards) without data loss.

Standard synchronous web protocols (HTTP/REST) fail during network drops, which risks dropped incident logs and uncoordinated crowd management. Deployment suggestions are useless if they arrive late or are lost — the same reason this quantum is event-driven rather than request/response.

We considered three alternatives:

1. **Synchronous REST API over HTTPS** — mobile devices hit backend endpoints directly. Requests fail when Wi-Fi drops and require manual retries, which violates fault tolerance.
2. **Persistent WebSockets with server-side buffering** — needs an active bidirectional socket. A drop disconnects the client, and recovery means re-establishing session state, plus high battery cost and complex reconnect logic on handhelds.
3. **Offline-first client with MQTT QoS 1 store-and-forward** — a local SQLite store on the staff device caches incoming dispatch notifications and outgoing incident logs. An MQTT broker with QoS 1 (at-least-once) queues messages while the device is offline, decoupling application state from live connectivity.

## Decision
We adopt **offline-first client architecture with MQTT QoS 1 store-and-forward** for all field staff apps.

Specifically:
- Operational state is written first to a local embedded SQLite database on the handheld — not directly to a cloud API.
- Communication with the central backend goes through an MQTT client using **QoS 1** (at-least-once delivery) and a disk-backed local message queue.
- Incoming `Staff Deployment Plan` pushes and outgoing `Incident Log` writes both survive Wi-Fi dead zones and sync when connectivity returns.

## Consequences

**Positive:**
- **Fault tolerance** — incident logs and dispatch receipts are not lost when the device is in a dead zone; the queue drains on reconnect.
- **Availability** — staff can keep logging and reading last-known deployment state without a live cloud session.
- **Responsiveness** — the app stays usable from local state instead of blocking on HTTP timeouts.
- Fits the estate constraint of MQTT-capable hardware as the path from the grounds to the cloud, and matches the Staffing quantum's event-driven style.

**Negative / trade-offs:**
- **Delayed dispatch in deep dead zones** — staff may see a stale `Staff Deployment Plan` until they reconnect. Mitigation: edge gateways around high-density rides push low-bandwidth emergency pings to handhelds (local LoRaWAN beacons) even when estate Wi-Fi is down.
- **Sync conflicts** — two staff members can update the same `Incident Log` while both are offline. Mitigation: deterministic last-write-wins using monotonic vector clocks, resolved by the central Staff Service.
- Client complexity — the mobile app must own a local schema, queue, and reconnect/sync logic, rather than a thin REST client. Accepted in exchange for operational continuity and eventual consistency over strict real-time immediacy while roaming.
