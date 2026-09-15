# Predictive Asset Maintenance & RAG Diagnostic Dispatch Flow

This flow covers edge sensor ingestion, TinyML/VLM anomaly detection, store-and-forward transport, vector knowledge retrieval, and field dispatch.

```mermaid
sequenceDiagram
    autonumber
    actor RideAsset as 40 Rides / 55 Enclosures
    participant EdgeAI as Edge Sensing & Local Inference
    participant Buffer as MQTT Buffer & Store-and-Forward
    participant Broker as Event Backbone / Message Broker
    participant Monitor as Predictive Maintenance Monitor
    participant RAG as RAG Diagnostic Copilot
    participant Knowledge as Asset Knowledge & Manual Store
    participant WorkOrders as Maintenance Work Orders
    participant StaffApp as Field Dispatch App

    RideAsset->>EdgeAI: Stream raw acoustics, temp, & camera frames
    Note over EdgeAI: Execute local TinyML acoustic check & piranha counting
    EdgeAI->>Buffer: Write compact anomaly payload to local SQLite

    alt Network Intermittent
        Buffer->>Buffer: Hold payloads in persistent queue
    else Wi-Fi Restored
        Buffer->>Broker: Flush payloads via MQTT QoS 1/2
    end

    Broker->>Monitor: Deliver sensor and anomaly events
    Monitor->>Monitor: Evaluate predictive wear curves
    Monitor->>RAG: Trigger diagnostic request (High Failure Risk)

    RAG->>Knowledge: Query schematics and vet treatment logs
    Knowledge-->>RAG: Return approved excerpts and resolution history
    Note over RAG: Synthesize root-cause analysis, step-by-step fix, & parts list

    RAG->>WorkOrders: Create enriched work order
    WorkOrders->>StaffApp: Push repair / treatment assignment
    StaffApp->>RideAsset: Perform physical repair / treatment
    StaffApp->>Knowledge: Voice-log resolution notes for future retrieval
```

## Operational commentary

The maintenance flow is the critical safety path for the estate. It combines low-latency local inference with a cautious fallback model: if vector similarity is weak or uncertain, the system reverts to safer procedures rather than inventing steps for high-risk physical work.
