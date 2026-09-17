# ADR: Dual Data Store — Events DB vs. Data Lake / Warehouse

## Status
Accepted

## Context
The Analytics quantum's top driving characteristic is **Data Integrity** (per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md)), and it must serve two very different data-access patterns:

1. **Auditable, explainable insight records** — each AI-generated insight/forecast, with its confidence score and reasoning, needs to be retrievable for explainability (per [ADR](ADR-001-ai-vendor-risk-and-monitoring.md)) and for tracking the human override rate that we use as our production drift signal.
2. **Bulk raw and aggregated telemetry** — footfall, ticketing, and sensor data at scale, needed for trend analysis, model retraining, and ad hoc historical queries (e.g. "how did October Saturdays compare last year"), which is a very different query shape than "look up this one insight."

A single store optimized for one pattern is a poor fit for the other: an OLTP-style store struggles with large historical aggregation queries, while a data lake is a poor fit for fast, structured lookup of a specific insight and its reasoning.

We considered three alternatives:

1. **Single structured/OLTP-style store for everything** — one schema for insights, raw telemetry, and aggregates. Simplest to operate and query consistently, but a structured store sized for fast indexed lookups is a poor fit for bulk historical telemetry at volume — trend analysis and model-retraining queries would either be slow or force compromises on the insight schema to accommodate them.
2. **Single data lake / warehouse for everything, including insight records** — good for bulk telemetry and historical aggregation, but a poor fit for the Estate Owner Dashboard's "insight review" flow and the override-rate monitoring in [ADR](ADR-001-ai-vendor-risk-and-monitoring.md), both of which need fast, structured lookup of *this specific insight, its confidence score, and its review outcome* — not a scan over a lake.
3. **Two stores split by access pattern** — an Events DB for individual, auditable insight/forecast records, and a data lake for bulk raw/aggregated telemetry.

## Decision
The Analytics quantum writes to **two separate stores** for these two purposes:
- **Events DB** — structured, indexed storage for individual insight/forecast records: the output, confidence score, reasoning, and (once reviewed) the human approve/correct outcome. This is what the Estate Owner Dashboard's "insight review" flow and our override-rate monitoring query against.
- **Data lake / warehouse** — raw and aggregated telemetry at volume, used for trend analysis, model retraining datasets, and the Personalization Recommender's visit-history queries.

The AI Analytics Agent writes to both after generating an insight: the insight itself to Events DB, the underlying raw/aggregated data to the data lake.

## Consequences

**Positive:**
- Each store is fit for its access pattern — fast, auditable insight lookups stay fast even as historical telemetry volume grows into millions of records.
- Model retraining and historical analysis (data lake) can run heavy queries without competing with, or slowing down, live dashboard/review queries (Events DB).
- Clean separation supports **data integrity**: the Events DB acts as an audit trail of what the AI said and what a human decided about it, independent of raw data volume or retention policy changes on the lake side.

**Negative / trade-offs:**
- Two stores means two schemas to maintain and keep in sync conceptually (an insight in Events DB should be traceable back to the raw data that produced it in the data lake).
- Slightly more operational overhead (two systems to provision, monitor, and back up) than a single store — accepted as a reasonable cost given the very different access patterns each serves.
- Requires a documented linking key (e.g. insight ID referencing a data lake batch/time range) so an Estate Owner reviewing a flagged insight can, if needed, trace back to the underlying raw data — not yet detailed, and worth a follow-up decision if this becomes a frequent need in the MVP phase.
