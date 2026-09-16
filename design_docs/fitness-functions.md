# Fitness Functions

Measurable success criteria for the architecture's driving characteristics. These are the signals that tell us the architecture is still working as designed — not just that the system is up.

Each function is marked **Automated** (can be enforced in CI/runtime) or **Monitored** (requires a dashboard or periodic audit).

---

## Cross-cutting — AI Integrity

| Fitness Function | Threshold | Type |
|---|---|---|
| Human override rate per quantum does not rise >50% above its 30-day rolling baseline | Alert + auto-rollback at gateway | Automated |
| Every AI-derived output carries a confidence score and provenance flag | Enforced at gateway — reject responses missing either field | Automated |
| AI Gateway uptime | ≥ 99.9% monthly | Monitored |
| Provider failover completes on timeout | < 5 seconds | Automated |
| Semantic cache hit rate on high-repetition paths (ticket assistant, feedback triage) | ≥ 20% after 30 days live | Monitored |
| Token spend per quantum visible in real-time | Lag ≤ 1 hour | Monitored |

---

## Visitors — Availability · Performance · Scalability

| Fitness Function | Threshold | Type |
|---|---|---|
| Ticket checkout p99 latency under 3x visitor load (15,000/day) | < 2s | Automated |
| AI advisory response (bundle recommendation) | < 3s or non-LLM fallback fires automatically | Automated |
| AI timeout never blocks a ticket purchase | Zero checkout failures attributable to AI path | Automated |
| Feedback safety triage classifies every submission | < 1s per submission; no dropped events | Automated |
| Ticketing throughput scales to 3x visitors | No schema or infrastructure change required | Automated (load test) |

---

## Staffing — Fault Tolerance · Availability

| Fitness Function | Threshold | Type |
|---|---|---|
| High-severity dispatch proposal delivered to manager after anomaly event | < 10 seconds | Automated |
| Field handheld serves last-known deployment plan during full Wi-Fi outage | 100% of devices — verified in offline mode test | Automated |
| No incident log entry lost during connectivity gap | QoS 1 delivery confirmed on reconnect; zero silent drops | Automated |
| 45-second escalation fires within tolerance | ± 5 seconds of threshold | Automated |
| Low-severity crowd nudges dispatched without manager approval | Confirmed auto-dispatch path never blocked by approval gate | Automated |

---

## Maintenance — Data Integrity · Deployability

| Fitness Function | Threshold | Type |
|---|---|---|
| No anomaly event dropped during connectivity gap | Every event on disk-backed queue delivered exactly once on reconnect (idempotent consumer) | Automated |
| Work order copilot never produces uncited steps | Every step cites a retrieved passage or triggers human escalation — zero uncited generations | Automated |
| Edge model fleet rollout to all ~95 locations | Completes in < 4 hours; rollback available within same window | Monitored |
| Biomass / feeding anomaly event published after threshold breach | < 60 seconds | Automated |
| Animal Population Registry baseline stays current | No enclosure has a stale expected-count baseline > 24 hours after a keeper-logged change | Automated |

---

## Analytics — Data Integrity · Interoperability

| Fitness Function | Threshold | Type |
|---|---|---|
| Raw telemetry and AI-derived insight always on separate published topics | Enforced by broker topic schema — mixed publication fails at ingest | Automated |
| Every AI insight traceable to the raw event that produced it | Linking key present on every Events DB record | Automated |
| Override rate computed and surfaced per model version | Lag ≤ 1 hour from decision to dashboard | Monitored |
| No quantum reads directly from another quantum's internal store | Enforced by network policy / API boundary — no cross-quantum DB access | Automated |

---

## Marketing — Adaptability · Interoperability

| Fitness Function | Threshold | Type |
|---|---|---|
| New campaign type deployable without modifying Visitors or Analytics services | Verified by deploying a campaign change with zero PRs to other quanta | Monitored |
| Every dismissed offer publishes an `OfferDismissed` event | 100% — no silent dismissals; verified by event audit | Automated |
| Personalisation recommender calls shared AI Gateway | Zero direct provider SDK calls from Marketing service | Automated |

---

## How to use these

- **Automated** functions should be wired into CI pipelines, load tests, and runtime alerts from day one — they fail the build or page on-call if breached.
- **Monitored** functions are checked on a cadence (weekly dashboard review, monthly audit) until tooling exists to automate them.
- A quantum does **not** advance from Phase 1 → Phase 2 in the [Roadmap](../README.md#roadmap) unless its override-rate fitness function is green for a sustained 30-day period.
