# Glossary

| Term | Definition |
| --- | --- |
| **Architecture quantum** | A cohesive group of capabilities that shares similar scalability, availability, and change-cadence needs. This platform uses Visitor, Maintenance, Staffing, Analytics, and Marketing quanta. |
| **AI Gateway** | An internal facade that all AI-calling services talk to instead of a vendor SDK directly, so a provider swap, price change, or outage is a gateway config change rather than a code change. See [ADR: External AI Integration Strategy](../ADRs/ADR-002-external-ai-integration.md). |
| **Confidence score** | A model's self-reported certainty in a given output, attached to every AI-derived decision in this system so it can be gated, reviewed, and monitored — never presented as if it were ground truth. |
| **Drift** | A gradual decline in an AI feature's real-world accuracy or usefulness, distinct from an outright error — detected here via a rising override rate or falling confidence trend, not a single wrong answer. |
| **Edge** | Computing and sensing performed near the rides, enclosures, and other estate equipment rather than in the cloud. |
| **Event backbone** | The event-driven communication layer that distributes ticketing, telemetry, staffing, and operational events between quanta. |
| **Event storming** | A collaborative modelling technique used here to map actor actions and domain events before identifying services and boundaries. |
| **Fail closed** | The rule that when an AI component's confidence or retrieval is too low to act on, it escalates to a human with no invented output, rather than guessing — used explicitly in Maintenance's work-order guidance and Analytics' insight review. |
| **GenAI** | Generative artificial intelligence that produces content or recommendations, such as diagnostic guidance, recovery offers, or visitor assistance. |
| **Golden path** | The expected sequence of events when a process completes successfully, without exceptions or errors. |
| **HMW** | "How might we?": a framing format used to express the four AI-enabled automation opportunities. |
| **Human-in-the-loop (HITL)** | A design pattern where a human must confirm, correct, or approve an AI-drafted decision before it takes effect on the physical estate — always required for high-severity Staffing dispatch, and used with different tiers across every quantum. |
| **Human override rate** | The share of AI-derived decisions a human edits, rejects, or corrects, tracked per model version as this system's primary signal that an AI feature may be misbehaving in production. See [ADR: Production Monitoring & Drift Detection](../ADRs/ADR-001-ai-vendor-risk-and-monitoring.md). |
| **MQTT** | A lightweight publish/subscribe messaging protocol used by estate hardware to send sensor data over unreliable connectivity. |
| **NLP** | Natural language processing used to interpret visitor feedback and support sentiment analysis and recovery actions. |
| **Pareto footfall** | The planning assumption that a relatively small share of locations accounts for a disproportionately large share of visits; this informs staffing prioritisation. |
| **RAG** | Retrieval-augmented generation: an AI pattern that grounds diagnostic responses in information retrieved from the asset knowledge store. |
| **Store-and-forward** | An edge pattern that buffers events locally during connectivity gaps and forwards them when communication with the cloud returns. |
