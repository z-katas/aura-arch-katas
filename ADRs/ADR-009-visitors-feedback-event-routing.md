# ADR: Feedback Event Routing — Anonymous vs. Identified

## Status
Accepted

## Context
Visitor feedback can be **anonymous** (a QR code at a ride or enclosure exit, no login) or **identified** (submitted in-app by a logged-in visitor, carrying a `visitor_id` from the Auth Service) — see the [Visitors quantum architecture](../assets/visitors-quantum-architecture.png) and the [sequence](../assets/visitors-quantum-sequence.png), which shows the anonymous-vs-identified branch on the final broker-to-Marketing hop. Both kinds are useful, but not for the same purpose or the same audience: Analytics wants the full volume of feedback to spot themes and sentiment trends across the estate, while Marketing wants feedback it can attach to a specific visitor to personalize outreach and retention offers, per the [personalization/retention use case](../design_docs/architecture-characteristics-styles.md#ai-use-cases), which already lists **Privacy** as an implicit requirement.

An anonymous complaint cannot be tied back to a visitor without re-identifying them from context — doing so defeats the point of offering an anonymous channel in the first place, and visitors who choose the QR path are making an implicit choice not to be individually tracked.

We considered three alternatives:

1. **Publish every feedback event identically to both Analytics and Marketing** — simplest to implement, but it silently defeats the anonymous channel: Marketing (or anyone building on its feed) could attempt to correlate an "anonymous" complaint back to a visitor via timestamp/location/context, which is exactly what a visitor choosing the QR point did not sign up for.
2. **Keep anonymous and identified feedback in entirely separate systems** — guarantees separation, but doubles the Feedback Service's write paths and gives Analytics two schemas to reconcile just to get one estate-wide view of "what visitors are saying."
3. **One Feedback Service, one event schema with an explicit anonymity tag, and routing rules enforced at publish time** — every feedback event carries a `visitor_id` field that is either populated (identified) or absent (anonymous); the publishing logic, not each downstream consumer, decides who is allowed to see which events.

## Decision
We adopt **one feedback event schema with anonymity enforced at the point of publication**, not left to each consumer to respect voluntarily.

Specifically:
- The **Feedback Service** tags every submission as anonymous or identified based on whether an authenticated session (via the Auth Service) was present at submission time. This tag is set once, at intake, and is not inferred or re-derived downstream.
- **All feedback events** (anonymous and identified) are published to the Central Message Broker for the **Analytics quantum**, which aggregates theme/sentiment trends across the full volume regardless of anonymity.
- **Only identified feedback events** (those carrying a `visitor_id`) are published to the **Marketing quantum**. Anonymous events are never routed there — there is no code path in Visitors that can hand Marketing an anonymous complaint.
- This routing decision lives in the Feedback Service / broker publishing logic, not in Marketing's consumer code — a downstream quantum cannot accidentally opt into data it isn't supposed to receive.

## Consequences

**Positive:**
- **Privacy by construction** — a visitor's choice to give anonymous feedback is respected structurally, not by convention; there is no event Marketing could subscribe to that would leak an anonymous visitor's identity.
- **Analytics still sees everything** — the estate does not lose signal on the (likely larger) anonymous feedback volume; theme and sentiment trends are computed across the full set.
- **Single schema, single service** — Visitors avoids running two parallel feedback stacks; the anonymity tag is the only structural difference between the two cases.

**Negative / trade-offs:**
- **Marketing sees a biased subset** — visitors who prefer to stay anonymous are systematically excluded from personalization-driving feedback, which may skew what Marketing believes visitors want. Accepted: personalization on identified visitors is the explicit use case; anonymous feedback still fully informs the separate, estate-wide Analytics view.
- **Enforcement is a single point of trust** — if the Feedback Service's publish-time routing logic has a bug, it is the only thing standing between an anonymous event and Marketing. Mitigation: this routing rule should be covered by its own contract test (an anonymous event must never appear on the Marketing-facing topic), not just left to code review.
- **No retroactive re-routing** — a visitor cannot later "attach" an old anonymous submission to their identified account under this model; that would require a new, explicit product decision, not a quiet backend change.
