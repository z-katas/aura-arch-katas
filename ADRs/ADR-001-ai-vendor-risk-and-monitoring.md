# ADR: Production Monitoring & Drift Detection for AI-Driven Functionality

## Status
Accepted

## Context
[ADR: External AI Integration Strategy](ADR-002-external-ai-integration.md) already answers *vendor* risk — a provider raising prices, degrading, or shutting down — by routing everything through an internal AI gateway with failover. That ADR does **not** answer a different question the brief asks explicitly:

> "How will you know if your AI-driven functionality starts misbehaving once in production?"

A gateway can be perfectly healthy — correct provider, low latency, normal cost — while the AI's *outputs* quietly get worse: the Analytics forecast starts missing real hotspots, the Staffing triage agent starts misclassifying severity, the Maintenance copilot starts citing stale manual pages, or the Visitors assistant starts recommending the wrong bundle. Unlike a deterministic bug, none of this throws an error. Verifying a deterministic function is a unit test; verifying a non-deterministic one requires an ongoing signal, because the same input can legitimately produce a different (and still valid) output tomorrow.

Every quantum already has a *piece* of this signal, added locally as each feature was designed:
- **Analytics** — [ADR: Dual Data Store](ADR-003-dual-data-store-strategy.md) stores each insight's confidence score and the human approve/correct outcome, specifically so the "override rate" can be tracked.
- **Staffing** — [ADR: Authorization Model for AI Staff Dispatch](ADR-013-staffing-ai-dispatch-authorization.md) requires manager approval on every high-severity proposal, which is itself a labeled correct/incorrect judgment on the AI's classification.
- **Maintenance** — [ADR: Work Order Guidance Strategy](ADR-012-maintenance-work-order-guidance.md) requires every drafted work order to cite the retrieved passages it's grounded in, and fails closed to a generic escalation when retrieval is empty or low-confidence.
- **Visitors / all quanta** — [ADR: Separate Raw Telemetry Path from AI-Derived Insight Path](ADR-004-separate-raw-and-ai-derived-paths.md) already established that AI-derived output must be visually and structurally distinguishable from ground truth, which is a prerequisite for measuring it at all.

What's missing is a **cross-cutting policy** that says: these signals are not incidental UX details, they are the estate's production monitoring strategy for AI, they apply the same way in every quantum, and they have explicit thresholds and a rollback path. Five other ADRs and the [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md) already link to this ADR by name — this is that ADR.

We considered three alternatives:

