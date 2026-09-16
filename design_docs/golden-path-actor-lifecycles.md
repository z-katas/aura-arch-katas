# Golden Path — Actor Lifecycles

In EventStorming, the **golden path** is the sequence of events when a process completes exactly as intended — no exceptions, no errors. It's established early to give the team a shared timeline before layering in edge cases and policies.

Each lane below is one actor's golden path — the steps that must succeed, in order, for their session to count as a success. Failure branches (failed safety checks, payment retries, flagged anomalies) are documented on the individual event-storming boards, not here.

![Golden path swimlanes](/assets/golden_path.png  "Golden path swimlanes")

## What the board shows

Five actors, five lifecycles, one shared pattern: **Start → Log in → core responsibilities → exit**. The Visitor's path is further split into three phases (discover & book, on-site visit, checkout), marked by the dashed dividers, since it's the only lifecycle spanning both a remote and an on-site experience.

## Observations

- **Two lanes end in a named outcome instead of a plain "End."** The Visitor lane closes on *Happy visitor experience* and the Estate Owner lane on *Single digital platform* — both kept as board-level goals rather than generic terminations, since they are the two outcomes the rest of this architecture is accountable to.
- **Every staff lane follows the same shape** (log in → do the job → log out), which is what let Staff be treated as a single reusable session pattern across Ride, Animal, and Front Office contexts rather than three bespoke ones — see the [component identification diagram](../assets/architecture-quantum-identification.png) for how that shows up in the service boundaries.
- **The Estate Owner lane has no explicit "Log out"** before its outcome, unlike the other four. The estate dashboard is a persistent operations view, not a visit session that ends when a guest leaves.
- **Ride and Animal staff both end their lane on a planning/prevention step** (Log incidents; Plan enrichment) rather than the customer-facing action itself — both roles are already oriented around *what happens next*, which is the seam the AI-assisted monitoring use cases plug into.

