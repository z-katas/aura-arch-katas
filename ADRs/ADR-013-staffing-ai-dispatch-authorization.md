# ADR: Authorization Model for AI Staff Dispatch

## Status
Accepted

## Context
The Staffing quantum uses a Generative AI Triage Agent to read incident logs, parse multi-modal MQTT sensor data, and draft routing orders onto the `Staff Deployment Plan` — at a crowd scale of up to 15,000 visitors/day. See the [Staffing quantum architecture](../assets/staffing-quantum-architecture.png) and [sequence](../assets/staffing-quantum-sequence.png). Per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md), Staffing is driven by **Fault Tolerance**, **Availability**, and **Usability**, but the estate brief also requires us to **verify AI-driven functionality** and detect misbehavior in production. GenAI is non-deterministic: a hallucinated reading or a misrouted emergency (vintage-ride breakdown, jumping-piranha anomaly, medical incident) must not autonomously redeploy the estate's emergency personnel.

This is the same "AI opinion vs. ground truth" seam called out for analytics-produced deployment suggestions in [ADR: Separate Raw Telemetry Path from AI-Derived Insight Path](ADR-004-separate-raw-and-ai-derived-paths.md). The MQTT push itself is the field delivery path defined in [ADR: Field Staff App Connectivity Strategy](ADR-014-staffing-field-app-connectivity.md).

We considered three alternatives:

1. **Fully autonomous AI dispatch** — the agent has full write access to the `Staff Deployment Plan` and pushes every incident immediately. A hallucination can misdirect medical or maintenance staff during a real crisis.
2. **Fully manual dispatch (AI advisory only)** — the AI never acts; managers draft and send every routing order. Safe, but it does not use AI to absorb the 3x visitor-growth load with a lean staff.
3. **Tiered human-in-the-loop authorization** — the AI auto-dispatches low-severity, predictive crowd nudges (e.g. "move two staff to the carousel"). For high-severity incidents it drafts the order and waits for a one-tap manager approval before the field notification is released.

## Decision
We adopt **tiered human-in-the-loop authorization**.

Specifically:
- The GenAI Triage Dispatcher classifies every incoming incident as low-severity or high-severity.
- **Low-severity** crowd adjustments write through to the `Staff Deployment Plan` and are pushed to field staff without a human gate.
- **High-severity** alerts (18th-century rides, exotic animals, medical needs) are sent to the Manager Dashboard as a proposal.
- The Staff Service **blocks the final MQTT push** to field devices until a manager explicitly approves that proposal.

## Consequences

**Positive:**
- High-stakes AI actions are explicitly verified before they move people — matches the judge criterion of validating AI results before they affect physical operations.
- Low-severity nudges still run autonomously, so the estate gets AI leverage on the high-volume, low-risk workload (crowd shaping) without a manager in every loop.
- Human accountability stays on the critical path for ride, animal, and medical incidents; a bad model version cannot silently empty a zone or mis-route keepers.

**Negative / trade-offs:**
- **Manager bottleneck** — if the on-duty manager is away from the dashboard, a critical dispatch sits pending. Mitigation: an automated escalation matrix. If a high-severity proposal is neither approved nor rejected within **45 seconds**, the same push is escalated to the mobile devices of all secondary senior staff on the estate.
- A human gate adds seconds (or, if escalation fires, up to the 45-second wait) on the emergency path. Accepted: we trade absolute automated speed for safety, accuracy, and a named human decision on anything that can harm visitors, animals, or heritage rides.
