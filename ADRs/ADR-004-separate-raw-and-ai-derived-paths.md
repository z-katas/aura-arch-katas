# ADR: Separate Raw Telemetry Path from AI-Derived Insight Path

## Status
Accepted

## Context
Early versions of our [Analytics quantum architecture](../assets/analytics-quantum-architecture.png) routed the Staff Dashboard's live heatmap feed through the AI Analytics Agent, alongside AI-generated hotspot forecasts. On review, this conflated two fundamentally different kinds of data with different trust, latency, and explainability needs:

- **Raw operational data** (current footfall counts, queue lengths) — deterministic, low-latency, needs no explanation or human review.
- **AI-derived insight** (hotspot forecasts, deployment suggestions) — probabilistic, subject to the confidence-based review process defined in [ADR](ADR-001-ai-vendor-risk-and-monitoring.md), and must carry reasoning for explainability.

Routing both through the same component made it unclear to staff which numbers on their dashboard were "ground truth" versus "AI opinion" — a distinction our judges' evaluation criteria (explainability, validation of AI results) explicitly care about.

We considered three alternatives:

1. **Single merged feed through the AI Analytics Agent** — the early version described above. Simplest pipeline (one component produces everything the dashboard shows), but it is exactly what conflates deterministic counts with probabilistic forecasts, and it was the design we rejected on review.
2. **Separate pipelines, merged in the UI** — keep the live heatmap and the AI forecast as two backend feeds, but let the dashboard combine them into a single undifferentiated view (e.g. one "expected crowd" number). Fixes the backend coupling, but not the actual problem: staff still can't tell which number on screen carries AI uncertainty and which doesn't, so it doesn't satisfy the explainability requirement it was meant to fix.
3. **Two structurally separate feeds, visually distinguished end-to-end** — the live heatmap is published directly by the Stream Processor with no AI Analytics Agent involvement at any point; the hotspot forecast is published separately, carries a confidence score, and is labeled as such in the UI.

## Decision
The Staff Dashboard receives **two explicitly separate feeds**:
1. **Live heatmap feed** — published directly by the Stream Processor from aggregated raw telemetry. No AI Analytics Agent involvement, no confidence score, no review gate.
2. **Hotspot forecast & deployment suggestion** — published by the AI Analytics Agent, carrying the same confidence/reasoning structure used elsewhere in this quantum, and subject to the same low-confidence review gate before triggering a staffing nudge.

The dashboard UI must visually distinguish these two feeds (e.g. "current" vs. "forecast" labeling) rather than merging them into one undifferentiated view.

## Consequences

**Positive:**
- Staff can trust the live heatmap unconditionally, since it carries no AI-introduced uncertainty — directly supports the judge criterion of knowing what's deterministic versus AI-derived.
- The AI-derived forecast path can be monitored, reviewed, and rolled back (per ADR-004) independently of the raw data path — a bad model version never corrupts the raw operational view.
- Reduces the AI Analytics Agent's responsibility to only what actually needs intelligence (forecasting, trend detection), rather than acting as a pass-through for data it doesn't add value to — simpler agent, smaller blast radius per component.

**Negative / trade-offs:**
- Two feeds into one dashboard is marginally more integration work than one unified feed.
- Staff need a brief onboarding/UI cue to understand the "current vs. forecast" distinction; this is a UX cost worth paying for the trust and explainability benefit.
