# Architecture Characteristics Analysis

## Driving Characteristics by Quantum

Using the Architecture Characteristics and Architecture Styles worksheets, AURA identified the top driving characteristics for each of the four architecture quanta, and selected a matching architecture style for each.

![Existing architectural characteristics](../assets/architecture-characteristics-styles1.png "Existing architectural characteristics")

![Existing architectural characteristics](../assets/architecture-characteristics-styles2.png "Existing architectural characteristics")


| Quantum         | Top 3 Characteristics                          | Others Considered                 | Style         | Why                                                                     |
| --------------- | ---------------------------------------------- | --------------------------------- | ------------- | ----------------------------------------------------------------------- |
| **Visitors**    | Availability, Performance, Scalability         | Concurrency                       | Microservices | Highest-traffic surface; must scale independently for 3x visitor growth |
| **Staffing**    | Fault Tolerance, Availability, Usability       | Consistency                       | Event-driven  | Deployment suggestions are useless if late or lost                      |
| **Maintenance** | Data Integrity, Extensibility, Deployability   | Data Consistency, Security        | Microservices | Safety/welfare-critical; a wrong reading is worse than a slow one       |
| **Analytics**   | Data Integrity, Interoperability, Adaptability | Data Consistency, Fault Tolerance | Event-driven  | Aggregates every other quantum's data; must stay correct and pluggable  |




## AI Use Cases


| #   | Use Case                                                                                                         | Top 3 NFRs                                      | Implicit         |
| --- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | ---------------- |
| 1   | **Ticket & family pass assistant** — LLM reads visitor intent, recommends the right pass                         | Availability, Performance, Scalability          | Explainability   |
| 2   | **Real-time footfall & staff deployment** — MQTT + anomaly detection turns sensor data into staffing suggestions | Fault Tolerance, Responsiveness, Data Integrity | Interoperability |
| 3   | **Animal health & piranha population monitoring** — vision/sensor triage, only ~15% flagged for keeper review    | Data Integrity, Extensibility, Deployability    | Explainability   |
| 4   | **Personalization & retention marketing** — visit history + analytics drive targeted offers                      | Interoperability, Adaptability, Data Integrity  | Privacy          |


**Key design notes:**

- **#2 spans two quanta** (Staffing consumes what Analytics produces) — not a clean single-quantum fit, called out deliberately.
- **#3's model must be biased toward over-flagging** — a false "healthy" is worse than a false alarm.
- **#1 and #4 both need Explainability/Adaptability as escape hatches** — recommendations and campaigns will be wrong sometimes; the architecture must make overriding or retuning them cheap, not a redeploy.



## Use Case → Quantum Map


| AI Use Case                           | Primary Quantum      |
| ------------------------------------- | -------------------- |
| Ticket & family pass assistant        | Visitors             |
| Footfall & staff deployment           | Staffing / Analytics |
| Animal health & population monitoring | Maintenance          |
| Personalization & retention           | Analytics            |


*Security and Privacy are treated as implicit across all four quanta (per worksheets) — baseline requirements, not differentiators.*