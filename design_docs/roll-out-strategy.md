# Roll-Out Strategy

## Overview

One-time build costs span hardware deployment (sensors, cameras, MQTT gateways) and software integration (edge models, cloud engines, RAG corpus). This strategy prioritizes high-impact assets first, reduces deployment risk through phased rollout, and aligns hardware and software timelines.

## Hardware Deployment

### Phase 1: High-risk assets (Weeks 1–8)

**Rides:**
- 5 heritage/high-throughput rides with the most visitor traffic
- Install vibration/acoustic sensors and local MQTT gateways
- Cost: ~$15K (sensors + gateways)

**Enclosures:**
- Piranhas (highest welfare risk)
- Top 3 visitor-traffic enclosures
- Install vision boxes and motion/weight sensors, local MQTT gateways
- Cost: ~$20K (cameras + sensors + gateways)

**Outcome:** Proof of concept on highest-risk assets; validates edge inference and MQTT pipeline before estate-wide rollout.

### Phase 2: Moderate-risk assets (Weeks 9–16)

**Rides:**
- Remaining 35 rides, deployed in batches of 10 per week
- Reuse gateway hardware from Phase 1 where applicable
- Cost: ~$25K (sensors)

**Enclosures:**
- Remaining 50 enclosures (non-piranhas)
- Cost: ~$30K (cameras + sensors)

### Phase 3: Coverage completion (Weeks 17–24)

- Any remaining locations (kiosks, staff areas for footfall sensing)
- Cost: ~$10K

**Total hardware cost:** ~$100K over 6 months

## Software Deployment

### Phase 1: Edge models (Weeks 1–8)

- Deploy TinyML vibration/acoustic models on ride gateways
- Deploy off-the-shelf vision models on enclosure boxes (general pose/motion, not species-specific yet)
- Models run in eval mode; flagged anomalies logged locally but not yet published to cloud
- Cost: Model licensing + integration (~$5K)

### Phase 2: Cloud engines and RAG corpus (Weeks 9–12)

- Predictive Wear Model for rides (trained on Phase 1 data + manufacturer specs)
- Multimodal Animal Health Engine (general animal pose, not yet species-tuned)
- RAG Maintenance Copilot: ingest and chunk estate manuals and veterinary records
- Cost: Cloud infrastructure + corpus preparation (~$10K)

### Phase 3: AI gateway and event pipeline (Weeks 13–16)

- Deploy Internal AI Gateway (shared by Visitors, Staffing, Maintenance quanta)
- Wire up production event publishing from edge to cloud
- Cost: Gateway implementation + testing (~$8K)

**Total software cost:** ~$23K over 4 months

Note: These phases refer to infrastructure/deployment timeline. The README's Roadmap table describes AI model maturity, which evolves in parallel with these hardware/software rollout phases.

## Deployment Risk Mitigation

1. **Phased hardware rollout** — start on highest-risk assets; validate and iterate before estate-wide deployment
2. **Edge models in eval mode first** — anomalies logged locally for 2–4 weeks before cloud publication, allowing tuning without production impact
3. **Fallback-first design** — Phase 1 static thresholds remain the active path until Phase 2 models beat them on override rate (per [ADR: Production Monitoring & Drift Detection](../ADRs/ADR-ai-vendor-risk-and-monitoring.md))
4. **Staged software releases** — edge model updates via firmware; cloud model updates via gateway config; both support rollback to the previous version within minutes

## Timeline

| Phase | Hardware | Software | Cost | Duration |
| --- | --- | --- | --- | --- |
| 1 | High-risk assets (8 rides/enclosures) | TinyML + vision models (eval mode) | $35K + $5K | Weeks 1–8 |
| 2 | Moderate-risk assets (40 rides/enclosures) | Cloud engines + RAG corpus | $55K + $10K | Weeks 9–16 |
| 3 | Coverage completion | AI gateway + event pipeline | $10K + $8K | Weeks 17–24 |

**Total:** ~$123K over 24 weeks (6 months)

## Success Criteria

- Phase 1: Edge inference runs stably on 8 assets; anomaly logs are complete and retrievable
- Phase 2: Cloud engines consume Phase 1 data; override rate on Phase 1 assets improves or stays flat
- Phase 3: Estate-wide event pipeline published; production monitoring active on all quanta

Advancement to the next phase gates on the previous phase's success criteria and on data quality (Phase 1 data exists and is actionable before Phase 2 models are trained).