1. **Trust the vendor's own monitoring** — rely on the LLM provider's dashboards (token usage, error rates). This tells us the *model* is responding, not that its *answers* are still good for our use case; a provider outage is visible, a slow accuracy decay is not.
2. **Ad hoc, per-quantum logging** — let each team log whatever seems useful for their feature (which is roughly today's state — scattered confidence scores and review flags with no shared threshold or escalation path). Cheap, but nothing rolls up into a single "is AI still working" answer, and a regression in one quantum's monitoring approach doesn't teach the others anything.
3. **Centralized AI behavior monitoring, built on the human-override signal already required per quantum** — every quantum's AI-derived output already passes through some form of human confirm/correct/override (a keeper confirming a flagged animal, a manager approving a dispatch, a technician following a cited work order, a visitor accepting or ignoring a bundle recommendation). Treat the **rate and rolling trend of overrides** as the primary drift signal, standardize how it's computed and surfaced, and back it with scheduled offline evals for the (rarer) case where a model degrades in a way no human happens to correct.

## Decision
We adopt **centralized AI behavior monitoring built on the human-override signal**, with scheduled offline evals as a backstop, both surfaced through the same internal AI gateway used for [vendor routing](ADR-002-external-ai-integration.md).

Specifically:

- **Every AI-derived output carries a confidence score and a provenance flag** (already required per-quantum by the ADRs above) so it is structurally distinguishable from ground truth and from other AI outputs, per [ADR: Separate Raw Telemetry Path](ADR-004-separate-raw-and-ai-derived-paths.md).
- **Human override rate is the primary production signal**, computed the same way in every quantum: `overrides / total AI-derived decisions reviewed`, on a rolling 7-day window, tracked per model version.
  - **Analytics** — an "insight" that a manager edits or dismisses on the Estate Owner Dashboard counts as an override (uses the Events DB from [ADR: Dual Data Store](ADR-003-dual-data-store-strategy.md)).
  - **Staffing** — a high-severity proposal a manager rejects or edits before approving counts as an override; low-severity auto-dispatches are sampled and back-checked, not gated live.
  - **Maintenance** — a work order a technician marks "not applicable" or escalates past the cited guidance counts as an override.
  - **Visitors** — a recommended bundle the visitor doesn't purchase, when they go on to purchase a *different* bundle in the same session, counts as an override signal (weaker than the others, since "just browsing" is a confound — treated as directional, not gating).
- **A rising override rate, a drop in average confidence score, or a rise in "low-confidence/escalate" outcomes, each relative to that model version's own rolling baseline**, is what "misbehaving" means operationally in this architecture — not a single wrong answer, which is expected and tolerated by design.
- **Alert thresholds are quantum-specific but follow one rule: an override-rate increase of more than 50% relative to the trailing 30-day baseline pages the on-call engineer**, not just a dashboard color change — this is deliberately louder than a typical ops alert because a silently-degrading AI on the animal-health or dispatch path can cause real harm before a human happens to notice the pattern.
- **Scheduled offline evals** (a small held-out set of known-good cases per quantum, replayed weekly against the current model version) catch the case where overrides *don't* rise because the AI is now wrong in a way nobody happens to review that week — this is the backstop for quanta with low review volume, most relevant to Staffing's low-severity auto-dispatch path, which by design skips live human gating.
- **Rollback is a gateway config change, not a redeploy**: when either signal crosses threshold, the gateway pins the previous model/prompt version for that quantum while the regression is investigated — the same mechanism [ADR: External AI Integration Strategy](ADR-002-external-ai-integration.md) already uses for provider failover, reused here for quality failover.
- This monitoring gates the **maturity transitions** in the [phased AI-adoption roadmap](../README.md#roadmap) — a quantum does not move from a heuristic/rule-based Phase 0-1 to a trained-model Phase 2 until the candidate model beats the existing baseline on the same override-rate metric over a held-out period, not on a calendar date.

## Consequences

**Positive:**
- **Directly answers the judges' criterion** — we have a named, specific answer to "how do you know AI is working," not just a gateway for vendor risk.
- **Reuses infrastructure that already exists** — every ADR referenced above already produces the override/confidence data this policy consumes; this ADR is a naming and thresholding exercise, not a new system to build.
- **One rollback lever** — because quality failover reuses the same gateway mechanism as vendor failover, on-call staff have exactly one place to look and one action to take, regardless of whether the cause is a bad provider or a bad prompt/model version.
- **Makes the phased roadmap defensible** — "we'll train a model once we have enough data" now has a measurable exit criterion (beats the baseline on override rate) instead of being a vague future promise.

**Negative / trade-offs:**
- **The override signal is noisy at low volume** — a quantum with few AI-derived decisions per day (e.g. Visitors early on) will have a jumpy override rate that looks alarming from small-sample noise alone. Mitigation: alert on the rolling 7-day rate, not single-day spikes, and require a minimum decision volume before the alert is considered actionable.
- **Overrides can lag a real regression** — if reviewers are inattentive or rubber-stamp AI output, the override rate under-reports the true error rate. Mitigation: the scheduled offline evals exist precisely to catch this; they don't depend on reviewer attentiveness.
- **Some override signals are weak proxies** — the Visitors "different bundle purchased" signal conflates genuine mis-recommendation with visitors legitimately changing their mind. Accepted: treated as directional context, not a gating signal, until a stronger proxy (e.g. an explicit "not what I wanted" action) is added to that UI.
- **Extra discipline required at design time** — every new AI-derived feature must define its own override signal and confidence score up front, rather than being added as an afterthought. Accepted: this is a small design-time cost for a monitoring story that otherwise doesn't exist at all.
