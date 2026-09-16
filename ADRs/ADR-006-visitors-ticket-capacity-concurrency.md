# ADR: Concurrency Control for Ticket and Slot Capacity

## Status
Accepted

## Context
Per our [architecture characteristics analysis](../design_docs/architecture-characteristics-styles.md), the Visitors quantum is driven by **Availability, Performance, and Scalability**, with **Concurrency** explicitly called out as a characteristic considered for this quantum. Ticketing must scale from today's ~5,000 visitors/day to a 15,000/day target, and family passes/slot-based tickets are sold against **finite per-ride and per-slot capacity** tracked by the Capacity and Inventory Service — see the [Visitors quantum architecture](../assets/visitors-quantum-architecture.png), where Catalog → Capacity → Order sits on the critical checkout path, and the [sequence](../assets/visitors-quantum-sequence.png), which shows both the reservation-succeeds and version-mismatch branches of a booking attempt.

At peak (a popular ride's morning slot, a holiday weekend), many visitors can attempt to book the same limited capacity at the same moment, from the Web/Mobile App and On-site Kiosk simultaneously. Two failure modes are both unacceptable: **overselling** a slot beyond its real capacity (a safety and operations problem for rides with fixed throughput), and **needlessly serializing** every booking attempt behind a lock, which would cap throughput well below what Performance/Scalability require at 3x growth.

We considered three alternatives:

1. **Pessimistic locking on each slot** — acquire a row/slot lock before checking capacity, hold it through payment. Guarantees no overselling, but serializes all attempts on a popular slot behind a single lock, including the (often slow) payment step — directly at odds with Performance and Scalability under peak concurrent load.
2. **No concurrency control, reconcile after the fact** — let all requests proceed and detect overselling afterward (e.g., a nightly reconciliation job that cancels excess bookings). Maximizes throughput, but a visitor can walk away believing they hold a confirmed slot and later have it revoked — unacceptable for Availability and trust, and operationally messy at a working ride.
3. **Optimistic concurrency control on the Capacity and Inventory Service** — reads the current remaining capacity and a version/token, attempts a conditional decrement, and only proceeds to Order/Payment if the decrement succeeds; a losing request is told immediately that the slot is gone (or offered the next available one), with no lock held across payment.

## Decision
We adopt **optimistic concurrency control** in the Capacity and Inventory Service.

Specifically:
- Each ride/slot capacity record carries a version (or equivalent token). A booking attempt reads the current remaining count and version, and submits a conditional update ("decrement by N only if version still matches").
- A successful conditional update reserves the capacity **before** the Order and Checkout Service proceeds to Payment; a losing request fails fast and is shown accurate remaining capacity (or an alternative slot) rather than being queued behind a lock.
- No lock is held across the Payment Gateway call — the slow, external part of checkout never blocks other visitors from competing for remaining capacity.
- A reserved-but-unpaid slot (payment abandoned or timed out) is released back to available capacity after a short, defined hold window, so failed checkouts don't quietly shrink real capacity.

## Consequences

**Positive:**
- **No overselling** — a conditional update either succeeds (capacity genuinely reserved) or fails visibly to the visitor; there is no window where two visitors both believe they hold the last slot.
- **High concurrency without serialization** — competing requests for the same popular slot are resolved by fast conditional writes, not by queuing behind a held lock, which is what Performance and Scalability require under 3x visitor load.
- **Payment stays off the critical concurrency path** — the Payment Gateway (the least predictable-latency step) never holds up other visitors' ability to compete for capacity.

**Negative / trade-offs:**
- **Visible contention at extreme popularity** — a very popular slot can show many failed attempts in a short window (each told "gone" or offered an alternative) rather than a smooth queue. Accepted: fast, honest failure is better than a queue that silently delays everyone, and the UI can soften this with a queueing/waitlist affordance if it becomes a frequent visitor complaint.
- **Held-but-unpaid reservations need a hold-window policy** — too short, and slow-but-genuine visitors lose their slot mid-payment; too long, and abandoned checkouts hold up real capacity. This needs a concrete, tuned value (and monitoring), not a one-time guess.
- **Retry logic on the client** — the Web/Mobile App and Kiosk must handle a failed conditional update gracefully (retry against fresh capacity, or offer an alternative), rather than treating it as a hard error — a small but real added client complexity compared to a naive "lock and wait" approach.
