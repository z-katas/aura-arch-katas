# Golden Path — Actor Lifecycles

In Event Storming, the **golden path** is the sequence of events when a process completes exactly as intended — no exceptions, no errors. It's established early to give the team a shared timeline before layering in edge cases and policies.

Each lane below is one actor's golden path — the steps that must succeed, in order, for their session to count as a success.

![Golden path swimlanes](/assets/golden_path.png "Golden path swimlanes")

## What the board shows

Five actors, five lifecycles, one shared pattern: **Start → Log in → core responsibilities → exit**. The Visitor's path is further split into three phases (discover & book, on-site visit, checkout), marked by the dashed dividers, since it's the only lifecycle spanning both a remote and an on-site experience.