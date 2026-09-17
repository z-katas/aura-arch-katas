# ADR: Ride and Animal Work Order Guidance Strategy

## Status
Accepted

## Context
Once a ride or enclosure event has been [ingested](ADR-010-maintenance-feed-ingestion.md) and [delivered](ADR-011-maintenance-event-delivery.md), someone on the grounds still has to act. Maintenance technicians and animal caretakers need **immediate, specific next steps** — especially on 18th-century rides and exotic species (including jumping piranhas), where a generic internet-trained model will be confidently wrong.

Maintenance is driven by **Data Integrity**, **Extensibility**, and **Deployability** ([architecture characteristics](../design_docs/architecture-characteristics-styles.md)). A hallucinated torque spec or a guessed veterinary dose is a wrong reading in procedural form. The estate also has proprietary manuals and vet records that a public model has never seen. The [Maintenance quantum architecture](../assets/maintenance-quantum-architecture.png) places a diagnostic layer between the cloud health / wear engines and the people who receive a Smart Work Order; the [sequence](../assets/maintenance-quantum-sequence.png) is retrieve-then-generate, then a handheld work order.

The question this ADR answers is **what a work order is allowed to be based on**, not which retrieval library we use.

We considered three alternatives:

1. **Un-grounded generation** — the copilot writes root cause and steps from the anomaly payload plus a general-purpose model. Fast to ship. Hallucinates on heritage mechanisms and exotic care; fails the requirement to verify AI-driven functionality before it affects physical operations.
2. **Static SOP lookup** — each anomaly code maps to a canned checklist. No hallucination, and it works for a handful of repeated ride faults. It cannot cover one-off 18th-century assemblies or species-specific vet notes, and every new manual is a code change rather than a corpus update (poor Extensibility).
3. **Retrieve estate documents, then generate** — at anomaly time, pull the relevant passages from vintage ride manuals and veterinary records, and only then draft the work order from those passages. The model is a composer over estate knowledge, not the source of it.

## Decision
We adopt **estate-grounded work order guidance**: the Maintenance Copilot may draft a Smart Work Order only from retrieved estate documents plus the triggering event.

Specifically:
- A vector store holds **chunked, versioned** vintage ride manuals and veterinary records — the estate's own corpus, not the open web.
- On an anomaly (early-failure trend, health / population flag), the copilot retrieves semantic context, then drafts a work order with **suspected root cause** and **step-by-step mitigation**.
- Every work order **cites the retrieved passages** (manual section, vet record id) so a keeper or technician can see what the draft is grounded in — the same explainability seam as raw-vs-AI elsewhere.
- If retrieval is empty or low-confidence, the copilot **does not invent a procedure**. It opens a generic escalate-to-human work order and attaches the anomaly payload only.
- The copilot calls models through the [internal AI gateway](ADR-002-external-ai-integration.md), so a provider swap does not rewrite diagnostic logic.
- RAG-over-a-vector-DB is the **current** way to implement that grounding. The contract is "no uncited procedure," not a specific RAG framework.

## Consequences

**Positive:**
- **Data integrity** — procedures for heritage rides and exotic animals come from the estate's manuals and vet records, not from a model's prior. A wrong citation is reviewable; a fluent guess is not.
- **Explainability** — staff can open the cited page instead of trusting a black-box paragraph, which is how we validate AI output before it becomes physical work.
- **Extensibility** — a new ride manual or updated piranha protocol is a corpus ingest, not a copilot redeploy. The cloud engines keep publishing the same event shapes.
- Completes the architecture: cloud engines → copilot + corpus → Smart Work Order → maintenance staff and caretakers. Matches the ~15% keeper-review path in the [cost model](../design_docs/cost-analysis.md): humans still confirm; they no longer start from a blank page.

**Negative / trade-offs:**
- **Corpus operations** — manuals and vet guidelines must be cleaned, chunked, versioned, and re-indexed when they change. Mitigation: treat the vector store as a **versioned artifact** (who uploaded, which revision is live, rollback to the previous index). That is the Deployability cost of this quantum, paid in document ops rather than in edge-model fleets.
- **Stale or missing passages** — retrieval can miss the right page, or serve an outdated one. Mitigation: low-confidence retrieval fails closed (escalate, do not draft steps); citations make a bad retrieve visible to the technician before they turn a wrench.
- **Latency and another moving part** — retrieve-then-generate is slower than a canned SOP. Accepted for this quantum: a correct procedure minutes after the event is better than an instant wrong one. The delivery ADR already made "late is acceptable; lost is not" the rule for the signal; the same rule applies to the guidance.
- We trade a standing ingest process for guidance we can stand behind on assets the public internet does not understand.
