# Spatial Crowd Triage & Staff Dispatch Flow

This flow covers Wi-Fi sniffer crowd density tracking, spatial triage prediction, tactical staff optimization, and offline mobile dispatch.

```mermaid
sequenceDiagram
    autonumber
    actor Sensors as Wi-Fi Sniffers & Crowd Sensors
    participant EdgeGateway as MQTT Buffer & Store-and-Forward
    participant CentralBroker as Event Backbone / Message Broker
    participant Triage as Crowd Triage & Dispatch Agent
    participant StaffDB as Staff Roster / Skills DB
    participant Dispatcher as Field Dispatch App
    actor FieldStaff as Field Staff

    Sensors->>EdgeGateway: Stream crowd density & incident logs
    EdgeGateway->>CentralBroker: Flush buffered telemetry via MQTT
    CentralBroker->>Triage: Forward crowd metrics & incident events

    Triage->>Triage: Predict spatial bottlenecks & triage severity
    Triage->>StaffDB: Query available staff, locations, & certified skills
    StaffDB-->>Triage: Return optimal staff profiles

    Triage->>Dispatcher: Generate optimized deployment plan
    Dispatcher->>StaffDB: Commit assignment and status

    alt Staff Mobile Online
        Dispatcher->>FieldStaff: Publish route via MQTT
        FieldStaff-->>Dispatcher: Acknowledge assignment receipt
    else Staff Mobile Offline
        Dispatcher->>CentralBroker: Queue retained route for Staff ID
        Note over FieldStaff: Re-enters Wi-Fi Zone
        CentralBroker->>FieldStaff: Deliver cached routing order
        FieldStaff->>FieldStaff: Update local task state
    end
```

## Operational commentary

Staff planning is based on dynamic conditions rather than static rosters. The flow favors asynchronous dispatch and retained messaging so that burnout-prone or geographically mobile tasks remain actionable even when staff devices briefly drop offline.
