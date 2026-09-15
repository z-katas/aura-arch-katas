# Executive Analytics & Human-in-the-Loop Intelligence Flow

This flow covers stream aggregation, AI analytics forecasting, confidence scoring, human-in-the-loop validation, and staff re-allocation.

```mermaid
sequenceDiagram
    autonumber
    participant Broker as Event Backbone / Message Broker
    participant StreamProc as Real-Time Stream Aggregator
    participant AIAgent as Executive Insights & Forecasting
    participant EventDB as Telemetry & Event Store
    actor Countess as Countess / Executive Dashboard
    participant StaffSvc as Crowd Triage & Dispatch Agent
    actor FieldStaff as Field Dispatch App

    Broker->>StreamProc: Forward ticket, telemetry, and feedback events
    StreamProc->>StreamProc: Aggregate KPIs (footfall, sales, queue wait times)
    StreamProc->>AIAgent: Push aggregated metric windows

    AIAgent->>AIAgent: Generate revenue forecasts & crowd heatmaps

    alt High-Confidence Insight (Score >= 0.85)
        AIAgent->>EventDB: Log insight payload
        AIAgent->>Countess: Publish live KPI and revenue trend
        AIAgent->>StaffSvc: Auto-trigger staff re-allocation order
        StaffSvc->>FieldStaff: Push re-routing notification
    else Low-Confidence Insight (Score < 0.85)
        AIAgent->>EventDB: Log tagged insight (Requires Review)
        AIAgent->>Countess: Flag insight for human approval on Dashboard
        Countess->>Countess: Review context & approve recommendation
        Countess->>AIAgent: Submit approval decision
        AIAgent->>StaffSvc: Execute approved deployment plan
    end
```

## Operational commentary

This analytics pattern keeps the estate's decision-making transparent and controlled. High-confidence signals can trigger automation, while low-confidence insights stay in a human review loop to protect operational quality and trust.
