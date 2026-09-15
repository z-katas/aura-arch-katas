# C2 Container View

This container diagram shows the four architectural quanta and the shared message backbone that connects them.

```mermaid
flowchart TB
    classDef client fill:#e8f3ff,stroke:#2b6cb0,stroke-width:1px,color:#111;
    classDef service fill:#e8f5e9,stroke:#2f7d32,stroke-width:1px,color:#111;
    classDef ai fill:#f3e8ff,stroke:#7e22ce,stroke-width:2px,color:#111;
    classDef broker fill:#e0f2fe,stroke:#0369a1,stroke-width:2px,color:#111;
    classDef storage fill:#f5f5f5,stroke:#444,stroke-width:1px,color:#111;

    subgraph VQ["Visitor Quantum"]
        web["Visitor Booking Web / App<br/>[React / iOS / Android]"]:::client
        passsvc["Pass Issuance & Access Control<br/>[Node.js / REST / JWT]"]:::service
        feedbacksvc["Visitor Insight & Feedback Service<br/>[Python / FastAPI]"]:::service
        nlp["NLP Sentiment & Recovery Engine<br/>[Python / FastNLP]"]:::ai
        ticketdb[("Ticket / Access Ledger<br/>[PostgreSQL]")]:::storage
    end

    subgraph MQ["Maintenance Quantum"]
        edge["Edge Sensing & Local Inference Layer<br/>[Jetson / TinyML / Edge VLM]"]:::ai
        buffer["Local MQTT Buffer & Store-and-Forward<br/>[Mosquitto / SQLite]"]:::service
        monitor["Predictive Maintenance Monitor<br/>[Python / Timeseries ML]"]:::ai
        rag["RAG Diagnostic Copilot<br/>[Python / LangChain]"]:::ai
        vector[("Asset Knowledge & Manual Store<br/>[pgvector / Qdrant]")]:::storage
        workdb[("Maintenance Work Orders<br/>[PostgreSQL]")]:::storage
    end

    subgraph SQ["Staffing Quantum"]
        triage["Crowd Triage & Dispatch Agent<br/>[Python / Ray]"]:::ai
        roster[("Staff Roster / Skills DB<br/>[PostgreSQL]")]:::storage
        mobile["Field Dispatch App<br/>[React Native / Mobile]"]:::client
    end

    subgraph AQ["Analytics Quantum"]
        broker["Event Backbone & Message Broker<br/>[EMQX / Kafka]"]:::broker
        stream["Real-Time Stream Aggregator<br/>[Apache Flink]"]:::service
        insights["Executive Insights & Forecasting<br/>[Python / LLM]"]:::ai
        dashboard["Executive Dashboard<br/>[React / Web UI]"]:::client
        events[("Telemetry & Event Store<br/>[ClickHouse / Timescale]")]:::storage
    end

    subgraph Legend["Technical container legend"]
        legend_client["User-facing app"]:::client
        legend_service["Application service"]:::service
        legend_ai["AI / ML component"]:::ai
        legend_broker["Message broker"]:::broker
        legend_storage[("Persistent storage")]:::storage
    end

    web -->|"Book / buy tickets"| passsvc
    passsvc -->|"Issue signed QR passes"| ticketdb
    web -->|"Visitor comments / ratings"| feedbacksvc
    feedbacksvc -->|"Text / tone / topics"| nlp
    nlp -->|"Recovery offers / alerts"| web

    edge -->|"Acoustic / thermal / vision"| buffer
    buffer -->|"Buffered events"| broker
    broker -->|"Sensor / anomaly feeds"| monitor
    monitor -->|"Risk triggers"| rag
    rag -->|"Fix guidance / parts"| workdb
    rag -->|"Semantic lookups"| vector
    workdb -->|"Dispatch"| triage
    triage -->|"Assignments"| roster
    roster -->|"Staff routes"| mobile

    ticketdb -->|"Access / ticketing events"| broker
    broker -->|"Telemetry & ticket events"| stream
    stream -->|"Aggregated KPIs"| insights
    insights -->|"Forecasts / alerts"| dashboard
    stream --> events
    events --> dashboard

    classDef outline fill:none,stroke:none,color:#111;
```

## Key architecture characteristics

- The Visitor quantum remains the front-door service area optimized for pass issuance, visitor booking, and feedback-driven recovery.
- The Maintenance quantum includes edge sensing, local buffering, anomaly monitoring, and domain knowledge retrieval for repair and welfare operations.
- Staffing and Analytics are event-driven consumers of operational telemetry and insight events.
- The Event Backbone decouples quanta while preserving a single, shared data movement layer for ticket, sensor, and insight flows.

## Design intent

The system intentionally isolates operational concerns while allowing events to bridge them. The Maintenance quantum handles edge resiliency and local processing, the Visitor quantum owns booking and feedback flows, the Staffing quantum optimizes dispatch, and the Analytics quantum consumes the resulting event stream for dashboarding and insight generation. This keeps failure domains independent, supports offline resilience at the edge, and preserves a path for future AI model swaps without disrupting service contracts.
