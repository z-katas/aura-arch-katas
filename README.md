[![Cover picture](assets/readme-cover-picture.png "cover picture")](assets/readme-cover-picture.png)

# AURA - Von Digitalis Estates | O'Reilly Architectural Katas (2026)

A structured approach to the **O'Reilly 2026 Architectural Kata Challenge: Von Digitalis Estates**.

## Table of Contents

- [AURA - Von Digitalis Estates | O'Reilly Architectural Katas (2026)](#AURA---von-digitalis-estates--oreilly-architectural-katas-2026)
  * [Team](#team)
  * [Glossary](#glossary)
- [Problem definition](#problem-definition)
  * [Context](#context)
  * [Current State](#current-state)
  * [Challenges](#challenges)
  * [Key Objective](#key-objective)
  * [Constraints](#constraints)
- [Solution](#solution)
  * [Business outcomes targeted](#business-outcomes-targeted)
  * [Automation use-cases using AI](#automation-use-cases-using-ai)
  * [Architecture characteristics](#architecture-characteristics)
  * [Detailed architecture designs](#detailed-architecture-designs)
    + [Ticketing & visitor experience use case](#ticketing--visitor-experience-use-case)
    + [Popularity / footfall analytics use case](#popularity--footfall-analytics-use-case)
    + [Animal health & welfare monitoring use case](#animal-health--welfare-monitoring-use-case)
    + [Visitor growth & retention use case](#visitor-growth--retention-use-case)
  * [Limitations with adoption of AI](#limitations-with-adoption-of-ai)
  * [Productionizing the AI-Powered System](#productionizing-the-ai-powered-system)
- [Final thoughts](#final-thoughts)
  * [Anti-patterns](#anti-patterns)
  * [Roadmap](#roadmap)
  * [Our Learnings](#our-learnings)

## Team

![Team cover picture](/assets/team_cover.png "Team cover picture")

- [**Mukundan Nallani Chakravartula**](https://www.linkedin.com/in/mukundannc/), Technical Product Manager
- [**Akhil Raja Reddy Kanthala**](https://www.linkedin.com/in/akhil-raja-reddy/), Senior Tech Lead
- [**Uday Kiran K**](https://www.linkedin.com/in/udaykirankavaturu/), Senior Tech Lead
- [**Durga Laxmi Immadi**](https://www.linkedin.com/in/durga-immadi-916893a3), Senior Tech Lead & UX designer
- [**Ravi Kiran Bhusetty**](https://www.linkedin.com/in/ravi-kiran-bhusetty/), Senior Software Engineer

## Glossary

[TO DO]

## Context

After a gardening accident, the 204th in line to the Von Digitalis title has become the **72nd Countess Von Digitalis**, inheriting a large estate that needs modernizing. The family's old business — explosive garden gnomes — is no longer viable, so the Countess is turning to digital solutions to make the estate profitable.

**Three main attractions:**
- **Amusement park** — 40 rides, an important 18th-century collection, recently passed safety inspection
- **Exotic animal collection** — 200+ animals across 55 enclosures, land and aquatic, opening to the public for the first time (including a jumping piranha collection)
- **Grounds & visitor facilities** — including ticketing and family passes

**Visitors:** ~5,000/day today → target **15,000/day within 3 years**. Missing this target means going back to the gnome business.

## Current State

Starting point is almost entirely analog:

- **Ticketing** — none exists; net-new
- **Visitor tracking** — no instrumentation; no idea which rides/zones are popular
- **Animal monitoring** — manual only; no health, feeding, or population tracking
- **Connectivity** — patchy wifi, no data pipeline to the cloud
- **Revenue** — no prior experience monetizing rides or animals at scale
- **Retention** — no repeat-visitor program

## Challenges

- **No popularity data** — can't tell which rides/zones draw crowds → hard to staff or invest well
- **Costly, reactive animal care** — sickness is expensive; no systematic health/feeding/population tracking (piranhas especially)
- **No growth strategy** — need 3x visitors in 3 years, no plan yet
- **Low repeat visits** — no loyalty or personalization, no visitor behavior data
- **Patchy connectivity** — any real-time system must handle unreliable wifi (MQTT + store-and-forward)
- **No ticketing** — can't monetize or track visitors without it
- **Profitability pressure** — must work, or it's back to garden gnomes
## Key Objective

"How might we use **AI** to help the Countess understand her estate, keep her animals healthy, and grow visitor numbers 3x — while working around patchy connectivity and a lean staff — so the Von Digitalis estates become profitable without a return to the garden gnome business?"

This breaks down into four supporting objectives:

1. **See the estate** — turn footfall across 40 rides and 55 enclosures into actionable insight, so staff and investment go where they're needed.
2. **Protect the animals** — monitor health, feeding, and population (including the piranhas) proactively, catching problems before they become costly.
3. **Grow visitation** — move from 5,000 to 15,000 visitors/day within 3 years through a working ticketing system and data-informed growth strategy.
4. **Build loyalty** — turn first-time visitors into repeat visitors through personalization and engagement.

## Constraints

**Technical**
- **Patchy wifi** — coverage across the park is unreliable; any solution depending on real-time data must tolerate connectivity gaps
- **MQTT hardware only** — there's budget for MQTT-capable devices to be installed throughout the park as the mechanism for getting data from the estate to the cloud; assume no other estate-wide connectivity option
- **Cloud-based** — solution can use cloud services, but must account for the gap between edge (patchy wifi) and cloud

**Business**
- **Scale today** — 40 rides, 55 animal enclosures, 200+ animals, ~5,000 visitors/day
- **Scale target** — 15,000 visitors/day within 3 years (3x growth)
- **Budget is finite** — hardware spend is sanctioned but not unlimited
- **No existing systems** — no ticketing, visitor tracking, or animal monitoring infrastructure to integrate with; this is a greenfield build

**AI-specific**
- **Model/vendor volatility** — AI is moving fast; the best model or provider today may not be the best tomorrow, so the architecture must tolerate switching models or providers
- **Pricing risk** — must handle a provider changing prices unexpectedly
- **Provider risk** — must handle a provider shutting down entirely
- **Non-determinism** — unlike deterministic functionality, GenAI outputs aren't consistently reproducible; the solution needs a way to verify AI-driven functionality is working, and detect if it starts misbehaving in production

# Solution

## Business outcomes targeted

| Outcome | Basis (compact) |
|---|---|
| **Staff efficiency: ~20% better coverage** of high-traffic zones | Pareto footfall (20% of locations → ~60% of visits); staff today spread evenly across ~95 locations. Estimate, no estate data yet. |
| **Animal welfare cost: ~93% reduction** ($16,310 → $1,114/wk) | Derived — see [cost analysis](design_docs/cost-analysis.md). AI monitoring + keeper review on ~15% flagged. |
| **Health incidents: ~6 fewer/year** (20 → ~14) | Est. ~20 events/yr (industry rate); faster detection cuts escalation lag from 12hrs to <1hr, preventing ~70% of late-detection escalations. |
| **Repeat visits: ~20% relative increase** (15% → 18%) | Typical lift from loyalty/personalization in leisure/retail; conservative estimate. |
| **Visitor growth: 3x in 3 yrs** (5,000 → 15,000/day) | Estate's stated target — use cases support it. |
| **Headcount: no linear increase** with 3x scale | Follows from above: hours stay flat/retargeted, not added. |

Refer to [detailed outcomes & cost analysis](design_docs/cost-analysis.md).

## Automation use-cases using AI

We prioritized the following use-cases for this exercise:

![HMW](/assets/hmw.png "HMW")

1. **HMW use AI to make buying tickets effortless** — assisted ticket purchase, family pass recommendations, and in-park wayfinding, so visitors spend less time figuring out logistics and more time enjoying the estate?

2. **HMW use sensor data and AI to understand what's actually popular** — an MQTT sensor + AI pipeline that shows which zones and rides are busiest, so staff and investment go where visitors are?

3. **HMW use AI to keep the animal collection healthy without adding headcount** — computer vision and sensor-based monitoring of feeding, health, and piranha population levels, so issues are caught early rather than discovered too late?

4. **HMW use AI to turn first-time visitors into repeat visitors** — personalization and targeted marketing that drive return visits, so the estate grows revenue without relying purely on new-visitor acquisition?

# Golden Path — Actor Lifecycles

In EventStorming, the **golden path** is the sequence of events when a process completes exactly as intended — no exceptions, no errors. It's established early to give the team a shared timeline before layering in edge cases and policies.

Each lane below is one actor's golden path — the steps that must succeed, in order, for their session to count as a success. Failure branches (failed safety checks, payment retries, flagged anomalies) are documented on the individual event-storming boards, not here.

![Golden path swimlanes](/assets/golden_path.png  "Golden path swimlanes")


## Notable design decisions

- **Every staff track ends in a plain "End,"** but the Visitor and Estate Owner tracks end in a named outcome ("Happy visitor experience," "Single digital platform") — this was a deliberate choice to keep the two audience-facing goals visible on the board itself, rather than only in prose elsewhere in the README.
- **The Estate Owner's path has no explicit "Log out"** shown before its outcome — worth confirming with the team whether that's intentional or a gap versus the other four lanes, which all show Log out before End.

## Event Storming — Identifying the Services
![event-storming-internal-operations](/assets/event-storming-internal-operations.png "event-storming-internal-operations")
![event-storming-visitor-facing](/assets/event-storming-visitor-facing.png "event-storming-visitor-facing")

* After an event storming exercise using the Actor-Action approach across all five actors (Visitor, Estate Owner, Ride Staff, Animal Care Staff, Front Office), the following candidate services were identified in the first run — **Auth, Analytics, Staff Scheduling, Safety Checklist, Queue/IoT, Care Scheduling, Payment Gateway, Recommendation, Notification/Marketing** and **Finance/Reporting**.
* Since several of these were triggered by the same aggregates and had similar scalability and availability needs — e.g. Analytics Engine was invoked identically for popularity reports, dashboards, and enclosure engagement data — they were consolidated rather than kept as separate services.
* Safety Checklist and Queue/IoT both operate at the individual-ride level with the same read/write patterns, so they were folded into a single **Ride Operations** capability.
* So finally, we have **Ticketing, Analytics, Staff Scheduling, Ride Operations, Animal Care, Notification/Marketing** and **Auth** as the identified services.

## Architecture Quantum Identification

![architecture-quantum-identification](assets/architecture-quantum-identification.png "architecture-quantum-identification")

* Grouping the identified aggregates by the services that operate on them, and the services by shared scalability, availability, and change-cadence needs, produced six candidate quanta in the first pass — **Analytics, Maintenance, Visitor, Feedback, Staffing** and **Marketing**.
* Since the Feedback Service has low, steady traffic and no distinct scaling or availability profile of its own, it doesn't warrant a dedicated quantum — it was folded into the **Visitor** quantum, which it functionally supports.
* Maintenance (rides + animal enclosures + sensor data) has real-time, safety-critical requirements that the other quanta don't share, so it was kept independent rather than merged with Visitor or Staffing.
* So finally, we have **Analytics, Maintenance, Visitor, Staffing** and **Marketing** as the architecture quanta for the Von Digitalis Estates system.


## Architecture characteristics

Refer to [detailed architecture characteristics analysis](design_docs/architecture-characteristics-styles.md).


| Quantum | Top 3 Characteristics | Others Considered | Style | Why |
| --- | --- | --- | --- | --- |
| **Visitors** | Availability, Performance, Scalability | Concurrency | Microservices | Highest-traffic surface; must scale independently for 3x visitor growth |
| **Staffing** | Fault Tolerance, Availability, Responsiveness | Consistency | Event-driven | Deployment suggestions are useless if late or lost |
| **Maintenance** | Data Integrity, Extensibility, Deployability | Data Consistency, Security | Microservices | Safety/welfare-critical; a wrong reading is worse than a slow one |
| **Analytics** | Data Integrity, Interoperability, Adaptability | Data Consistency, Fault Tolerance | Event-driven | Aggregates every other quantum's data; must stay correct and pluggable |
