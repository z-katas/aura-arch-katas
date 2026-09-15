# Architecture Characteristics Analysis

## Driving Characteristics by Quantum

Using the Architecture Characteristics and Architecture Styles worksheets, AURA identified the top driving characteristics for each of the five architecture quanta, and selected a matching architecture style for each.


![Existing architectural characteristics](/assets/architecture-characteristics-styles1.png "Existing architectural characteristics")


![Existing architectural characteristics](/assets/architecture-characteristics-styles2.png "Existing architectural characteristics")

![Marketing architecture characteristics worksheet](/assets/arch-characteristics-marketing.png "Marketing architecture characteristics worksheet")

![Marketing architecture styles worksheet](/assets/arch-styles.marketing.png "Marketing architecture styles worksheet")

| Quantum | Top 3 Characteristics | Others Considered | Style | Why |
| --- | --- | --- | --- | --- |
| **Visitors** | Availability, Performance, Scalability | Concurrency | Microservices | Highest-traffic surface; must scale independently for 3x visitor growth |
| **Staffing** | Fault Tolerance, Availability, Responsiveness | Consistency | Event-driven | Deployment suggestions are useless if late or lost |
| **Maintenance** | Data Integrity, Extensibility, Deployability | Data Consistency, Security | Microservices | Safety/welfare-critical; a wrong reading is worse than a slow one |
| **Analytics** | Data Integrity, Interoperability, Adaptability | Data Consistency, Fault Tolerance | Event-driven | Aggregates every other quantum's data; must stay correct and pluggable |
| **Marketing** | Adaptability, Interoperability, Deployability | Data Integrity, Availability | Event-driven | Campaign rules and channels will change frequently; event subscriptions let Marketing react to behavior without coupling to Analytics or Visitor internals |

### Marketing Quantum Worksheet

#### Architecture Characteristics

| Worksheet Field | Selection |
| --- | --- |
| **System/Project** | Von Digitalis Estate |
| **Architect/Team** | AURA |
| **Domain/Quantum** | Marketing |
| **Top 3 Driving Characteristics** | Adaptability, Interoperability, Deployability |
| **Implicit Characteristics** | Security, Privacy |
| **Others Considered** | Data Integrity, Availability |

#### Architecture Style

| Worksheet Field | Selection |
| --- | --- |
| **System/Project** | Von Digitalis Estate |
| **Architect/Team** | AURA |
| **Domain/Quantum** | Marketing |
| **Selected Architecture Style** | Event-driven |

**Marketing style rationale:** Campaign rules, audience definitions, and delivery channels will change frequently. An event-driven style lets Marketing subscribe to visitor and analytics events, publish campaign and engagement events, and add or replace delivery channels without coupling to the internal implementation of the Visitor or Analytics quanta.

## AI Use Cases

| # | Use Case | Top 3 NFRs | Implicit |
| --- | --- | --- | --- |
| 1 | **Ticket & family pass assistant** — LLM reads visitor intent, recommends the right pass | Availability, Performance, Scalability | Explainability |
| 2 | **Real-time footfall & staff deployment** — MQTT + anomaly detection turns sensor data into staffing suggestions | Fault Tolerance, Responsiveness, Data Integrity | Interoperability |
| 3 | **Animal health & piranha population monitoring** — vision/sensor triage, only ~15% flagged for keeper review | Data Integrity, Extensibility, Deployability | Explainability |
| 4 | **Personalization & retention marketing** — visit history + analytics drive targeted offers | Adaptability, Interoperability, Data Integrity | Privacy |

**Key design notes:**
- **#2 spans two quanta** (Staffing consumes what Analytics produces) — not a clean single-quantum fit, called out deliberately.
- **#3's model must be biased toward over-flagging** — a false "healthy" is worse than a false alarm.
- **#1 and #4 both need Explainability/Adaptability as escape hatches** — recommendations and campaigns will be wrong sometimes; the architecture must make overriding or retuning them cheap, not a redeploy.

## Use Case → Quantum Map

| AI Use Case | Primary Quantum |
| --- | --- |
| Ticket & family pass assistant | Visitors |
| Footfall & staff deployment | Staffing / Analytics |
| Animal health & population monitoring | Maintenance |
| Personalization & retention | Marketing (Analytics supplies the behavioral data) |

*Security and Privacy are treated as implicit across all five quanta (per worksheets) — baseline requirements, not differentiators.*