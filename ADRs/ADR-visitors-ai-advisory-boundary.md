# ADR: AI Advisory Boundary for Ticketing and Family Pass Recommendation

## Status
Accepted

## Context
Per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md), the Visitors quantum's top driving characteristics are **Availability, Performance, and Scalability** (Concurrency also considered) — it is the estate's highest-traffic surface and must scale independently to carry 3x visitor growth. Ticketing is also, functionally, a transaction: check capacity, price, pay, issue. None of that is helped by non-determinism, and a purchase flow that *depends* on a model call would trade away the exact characteristic this quantum is scored on.

At the same time, our [AI use-case analysis](../design_docs/architecture-characteristics-styles.md#ai-use-cases) identified a genuine opening: a visitor describing their party in free text ("2 adults, 2 kids ages 5 and 10, we like animals") is an open-ended natural-language input that a fixed rule table struggles to cover as combinations grow, while an LLM handles it well. See the [Visitors quantum architecture](../assets/visitors-quantum-architecture.png) for where the **Family Pass and Wayfinding Assistant** and its **Rule-Based Pass Recommender** fallback sit relative to the deterministic Catalog → Capacity → Order → Payment → Issuance chain; the [sequence](../assets/visitors-quantum-sequence.png) shows the AI-confident and fallback branches side by side in the golden-path purchase flow.

The question this ADR answers is **how far AI is allowed to reach into the ticketing path**, not which model or prompt the Assistant uses.

We considered three alternatives:

1. **AI-driven checkout / dynamic per-visitor pricing** — let the model influence price or approve the order directly. Maximizes personalization, but makes a transaction non-deterministic and unauditable, and ties Availability to a third-party provider's uptime — unacceptable for this quantum's top characteristic.
2. **AI as a required step before purchase** — every visitor must go through the Assistant to get a recommended bundle before checkout is enabled. Ensures everyone gets a recommendation, but a slow or unavailable model now blocks a sale outright.
3. **AI as an optional, advisory overlay with a deterministic fallback** — the Assistant may suggest a bundle, but the deterministic Catalog → Capacity → Order → Payment → Issuance chain works identically whether or not the Assistant ever ran, and a low-confidence or timed-out AI call is answered by a local Rule-Based Recommender instead.

## Decision
We adopt **AI as an optional, advisory overlay** on top of a fully deterministic ticketing core.

Specifically:
- The **Assistant** only ever produces a *recommendation* (a suggested bundle plus its reasoning) back to the Web/Mobile App. It never writes an order, holds inventory, or touches payment — the visitor still confirms through the ordinary Catalog → Capacity → Order → Payment → Issuance path, which functions with or without the Assistant.
- The Assistant calls out through the shared [Internal AI Gateway](ADR-external-ai-integration.md) (the same vendor-agnostic facade used by the Staffing quantum's dispatcher), so a provider outage or swap is an ops change, not a Visitors redeploy.
- On timeout or low model confidence, the Assistant's caller falls through to a local **Rule-Based Pass Recommender** — plain conditional logic over party size, ages, and stated interests, with no external call — so a visitor is never blocked from buying a ticket by an AI outage.
- Dynamic, per-visitor pricing driven by the model is explicitly out of scope; pricing stays in the deterministic Catalog Service's rule set.

## Consequences

**Positive:**
- **Availability** — the ticketing core's uptime is decoupled from any AI provider's uptime; the highest-traffic, revenue-critical path cannot be taken down by a model outage.
- **Explainability and trust** — a visitor (or an auditor) can always tell which bundle came from a rule and which came from a model recommendation, and the recommendation carries reasoning rather than being a black-box price.
- **Vendor independence** — swapping or failing over the underlying model is contained to the Gateway, per the existing estate-wide pattern, rather than becoming a Visitors-specific integration to maintain.
- Matches this quantum's stated style (Microservices, Availability/Performance/Scalability-first) instead of quietly turning ticketing into an AI-dependent service.

**Negative / trade-offs:**
- **Two recommendation paths to maintain** — the Rule-Based Recommender must be kept reasonably competent on its own, since it is the answer a meaningful fraction of visitors will actually see (any time the Assistant is slow, unavailable, or unsure). Mitigation: treat it as a real fallback, not an afterthought — test it directly, not just as "the thing behind the AI."
- **Missed personalization on fallback** — a visitor who hits the fallback gets a coarser recommendation than the AI could have produced. Accepted: a good-enough deterministic suggestion beats a blocked or delayed purchase.
- **No AI-driven yield management** — we give up a potential revenue lever (dynamic pricing) in exchange for a fully auditable, always-available checkout. Revisit only if a future ADR can show the same guarantees hold with pricing in the loop.
