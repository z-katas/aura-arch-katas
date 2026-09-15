# Architecture Views

This directory contains the architecture views and operational models for the AURA estate platform.

- [C1 System Context](./c1-system-context.md)
- [C2 Container View](./c2-container-view.md)
- [Operational Flows](./operational-flows)

The implementation follows the plan in `plan.md` and preserves the four quanta and edge-first async messaging model. The Maintenance quantum includes local edge sensing and store-and-forward buffering, while the Visitor quantum includes booking, pass issuance, feedback collection, and NLP recovery processing.
