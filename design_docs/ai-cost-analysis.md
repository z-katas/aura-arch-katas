# AI Cost Analysis

Covers token budgets, model assignments, and projected AI spend across all quanta.
For business/operational cost savings (keeper labour vs AI), see [Cost Analysis](cost-analysis.md).

All pricing is illustrative — sourced from publicly listed rates as of mid-2025 and subject to change. The gateway's centralised token metering is the live source of truth. **The architecture is provider-agnostic** — model selection per tier is a gateway config decision, not a code change.

---

## Model Tiers

Three tiers assigned per call type. Switching a tier's model = **gateway config change only**, no service redeploy.

| Tier | Target use |
|---|---|
| **Fast** | High-volume, low-stakes, latency-sensitive (<500ms) |
| **Standard** | Moderate volume, output is human-reviewed before action |
| **Careful** | Low-volume, safety-critical, large RAG context (>2K tokens) |

---

## Provider Comparison by Tier

Approximate public pricing as of mid-2025. Evaluate on **estate-specific prompts** before selecting — benchmarks on generic tasks are not a proxy for estate domain fit.

### Fast Tier

| Provider | Model | Input ($/1M tokens) | Output ($/1M tokens) | Notes |
|---|---|---|---|---|
| Anthropic | Claude Haiku 3.5 | $0.80 | $4.00 | Strong instruction following |
| Google | Gemini 2.0 Flash | $0.10 | $0.40 | Cheapest option; good for classification |
| Meta (via Groq/Together) | Llama 3.1 8B | ~$0.06 | ~$0.06 | Open-weight; self-hostable on-prem |
| Mistral AI | Mistral Small | $0.10 | $0.30 | EU-hosted option; GDPR-friendly |
| OpenAI | GPT-4o-mini | $0.15 | $0.60 | Reliable baseline; wide ecosystem |

### Standard Tier

| Provider | Model | Input ($/1M tokens) | Output ($/1M tokens) | Notes |
|---|---|---|---|---|
| Anthropic | Claude Sonnet 4 | $3.00 | $15.00 | Strong reasoning and tool use |
| Google | Gemini 1.5 Pro | $1.25 | $5.00 | Long context; cost-efficient at volume |
| Meta (via Together) | Llama 3.1 70B | $0.59 | $0.79 | Open-weight; good for structured output |
| Mistral AI | Mistral Large | $2.00 | $6.00 | EU-hosted; strong multilingual |
| OpenAI | GPT-4o | $2.50 | $10.00 | Broad capability; vision support |

### Careful Tier

| Provider | Model | Input ($/1M tokens) | Output ($/1M tokens) | Notes |
|---|---|---|---|---|
| Anthropic | Claude Opus 4 | $15.00 | $75.00 | Strongest reasoning; highest cost |
| Google | Gemini 1.5 Pro (128K ctx) | $3.50 | $10.50 | Cost-effective at long RAG context |
| Meta (via Together) | Llama 3.1 405B | $3.50 | $3.50 | Open-weight; self-hostable |
| OpenAI | GPT-4o (full) | $2.50 | $10.00 | Reliable; vision useful for maintenance |

> **Careful tier fallback is always human escalation — not a cheaper model.** Safety-critical paths (work order copilot) fail closed to a human, not to a lower-grade LLM.

---

## Model Assignment per Quantum

| Quantum | Call type | Tier | Fallback (if LLM unavailable) |
|---|---|---|---|
| Visitors — ticket assistant | Bundle recommendation | **Fast** | Rule-based recommender |
| Visitors — feedback triage | Safety classification | **Fast** | Flag ALL as urgent (fail-safe) |
| Analytics — insight generation | Forecast narrative + confidence | **Standard** | Skip publish; retry next cycle |
| Staffing — triage dispatcher | Severity classification + routing | **Standard** | Rule-based severity tiers |
| Maintenance — work order copilot | RAG-grounded cited steps | **Careful** | Human escalation (fail-closed) |
| Marketing — personalisation | Return-offer recommendation | **Fast** | Static offer rules |

---

## Token Budget per Call

| Quantum / call type | Input tokens | Output tokens | Total | Notes |
|---|---|---|---|---|
| Ticket assistant | 500 | 200 | **700** | Visitor intent + product catalogue |
| Feedback triage | 200 | 20 | **220** | Short free-text → binary label |
| Insight generation | 1,000 | 300 | **1,300** | Telemetry summary + narrative |
| Triage dispatcher | 800 | 400 | **1,200** | Incident + venue context |
| Work order copilot | 2,500 | 600 | **3,100** | Anomaly + retrieved manual passages (RAG) |
| Personalisation | 400 | 150 | **550** | Visit history + offer catalogue |

---

## Monthly Cost Projection

Visitor scale: **1x = 5,000 visitors/day → 3x = 15,000 visitors/day** (3-year target).

Ranges reflect cheapest vs mid-range provider choice per tier (see provider tables above).

| Quantum / call type | Tier | Calls/day (1x → 3x) | Monthly cost 1x | Monthly cost 3x |
|---|---|---|---|---|
| Ticket assistant | Fast | 2,000 → 6,000 | $1 – $18 | $4 – $54 |
| Feedback triage | Fast | 1,000 → 3,000 | $0.30 – $3 | $1 – $9 |
| Insight generation | Standard | 100 → 100 | $4 – $24 | $4 – $24 |
| Triage dispatcher | Standard | 30 → 90 | $1 – $7 | $4 – $22 |
| Work order copilot | Careful | 20 → 60 | $11 – $50 | $32 – $149 |
| Personalisation | Fast | 200 → 600 | $0.20 – $2 | $0.50 – $5 |
| **Total (gross)** | | | **~$18 – $104/month** | **~$45 – $263/month** |
| **Total (with 30% cache)** | | | **~$13 – $73/month** | **~$32 – $184/month** |

The low end assumes open-weight (Llama) or cheapest API (Gemini Flash) for Fast/Standard tiers. The high end assumes premium closed-source across all tiers. Most realistic deployments land in between.

**Cache:** Semantic caching on the gateway applies to high-repetition paths (ticket assistant, feedback triage). Estimated 30% hit rate at 1x; rises toward 40% at 3x as prompt patterns stabilise.

---

## Cost Control Levers

| Lever | Where it applies | Effect |
|---|---|---|
| Semantic cache (gateway) | Ticket assistant, feedback triage | 30–40% call reduction |
| Switch to cheaper provider (gateway config) | Any tier | No service change; monitor override rate post-switch |
| Self-host open-weight model | Fast / Standard tiers | Eliminates per-token cost; adds infra overhead |
| Prompt compression | Work order copilot | Smaller RAG window → lower Careful-tier spend |
| Batch insight runs off-peak | Analytics | No latency impact; some providers charge less off-peak |

---

## Switching a Model or Provider

1. Update the tier→model mapping in gateway config (e.g. `FAST_MODEL=gemini-2.0-flash`).
2. Deploy gateway config — no application service redeploy.
3. Monitor override rate ([ADR-001](../ADRs/ADR-001-ai-vendor-risk-and-monitoring.md)) for 7 days; rollback if rate rises >50% above its 30-day baseline.
