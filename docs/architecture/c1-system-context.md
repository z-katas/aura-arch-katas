# C1 System Context View

This view defines the macro-system boundary of the Von Digitalis Estate Platform, identifying the external actors, edge environments, cloud boundary, and third-party integrations that shape the platform.

```mermaid
flowchart LR
    classDef actor fill:#e8f3ff,stroke:#2b6cb0,stroke-width:1px,color:#111;
    classDef system fill:#e8f5e9,stroke:#2f7d32,stroke-width:1px,color:#111;
    classDef ext fill:#fff4e5,stroke:#c77700,stroke-width:1px,color:#111;

    countess["Countess<br/>Von Digitalis"]:::actor
    visitor["Park Visitor<br/>/ Family"]:::actor
    caretaker["Field Staff<br/>/ Caretaker"]:::actor

    aura["AURA Estate Platform<br/>Unified estate platform"]:::system

    edge["Edge Sensor & Camera Fleet<br/>40 rides / 55 enclosures"]:::ext
    payment["Payment Gateway"]:::ext

    subgraph Legend["Context legend"]
        legend_user["User"]:::actor
        legend_system["System"]:::system
        legend_external["External system"]:::ext
    end

    visitor -->|"Tickets / QR passes / feedback"| aura
    countess -->|"KPIs / approvals"| aura
    caretaker -->|"Work orders / status updates"| aura

    aura -->|"REST / online payment"| payment
    edge -->|"Buffered telemetry<br/>MQTT store-and-forward"| aura

    classDef note fill:#f7f7f7,stroke:#666,stroke-width:0px;
```

## Scope

- Visitor and family interactions across ticketing, gate entry, and feedback capture
- Maintenance and welfare monitoring for rides and animal enclosures
- Staff dispatch and operational triage
- Executive analytics and human review of AI-generated recommendations

## Architectural intent

The system is intentionally designed as a bounded digital estate platform with edge-first data collection and asynchronous event transfer to central services. This creates resilience in low-connectivity conditions while preserving a coherent cloud-side operational model.

Related views:
- [C2 Container View](./c2-container-view.md)
- [Operational flow: Visitor](./operational-flows/visitor-flow.md)
