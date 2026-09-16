# HMW use AI to turn first-time visitors into repeat visitors

## Primary Goal

**HMW use AI to turn first-time visitors into repeat visitors** — personalization and targeted marketing that drive return visits, so the estate grows revenue without relying purely on new-visitor acquisition?

Quantum: **Marketing**. Marketing is a thin, event-driven consumer of published Visitor and Analytics events, not a new subsystem — the sketch below is the intended container view for this use case. Driving characteristics and style are in the [architecture characteristics worksheet](../design_docs/architecture-characteristics-styles.md#marketing-quantum-worksheet).

## Key screen

![Come back to the estate](../assets/ux-07-return-visit-offer.png "Return offer — event-driven, dismissable")

Twelve days after Elena Hart's first visit, she's offered a 20% Family Day Pass for the following Saturday, with the reasoning shown inline: her children asked for the jumping piranhas again, the carousel lawn was her second stop, and the typical return window for a first-time visitor is 21 days (she's inside it). Three things about this screen are architecturally deliberate, not just UX polish:

- **"Dismissing this does not require a Marketing redeploy. The campaign rule stays; your preference is just another event."** — a rejected offer is data, not a code change.
- **The campaign card names its own inputs and outputs explicitly**: trigger (`VisitCompleted` + no return in 21 days), reads (visit history from the data lake), publishes (`OfferIssued` / `OfferDismissed`), and channel (in-app now, email/SMS pluggable later) — this is the [ADR: Event-Driven Architecture Style](../ADRs/ADR-analytics-event-driven-style.md) pattern applied to Marketing instead of Analytics.
- **"Personalization Recommender is a Marketing consumer of published insight, not a query into Analytics' stores."** — the footnote is the architecture: Marketing never reaches into Analytics' internals, only its published events.

## High-level solution approach

![Marketing quantum architecture](../assets/marketing-quantum-architecture.png "Marketing quantum — return visitor personalization")

- **Adaptability and Interoperability are the top driving characteristics** ([architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md)) because campaign rules, audience segments, and delivery channels are expected to change often — an event-driven style lets Marketing subscribe to Visitor and Analytics events and add or swap delivery channels without touching Visitor or Analytics internals, and without a Marketing redeploy every time a rule changes.
- **Marketing reads, it never writes into another quantum's store.** Visit history comes from the data lake via a published-event contract, the same boundary [ADR: Separate Raw Telemetry Path from AI-Derived Insight Path](../ADRs/ADR-separate-raw-and-ai-derived-paths.md) establishes for Analytics' own consumers.
- **The Personalization Recommender calls the shared Internal AI Gateway** ([ADR: External AI Integration Strategy](../ADRs/ADR-external-ai-integration.md)) for the "why you're seeing this" reasoning text — same vendor-failover and cost-control guarantees as the Visitors and Staffing quanta, no Marketing-specific provider logic.
- **A dismissed offer is a first-class event (`OfferDismissed`), not a UI-only action.** This is what makes the override signal for this quantum (see below) possible at all — if dismissal weren't published, there'd be nothing to measure.

## Golden path

1. Visitors quantum publishes `VisitCompleted` to the Central Message Broker at the end of a visit.
2. Marketing's Campaign Engine, subscribed to that event type, checks the rule ("no return in 21 days") against the data lake's visit history.
3. If the rule fires, the Personalization Recommender reads that visitor's specific visit history (which rides/enclosures, party composition) and drafts a targeted offer with cited reasoning.
4. The offer publishes to the visitor's chosen channel (in-app shown here; email/SMS are the same event, different channel adapter).
5. Visitor accepts (→ hands off to Ticket Service, same booking path as their first visit) or dismisses (→ `OfferDismissed` published, no ticket created, campaign rule itself unchanged).

## Implementation details

- **Cold-start is explicit in the copy, not hidden**: "First visit, no return yet — typical return window is 21 days" is shown as reasoning, not suppressed. The system is honest that this visitor doesn't have enough history yet to justify a highly-tailored offer beyond "your own last visit."
- **Channel is a swap, not a rewrite**: because the trigger/audience/recommendation logic is decoupled from the channel adapter, adding SMS delivery is additive — no change to the Campaign Engine or the Personalization Recommender.

## How this is monitored in production

Per [ADR: Production Monitoring & Drift Detection](../ADRs/ADR-ai-vendor-risk-and-monitoring.md), `OfferDismissed` (as a fraction of `OfferIssued`) is Marketing's override signal — a rising dismissal rate for a given campaign or recommendation version is the drift signal, tracked the same way Analytics tracks insight overrides and Staffing tracks dispatch corrections. It carries the same caveat noted for the Visitors use case: dismissal can mean "the recommendation was wrong" or "not interested in returning right now for unrelated reasons," so it's directional, not a hard gate, until a stronger explicit reason-for-dismissal signal is added to the UI.

## Phased rollout for this use case

Per the [phased AI-adoption roadmap](../README.md#roadmap), personalization is one of the clearest cold-start cases in the whole system, because the data needed is inherently visitor-specific and slow to accumulate:
- **Phase 0**: no personalization — every first-time visitor gets the same generic "come back" offer, if any.
- **Phase 1**: simple rule-based segmentation (e.g. "visited in the last 30 days" vs. not), no ML.
- **Phase 2**: once enough visitors have **multiple recorded visits** to support a recommendation model — which itself takes several months estate-wide, since it requires visitors to actually return at least once before their pattern is learnable — introduce ML-driven personalization.
- **Maturity is partly per-visitor, not just estate-wide**: even once the estate-wide model is trained, a given visitor still gets the generic Phase 0/1 offer until *they individually* cross the repeat-visit threshold. A new visitor next year starts at Phase 0 regardless of how mature the estate's overall model is.

## Limitations

- **Adaptability cuts both ways**: campaign rules that are easy to add are also easy to accumulate into an unmanaged pile over time — needs a rule-retirement/audit process that isn't yet designed.
- **Privacy**: personalization requires identified visit history, which is why the Visitors quantum's message broker explicitly routes only "identified feedback + visit history" to Marketing — never raw feedback text, and never anonymous QR feedback (see [`hmw-01-ticket-pass-assistant.md`](hmw-01-ticket-pass-assistant.md)).
