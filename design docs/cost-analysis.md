# Cost Analysis

This analysis focuses on the **animal welfare monitoring** use case, since it is the estate's most labor-intensive recurring cost and the clearest place to show AI's financial impact. The same modeling approach (manual baseline → scaling stress test → AI-assisted cost) can be applied to the other use cases; see [Details: link additional cost breakdowns if your team models ticketing/footfall/growth costs separately].

- A **hybrid approach** (AI-assisted monitoring + keeper review for flagged cases) is the cost-effective option — full manual monitoring does not scale to the estate's 3-year growth target.
- AI-assisted monitoring is roughly **15x cheaper** than fully manual monitoring at current scale (**~$1,140/week vs ~$16,310/week**), while improving how quickly issues are caught, since sensors and cameras run continuously rather than relying on periodic keeper rounds.
- The **Human Review Factor** is assumed at **15%** — the estimated share of animals that need a keeper's direct attention in a given week after AI-based triage. This is expected to fall further as the vision/anomaly models are tuned on keeper feedback.
- Even as the animal collection grows alongside visitor numbers (2x–3x over 3 years), AI-assisted monitoring costs stay low, while fully manual costs become unsustainable for a lean keeper team.
- Build costs (model integration, sensor/camera rollout, MQTT ingestion pipeline) and run costs (inference, cloud, storage) are still far lower than the keeper hours they replace.
- Costs shown are illustrative estimates for the purposes of this kata — see assumptions inline.

All figures below assume a **$35/hour** keeper rate and a **200-animal, 55-enclosure** baseline, matching the estate as described in the brief.

## Current Costs (200 animals, fully manual monitoring)

- Manual health/feeding check: **20 minutes (0.33 hrs) per animal per day**
- 200 animals × 0.33 hrs × 7 days = **466 hours/week**

| Total time (hrs/week) | Keeper cost ($/week) | Cost per animal/week |
| ---------------------- | --------------------- | ---------------------- |
| 466 hrs                | $16,310                | $81.55                 |

## Scaling Up Costs (manual monitoring, as the collection grows)

The animal collection is assumed to grow in step with the estate's visitor growth target (5,000 → 15,000 visitors/day, i.e. ~3x over 3 years).

- **2x (400 animals):** 466 × 2 = 932 hrs/week
- **3x (600 animals):** 466 × 3 = 1,398 hrs/week

| Scaling factor | Number of animals | Total keeper time (hrs/week) | Keeper cost ($/week) | Cost per animal/week |
| --------------- | ------------------ | ------------------------------ | ---------------------- | ---------------------- |
| **2x**          | 400                 | 932 hrs                        | $32,620                 | $81.55                  |
| **3x**          | 600                 | 1,398 hrs                      | $48,930                 | $81.55                  |

At 3x scale, monitoring alone would require the equivalent of **~35 full-time keepers** (1,398 hrs ÷ 40 hrs/week) — not realistic for a lean estate team.

## Costs with AI-Assisted Monitoring

With continuous camera and MQTT sensor coverage (weight, motion, feeding-station sensors) feeding an AI pipeline, most animals need only a lightweight automated check, and keepers focus on the animals flagged as anomalous.

Assumptions:
- **Baseline automated spot-check** (keeper confirms sensor/vision readings look sane): 5 minutes (0.083 hrs) per animal per week
- **Human Review Factor:** 15% of animals flagged by the AI for a deeper keeper review each week, at 30 minutes (0.5 hrs) per flagged animal

**Keeper time (200 animals):**
- Spot-checks: 200 × 0.083 = 16.6 hrs/week
- Flagged reviews: 200 × 0.15 × 0.5 = 15 hrs/week
- **Total: 31.6 hrs/week**

### Keeper Time and Cost at Scale

| Scaling factor | Number of animals | Spot-check time (hrs) | Flagged review time (hrs) | Total keeper time (hrs/week) | Keeper cost ($/week) |
| --------------- | ------------------ | ----------------------- | ---------------------------- | ------------------------------ | ---------------------- |
| **1x**          | 200                 | 16.6 hrs                | 15 hrs                        | 31.6 hrs                        | $1,106                  |
| **2x**          | 400                 | 33.2 hrs                | 30 hrs                        | 63.2 hrs                        | $2,212                  |
| **3x**          | 600                 | 49.8 hrs                | 45 hrs                        | 94.8 hrs                        | $3,318                  |

### AI Inference Costs (Automated Monitoring)

**Cost formula:**
AI cost = number of inferences per animal per day × number of animals × cost per inference × 7 days

- Assumes 1 vision-model inference per animal per operating hour (12 operating hrs/day), using a low-cost vision/anomaly-detection model
- Estimated cost per inference: **$0.0005** (illustrative — small vision-classification call, not a full multimodal LLM call)

- **1x (200 animals):** 12 × 200 × 0.0005 × 7 = **$8.40/week**
- **2x (400 animals):** 12 × 400 × 0.0005 × 7 = **$16.80/week**
- **3x (600 animals):** 12 × 600 × 0.0005 × 7 = **$25.20/week**

Inference cost stays negligible relative to keeper cost at every scale — consistent with the fact that this workload is high-volume but low-complexity (anomaly detection on sensor/image data), not open-ended generation.

### Total Costs with AI-Assisted Monitoring

| Scaling factor | Number of animals | Keeper cost ($/week) | AI inference cost ($/week) | Total cost ($/week) | Cost per animal/week |
| --------------- | ------------------ | ---------------------- | ----------------------------- | ---------------------- | ---------------------- |
| **1x**          | 200                 | $1,106                  | $8.40                          | $1,114                  | $5.57                   |
| **2x**          | 400                 | $2,212                  | $16.80                         | $2,229                  | $5.57                   |
| **3x**          | 600                 | $3,318                  | $25.20                         | $3,343                  | $5.57                   |

## Summary

| Scaling factor | Fully manual ($/week) | AI-assisted ($/week) | Savings | Cost per animal (manual vs AI) |
| --------------- | ----------------------- | ----------------------- | -------- | --------------------------------- |
| **1x**          | $16,310                  | $1,114                   | ~93%      | $81.55 → $5.57                     |
| **2x**          | $32,620                  | $2,229                   | ~93%      | $81.55 → $5.57                     |
| **3x**          | $48,930                  | $3,343                   | ~93%      | $81.55 → $5.57                     |

AI-assisted monitoring keeps welfare monitoring cost roughly flat per animal as the collection scales, while fully manual monitoring scales linearly and becomes operationally infeasible well before the estate reaches its 3-year visitor target. This frees keeper budget to be redirected toward animal care itself rather than routine observation, directly supporting the Countess's "healthy and happy animals" objective without a corresponding headcount increase.

**Not included in this estimate:** one-time build costs (camera/sensor installation, MQTT gateway rollout, model integration), which are addressed separately in the [roll-out strategy](roll-out-strategy.md).