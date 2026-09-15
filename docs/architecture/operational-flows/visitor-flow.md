# Visitor Ticketing & AI Feedback Recovery Flow

This flow covers offline gate validation, ticketing ingestion, NLP sentiment evaluation, and dynamic recovery offer generation.

```mermaid
sequenceDiagram
    autonumber
    actor Visitor
    participant WebApp as Visitor Booking Web / App
    participant Payment as Payment Gateway
    participant Access as Pass Issuance & Access Control
    participant TicketDB as Ticket / Access Ledger
    participant Feedback as Visitor Insight & Feedback Service
    participant NLP as NLP Sentiment & Recovery Engine
    actor Countess as Estate Owner Dashboard

    Visitor->>WebApp: Select passes (Individual / Family)
    WebApp->>Payment: Process payment transaction
    Payment-->>WebApp: Payment confirmation
    WebApp->>Access: Request signed pass
    Access->>TicketDB: Store pass and access entitlement
    Access-->>Visitor: Deliver signed offline QR pass

    Note over Access: Wi-Fi Disconnected / Local Mode
    Visitor->>Access: Present QR pass at entrance
    Access->>Access: Validate pass using local cryptographic cache
    Access-->>Visitor: Entry granted (Turnstile opens)
    Access->>TicketDB: Record local access event

    Note over Access: Wi-Fi Reconnected
    Access->>TicketDB: Sync buffered access events

    Visitor->>WebApp: Submit free-text feedback
    WebApp->>Feedback: Submit comments and ratings
    Feedback->>NLP: Send feedback text and visitor context
    NLP->>NLP: Perform sentiment analysis and topic tagging
    alt Negative / Urgent Feedback Detected
        NLP->>Feedback: Request recovery action
        Feedback-->>Visitor: Push voucher / free beverage pass
        NLP->>Countess: Alert on high-priority negative trend
    end
```

## Operational commentary

This flow highlights the resilient edge-first pass validation pattern and the end-user recovery loop. It keeps entry experiences smooth when connectivity is weak while still feeding back into the estate's insight systems once the network is available.
