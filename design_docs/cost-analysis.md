# Cost Analysis — Business Operations

> **Scope:** operational labour costs and AI-assisted savings for animal monitoring and ride maintenance. For AI token budgets and model spend across all quanta, see [AI Cost Analysis](ai-cost-analysis.md).

All figures are illustrative estimates matching the estate brief. Rates used: **$35/hr keeper**, **$45/hr maintenance technician**.

---

## 1. Animal Welfare Monitoring

### Manual baseline (200 animals, 55 enclosures)

- 20 min health/feeding check per animal per day
- 200 × 0.33 hrs × 7 days = **466 hrs/week → $16,310/week**

### At scale (visitor growth = collection growth)

| Scale | Animals | Keeper hrs/week | Cost/week |
|---|---|---|---|
| 1x | 200 | 466 | $16,310 |
| 2x | 400 | 932 | $32,620 |
| 3x | 600 | 1,398 | $48,930 |

At 3x, manual monitoring needs ~35 full-time keepers — not viable for a lean team.

### With AI-assisted monitoring

Continuous sensors + camera feeds triage automatically. Keepers act only on flagged cases.

Assumptions: 5 min/animal/week spot-check; 15% flagged for 30-min deeper review.

| Scale | Animals | Keeper hrs/week | AI inference/week | Total/week | Cost per animal/week |
|---|---|---|---|---|---|
| 1x | 200 | 31.6 hrs | $8.40 | **$1,114** | $5.57 |
| 2x | 400 | 63.2 hrs | $16.80 | **$2,229** | $5.57 |
| 3x | 600 | 94.8 hrs | $25.20 | **$3,343** | $5.57 |

AI inference: 1 vision-model call/animal/operating hour × 12 hrs/day × $0.0005/call.

### Animal monitoring summary

| Scale | Manual/week | AI-assisted/week | Saving |
|---|---|---|---|
| 1x | $16,310 | $1,114 | **93%** |
| 2x | $32,620 | $2,229 | **93%** |
| 3x | $48,930 | $3,343 | **93%** |

Cost stays flat per animal at every scale. Keeper budget is freed for direct animal care, not routine observation.

---

## 2. Ride Maintenance

The estate has **40 rides**, including a safety-critical 18th-century heritage collection. Ride count is fixed — it does not scale with visitors.

### Manual baseline (reactive)

| Activity | Assumption | Hrs/week | Cost/week |
|---|---|---|---|
| Daily safety inspection | 1 hr/ride/day (regulatory) | 280 hrs | $12,600 |
| Unplanned breakdown callouts | ~20/week × 3 hrs diagnosis + repair | 60 hrs | $2,700 |
| **Total** | | **340 hrs** | **$15,300** |

### With AI-assisted monitoring

Edge TinyML sensors pre-screen rides continuously. Anomalies trigger the Work Order Copilot (RAG-grounded cited steps), cutting diagnosis time and reducing unplanned incidents.

Assumptions:
- Sensor pre-screening reduces inspection time by **40%** (technician reviews summary, not full manual check)
- Predictive anomaly detection reduces unplanned breakdowns by **60%** (20 → 8/week)
- Work order copilot cuts per-incident diagnosis time by **50%** (3 hrs → 1.5 hrs)

| Activity | Hrs/week | Cost/week |
|---|---|---|
| AI-assisted inspections (40 rides × 0.6 hrs × 7 days) | 168 hrs | $7,560 |
| Remaining unplanned callouts (8/week × 1.5 hrs) | 12 hrs | $540 |
| AI inference (edge anomaly detection + copilot calls) | — | ~$50 |
| **Total** | **180 hrs** | **$8,150** |

### Ride maintenance summary

| | Manual/week | AI-assisted/week | Saving |
|---|---|---|---|
| Inspection + callouts | $15,300 | $8,150 | **~47%** |

The saving is lower than animal monitoring because daily safety inspections are a **regulatory requirement** — AI reduces inspection time and reactive callouts but cannot eliminate the inspection itself.

---

## Combined Weekly Cost

| Area | Manual/week | AI-assisted/week | Saving |
|---|---|---|---|
| Animal monitoring (1x) | $16,310 | $1,114 | 93% |
| Ride maintenance | $15,300 | $8,150 | 47% |
| **Combined** | **$31,610** | **$9,264** | **~71%** |

---

**Not included:** one-time build costs (sensor/camera installation, MQTT gateway, model integration). See [roll-out strategy](roll-out-strategy.md).
