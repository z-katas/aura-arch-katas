# ADR: Scope of AI in Visitor Feedback — Real-Time Safety Triage Only

## Status
Accepted

## Context
The Visitors quantum absorbed the Feedback capability during [quantum identification](../README.md#architecture-quantum-identification), since feedback has low, steady traffic and no distinct scaling profile of its own. Feedback arrives both **anonymous** (post-visit QR point, no login) and **identified** (logged-in, via the Web/Mobile App) — see the [Visitors quantum architecture](../assets/visitors-quantum-architecture.png) and the [feedback triage/escalation sequence](../assets/visitors-quantum-sequence.png).

Free-text feedback is exactly the kind of open-ended input where a keyword filter under-performs and an AI model can genuinely help — the same reasoning that justified the Assistant in [ADR: AI Advisory Boundary for Ticketing and Family Pass Recommendation](ADR-007-visitors-ai-advisory-boundary.md). But the Analytics quantum already owns aggregation, theming, and sentiment mining across *every* quantum's data, per its [event-driven ADR](ADR-005-analytics-event-driven-style.md) — it is explicitly the estate's home for "read this data and tell us what it means," including feedback. Re-implementing that inside Visitors would duplicate a responsibility Analytics already has, and would fragment where "what visitors think" is analyzed.

There is, however, one thing bulk/batch analysis cannot do: catch a **safety-critical** complaint (an injury, a hazard, a frightening incident) in time to matter. A complaint like "there's broken glass near the piranha tank" sitting in an overnight batch job is a problem discovered too late.

We considered three alternatives:

1. **No AI in Visitors' feedback path at all** — Feedback Service just captures and forwards every event to Analytics, which does all analysis, including safety-relevant detection, on its own schedule. Simplest, but a safety complaint waits for a batch cycle before anyone sees it.
2. **Full sentiment/theme analysis inside Visitors** — build the same NLP capability Analytics already has, directly in this quantum, so Visitors can act on its own analysis. Duplicates Analytics' stated responsibility, and now two quanta must be kept correct and consistent on the same kind of insight.
3. **A narrow, real-time safety/urgency classifier in Visitors, everything else left to Analytics** — Visitors runs a lightweight classifier on each feedback submission as it arrives, whose only output is "urgent safety issue, yes or no." Routine feedback (the vast majority) is simply tagged and forwarded; no theming, no sentiment score, no summarization happens in this quantum.

## Decision
We adopt **narrow real-time safety triage in Visitors, with all other feedback analysis left to Analytics**.

Specifically:
- The **Feedback Service** captures every submission (anonymous or identified) and always forwards it to the Central Message Broker for Analytics — this happens regardless of what triage decides.
- A lightweight **Real-time Safety/Urgency Triage** classifier, called through the shared [Internal AI Gateway](ADR-002-external-ai-integration.md), runs on each submission's free text at intake time. Its only output is a binary urgent/routine classification — it does not score sentiment, extract themes, or summarize.
- An **urgent** classification publishes a separate, high-priority event straight to the Staffing/Maintenance quantum, reusing their existing [tiered human-in-the-loop dispatch](ADR-013-staffing-ai-dispatch-authorization.md) rather than Visitors inventing its own escalation path.
- A **routine** classification results in no special action beyond the normal tag-and-forward to Analytics; Visitors does not attempt sentiment or theme analysis on it.

## Consequences

**Positive:**
- **Speed where it matters** — a safety-relevant complaint reaches a human on the ground in near real time instead of waiting for Analytics' batch cycle, without requiring every quantum to build its own full-scale analysis stack.
- **No duplicated responsibility** — Analytics remains the single place theming/sentiment/trend logic lives and is maintained; Visitors' AI footprint stays small and single-purpose, which also keeps its blast radius small if the classifier is ever wrong.
- **Reuses existing patterns** — escalation rides on Staffing's already-reviewed dispatch authorization model instead of a new, unreviewed alerting path.

**Negative / trade-offs:**
- **Two places touch feedback** — a submission is both triaged locally (urgent/routine) and separately analyzed later by Analytics (theme/sentiment). This is intentional layering, not redundancy, but it means a developer reading only the Visitors code will not see the full picture of what happens to a piece of feedback.
- **Binary classification can be wrong in both directions** — a false negative delays a real safety issue to the batch path (acceptable: it still reaches Analytics); a false positive sends a routine complaint to Staffing/Maintenance as if urgent. Mitigation: bias the classifier toward over-flagging, consistent with the same "false alarm is cheaper than a missed one" principle already established for [Maintenance's animal-health monitoring](../design_docs/architecture-characteristics-styles.md).
- **Another Gateway consumer to monitor** — cost and failure modes for this classifier need to be tracked separately from the Assistant's, even though both share the same Gateway.
