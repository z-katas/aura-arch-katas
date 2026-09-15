# ADR: Ride and Enclosure Feed Ingestion Strategy

## Status
Accepted

## Context
The Maintenance quantum watches **40 vintage rides** (high-frequency acoustic and vibration telemetry) and **55 enclosures** (continuous camera and sensor feeds, including jumping piranhas) — see the [Maintenance quantum architecture](../assets/maintenance-quantum-architecture.png) and [sequence](../assets/maintenance-quantum-sequence.png). Per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md), this quantum is driven by **Data Integrity**, **Extensibility**, and **Deployability**: a missed or invented "healthy" reading is worse than a slow one, and new ride or enclosure types will appear over the 3-year growth window.

The estate constraint is **patchy Wi-Fi** with **MQTT as the only sanctioned path** from the grounds to the cloud. Streaming raw telemetry and video is not viable: the pipe cannot carry it, cloud ingest and storage cost would erase the [keeper-hour savings](../design_docs/cost-analysis.md) this use case is meant to create, and a wifi drop mid-stream would lose the exact moments we care about (a bearing going bad, a piranha count change).

The cloud still needs a signal: filtered ride anomalies for the predictive wear model, and compact health / count summaries for the multimodal animal-health engine and the RAG maintenance copilot. The question this ADR answers is **what is allowed to leave the estate**, not which model brand sits on the device.

We considered three alternatives:

1. **Cloud-side inference on the raw feeds** — every acoustic sample and camera frame is shipped to the cloud, which runs detection there. Fastest to iterate on models, but it assumes a fat, reliable uplink the estate does not have, and it makes inference cost scale with visitors and camera-hours rather than with incidents.
2. **Store-and-forward of the raw feeds** — edge gateways buffer video and telemetry to disk and dump them when wifi returns. Nothing is lost locally, but the first reconnect saturates MQTT, the cloud bill is unchanged, and keepers still wait for a bulk upload before they see an alert.
3. **Infer on the estate, publish only decision-ready payloads** — devices next to the ride or enclosure turn raw feeds into compact events (filtered anomaly payloads, text health and count summaries). The MQTT gateway store-and-forwards those small messages. The cloud models consume events, not video.

## Decision
We adopt **on-estate inference with selective MQTT publication**. Raw ride telemetry and enclosure video stay on the device. Only lightweight, schema'd events cross the wifi gap.

Specifically:
- Ride-side devices run small acoustic / vibration models and publish **filtered anomaly payloads** (not the waveform).
- Enclosure-side devices run vision models and publish **health and count summaries** (not the camera stream). Edge vision is biased toward **over-flagging** — a false "healthy" is worse than a false alarm, as already noted for this use case.
- The MQTT gateway **store-and-forwards** those payloads and syncs when wifi returns, matching the estate's MQTT-only constraint.
- Cloud services (predictive wear, multimodal animal health, RAG maintenance copilot) subscribe to those events. They do not pull raw estate media.
- TinyML (rides) and on-device VLMs (enclosures) are the **current** runtimes that implement this. They can be replaced without changing the ingestion contract — the ADR is the contract, not the chip.

## Consequences

**Positive:**
- **Data integrity under patchy wifi** — a dead zone queues a small MQTT payload, not an unreadable video dump. The moment of the anomaly is not discarded because the uplink was busy.
- **Bandwidth and cost** — the estate-to-cloud path stays inside MQTT. Cloud inference and storage scale with events (~15% of animals flagged for keeper review in the cost model), not with 55 continuous cameras.
- **Extensibility** — a new ride or enclosure type adds an edge model that emits the same event shapes. The cloud engines and copilot do not take a new media format.
- Matches the Maintenance architecture: Physical assets → edge inference → MQTT gateway → central broker → cloud engines / copilot → work orders for keepers and technicians.

**Negative / trade-offs:**
- **Deployability of models** — updating inference across ~95 locations is harder than shipping one cloud model. Mitigation: treat edge models as versioned fleet artifacts (staged rollout, ride vs enclosure channels, rollback to the previous version). This is the Deployability characteristic showing up as an ops cost, not a reason to stream raw video.
- **Upfront device cost** — inferencing hardware is more than a dumb MQTT sensor. Mitigation: the brief already budgets MQTT-capable estate devices; start the heavier vision boxes on high-risk assets (heritage rides, piranha enclosures) and expand, rather than instrumenting every location on day one.
- **Less raw media in the cloud** — a keeper or the copilot cannot freely rewind a day of video from object storage. Mitigation: on anomaly, keep a short **local ring buffer** of the triggering clip / waveform and upload *that* artifact with the event when wifi returns — evidence for the flagged 15%, not a 24/7 stream.
- Edge models will be wrong sometimes. Accepted: over-flag and let a keeper confirm, rather than under-flag and discover a sick animal or a failed ride too late.
