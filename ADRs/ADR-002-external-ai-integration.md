# ADR: External AI Integration Strategy

## Status
Accepted

## Context
The estate brief (and the judges' criteria) requires us to architect for **AI volatility**: today's best model may be obsolete tomorrow, a provider may change pricing overnight, or a vendor may shut down. Hardcoding the Staff Service or its agents to a proprietary SDK (OpenAI, Anthropic, and so on) locks us to that vendor's schema and client. A price hike or outage would then mean rewriting core dispatch logic — downtime the Staffing quantum cannot afford, given [Fault Tolerance, Availability, and Responsiveness](../design_docs/architecture-characteristics-styles.md).

The GenAI Triage Dispatcher from [ADR: Authorization Model for AI Staff Dispatch](ADR-013-staffing-ai-dispatch-authorization.md) is the first consumer of this integration; a Shift Roster Optimizer (and any later staffing agent) must use the same path so a provider swap is an ops change, not a rewrite.

We considered three alternatives:

1. **Direct vendor SDK integration** — application code imports a provider SDK and builds requests in that vendor's schema. Fastest to ship, and it directly violates the volatility constraint.
2. **Open-source AI gateway / facade** — an intermediary (LiteLLM, a unified router, or a small custom facade) sits between estate services and external LLM providers. Routing, logging, keys, and failover live in infrastructure, not in Staff Service business logic.
3. **Multi-SDK wrapper inside application logic** — Staff Service itself hosts several SDKs behind a home-grown interface. Provider concerns leak into the staffing domain and become a permanent maintenance tax on every dispatch change.

## Decision
We adopt an **open-source AI gateway / facade layer**.

Specifically:
- All internal staffing services (GenAI Triage Dispatcher, Shift Roster Optimizer, and any later agent) talk **only** to the internal AI Gateway, using a single lowest-common-denominator schema (e.g. the OpenAI API shape).
- The gateway translates that schema to each provider, holds API keys, and applies failover policy (if Provider A times out, route the same prompt to Provider B immediately).
- Token use and cost are metered at the gateway so provider spend is visible in one place, not scattered across service logs.

## Consequences

**Positive:**
- **Vendor independence** — swapping or adding a model is a gateway config change, not a Staff Service deploy, which is how we meet the model/vendor-volatility constraint.
- **Failover** — a provider outage does not take down triage or roster optimization; the next configured provider receives the same standardized request.
- **Cost control** — centralized token monitoring makes a sudden price change visible and routable (shift traffic, or fail over) without hunting through application code.
- Business logic stays about incidents and deployment plans, not about provider SDKs.

**Negative / trade-offs:**
- **Loss of provider-specific features** — a unified schema will lag a vendor's newest proprietary API. Mitigation: use the gateway for all core operational flows (triage, routing, roster logic). Isolated bypasses are allowed only when a feature cannot be abstracted *and* has clear, documented ROI for the estate — each such bypass needs its own follow-up ADR.
- **Extra hop / extra service** — the gateway is another cloud component and adds a small amount of latency. Mitigation: deploy it in the same region/VPC as the central staffing services, and enable **semantic caching** on the gateway. An identical or highly similar incident can return a cached triage result without a provider call, which also cuts token cost.
- We accept that extra infrastructure in exchange for resilience, provider independence, and cost control — the alternative is rewriting dispatch code under an outage.
