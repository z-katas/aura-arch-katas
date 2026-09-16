# HMW use AI to make buying tickets effortless

## Primary Goal

**HMW use AI to make buying tickets effortless** — assisted ticket purchase, family pass recommendations, and in-park wayfinding, so visitors spend less time figuring out logistics and more time enjoying the estate?

Quantum: **Visitors**. See the [architecture](../assets/visitors-quantum-architecture.png) and [golden-path sequence](../assets/visitors-quantum-sequence.png) diagrams.

## Key screen

![Book your visit](../assets/ux-01-ticket-pass-assistant.png "Book your visit — pass assistant recommendation")

A party of four tells the assistant who's coming and what they want to do, in their own words ("first time with the kids... jumping piranhas and a couple of gentle rides"). The assistant recommends a Family Day Pass with the reasoning shown inline, and a visible price comparison against buying four individual tickets. Two things are deliberate about this screen, both directly reflecting how the quantum is built:

- **The recommendation is advisory, never a charge.** The footnote is explicit: "Payment is not taken until you confirm. The Ticket Service issues the pass; the assistant does not charge you." This mirrors [ADR: Separate Raw Telemetry Path from AI-Derived Insight Path](../ADRs/ADR-004-separate-raw-and-ai-derived-paths.md) — AI-derived output (the recommendation) is structurally and visually separated from the deterministic transaction (the actual ticket issuance).
- **"Choose a different ticket" is a first-class action, not an escape hatch buried in a menu.** The alternatives (individual tickets, grounds-only) are priced and shown right below the recommendation, not hidden.

## High-level solution approach

![Visitors quantum architecture](../assets/visitors-quantum-architecture.png "Visitors quantum architecture")

- The **Ticketing and Family Pass core** (Ticket & Pass Catalog, Capacity & Inventory, Order & Checkout) is a deterministic microservices path that works with zero AI involvement — a visitor can walk up to an on-site kiosk, pick a pass from the offline-cached estate map, and check out. This is the estate's ground truth and its fallback.
- The **AI Advisory Overlay** (Family Pass and Wayfinding Assistant) sits beside that core, never inside it. It reads free-text party/intent, calls the shared **Internal AI Gateway** ([ADR: External AI Integration Strategy](../ADRs/ADR-002-external-ai-integration.md) — the same gateway the Staffing quantum's dispatcher uses), and returns a recommended bundle with cited reasoning.
- **On timeout or low confidence, the assistant falls back to a Rule-Based Pass Recommender** — a simple party-size/age → bundle lookup — rather than blocking the purchase or guessing. This fallback is not a stopgap; per the [phased AI-adoption roadmap](../README.md#roadmap), it is the intended **Phase 0** implementation, since a party-composition → bundle mapping needs no estate-specific training data to be correct on day one.
- **Real-time Safety/Urgency Triage** on the post-visit feedback path (a lightweight classifier, not the same model as the recommender) flags urgent safety feedback for immediate routing to Staffing/Maintenance, separately from routine feedback that flows to Analytics for sentiment/theme mining.
- Everything the assistant and the triage classifier produce publishes onto the **Central Message Broker**, which is how Analytics (footfall/ticketing events), Marketing (identified visit history only, never raw feedback text — privacy boundary shown explicitly in the diagram), and Staffing/Maintenance (urgent flags only) each get exactly the slice of data they need without coupling to Visitors' internals.

## Sequence Diagram

![Visitors quantum sequence](../assets/visitors-quantum-sequence.png "Ticket & Family Pass purchase — golden path")

1. Visitor solos in natural language ("2 adults, 2 kids ages 5 & 10, love animals") → Family Pass & Wayfinding Assistant → Internal AI Gateway.
2. **If the gateway is available and confident**, it returns a recommended bundle with cited reasoning.
3. **If the gateway times out or returns low confidence**, the assistant falls back to the Rule-Based Recommender automatically — the visitor never sees an error, only a (slightly less tailored) recommendation.
4. Visitor requests the bundle → Ticket & Pass Catalog Service returns pricing → Capacity & Inventory Service reserves the slot (optimistic concurrency; a conflict here re-offers an alternative slot or waitlist, shown as an explicit branch on the sequence diagram) → Order & Checkout Service charges via Payment Gateway → ticket/pass issued.
5. Separately, visitor feedback (logged-in or anonymous QR scan) is classified for urgency in real time; urgent flags route to human dispatch, routine feedback publishes to the Central Message Broker tagged by identity status.

## Implementation details

- **Fallback-first design**: the Rule-Based Recommender is not a degraded experience — it is deliberately kept as a **permanent** fallback (see the [phased roadmap](../README.md#roadmap)), not something removed once the LLM path matures. A recommendation engine with 100% uptime and an LLM overlay with 100% uptime achieve the same reliability as one with 100%, because the two never fail in the same way at the same time.
- **Shared gateway, no Visitors-specific AI logic downstream**: the Internal AI Gateway is the same component the Staffing quantum's triage dispatcher uses ([ADR: External AI Integration Strategy](../ADRs/ADR-002-external-ai-integration.md)). A provider swap, price change, or outage is handled once, centrally, and both quanta benefit without either owning provider-specific code.
- **Privacy boundary is structural, not policy**: the architecture diagram shows the Central Message Broker fanning out three different slices to three different consumers — Analytics gets footfall/ticketing/sentiment signal, Marketing gets identified visit history *only* (no raw feedback text), Staffing/Maintenance get urgent safety flags *only*. This means a bug in Marketing's code cannot leak raw feedback, because the data was never routed there in the first place.

## How this is monitored in production

Per [ADR: Production Monitoring & Drift Detection](../ADRs/ADR-001-ai-vendor-risk-and-monitoring.md), the Visitors quantum's override signal is the weakest of the four use cases (recommending a bundle the visitor doesn't buy, when they buy a *different* one in the same session) — genuinely changing your mind and a bad recommendation look the same from this signal alone. It is treated as **directional context, not a gating metric**, until a stronger explicit signal ("not what I wanted") is added to the "Choose a different ticket" action. What *is* gating: a drop in the recommendation's average confidence score, which pins the gateway to the last known-good prompt/model version while the regression is investigated.

## Limitations

- **Cold start**: on day one, there is no data on which recommendations get accepted, so there's nothing yet to few-shot or threshold-tune the prompt against. This is expected — see Phase 0/1 in the [phased roadmap](../README.md#roadmap).
- **Weak override signal**: as noted above, "visitor bought something else" is a noisy proxy for "the recommendation was wrong." Not fixed by more data alone — needs a UI change to get a cleaner signal.
- **Free-text intent can be ambiguous or contradictory** ("first time" but party has clearly visited before per account history) — the assistant does not currently reconcile stated intent against known visit history; it takes the current session's input as authoritative.
