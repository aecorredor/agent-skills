---
name: game-programming-patterns
description: >-
  Chooses language-neutral game architecture for time, entities, decoupling,
  and hot-path layout. Use when designing a game loop, entity model, input,
  undo, replay, events, services, state machines, content-driven behavior,
  object lifetime, spatial queries, or cache layout in any engine or language,
  or when the user mentions game architecture, frame timing, coupling, or
  which pattern fits a gameplay problem.
license: MIT
metadata:
  version: "1.0.0"
---

# Game programming patterns

Pick the smallest structure that removes a real coupling or a measured cost. These patterns are language-neutral and engine-neutral. Samples in the source book are C++. Do not copy that style into the project. Match the language and engine already in use.

Ideas here are informed by Robert Nystrom's *Game Programming Patterns*. This skill is an original decision procedure, not a reproduction of that book.

Add an abstraction only once that spot needs to flex. Stay flexible until the design settles, then remove abstraction where a profile says it matters. Get the data layout right before inventing an algorithm. A prototype that might ship should be written as if it will stay.

## How to choose

1. Name the concrete problem: wrong time, wrong owner, too many kinds, or a measured hot path.
2. Take the first row below that matches. Read that reference before writing code.
3. Prefer the "reject when" column over the pattern. A direct call, a fixed step, or a breed record is often the whole design.
4. Do not stack command, observer, and an event bus on one signal. Pick the distance from a direct call that the problem actually needs.

| Problem | Look here | Reject it when |
| --- | --- | --- |
| Frame rate, physics step, or who owns the loop | [references/sequencing.md](references/sequencing.md) | The engine already calls you. Do not add a second loop. Do not feed raw frame delta into physics or lockstep play. |
| Actors must tick, spawn, or die without skipping or double-updating | [references/sequencing.md](references/sequencing.md) | The rules are turn-based. Do not tick every piece every frame. |
| Readers must not see a half-drawn frame or a half-updated world | [references/sequencing.md](references/sequencing.md) | Nothing reads during the write, or you need ordered conflict resolution. |
| Input rebinding, undo, replay, or one action vocabulary for player and AI | [references/decoupling.md](references/decoupling.md) | The binding is fixed and nobody needs history. |
| A separate feature must hear an event it should not be called from | [references/decoupling.md](references/decoupling.md) | You need both sides in mind to understand the feature, or the path is hot. |
| Work must happen later, on another thread, or as a batch | [references/decoupling.md](references/decoupling.md) | The caller needs a return value now. |
| One entity type owns several domains, or inheritance cannot mix capabilities | [references/decoupling.md](references/decoupling.md) | The type is still small. Pieces would only add wiring. |
| Many call sites need one ambient facility | [references/decoupling.md](references/decoupling.md) | The dependency belongs on the API. Pass it in. |
| Modes, booleans, and input branches allow impossible combinations | [references/behavior.md](references/behavior.md) | One or two flags still match the design, or the behavior is open-ended AI. |
| Many kinds differ by data, not by code | [references/behavior.md](references/behavior.md) | A handful of kinds have real, different code. |
| Many variants share a set of engine operations | [references/behavior.md](references/behavior.md) | They differ only by ids and magnitudes. Store data. |
| Designers must change behavior without a rebuild, safely | [references/behavior.md](references/behavior.md) | You cannot build the authoring tool, or the script would run on the hottest path. |
| Huge numbers of similar objects repeat the same heavy data | [references/optimization.md](references/optimization.md) | Counts are small, or instances differ in the heavy data. |
| A hot loop chases pointers | [references/optimization.md](references/optimization.md) | The path is cold, or a profile says the time is spent elsewhere. |
| Derived data is recomputed more often than it is read | [references/optimization.md](references/optimization.md) | You read it after every write, or you can patch it incrementally. |
| Create and destroy churn fragments memory or shows up in allocation | [references/optimization.md](references/optimization.md) | Creation is rare, sizes vary wildly, or a compacting collector already solves it. |
| "Who is near me?" dominates and the set is large | [references/optimization.md](references/optimization.md) | The set is small, or objects move so often that relinking costs more than the pairs you skip. |

## Defaults that are usually wrong

- A process-wide singleton as the way to reach audio, logging, or randomness. One owned instance is fine. Global reach is the problem. Pass the dependency, hang it on the root you already have, or use a locator only for a truly ambient facility. Details in [references/decoupling.md](references/decoupling.md).
- One subclass per spell, item, or breed that reaches into unrelated systems.
- A variable timestep because the engine hands you a delta.
- A second full copy of world state "so updates look parallel," which delays every reaction by a frame and throws away ordered updates.
- A component, event bus, or script machine added because the pattern is available.

## How the pieces fit

The loop calls the update walk. The display flip is the graphics form of a double buffer. Double-buffer actor state only when same-frame reads must not see same-frame writes.

Components are the usual home of the per-frame update once domains must be mixed. Store each hot domain in its own packed array when a profile says pointer chasing matters.

State, type object, and bytecode are three places behavior can live: what the actor is doing right now, what kind it is, and a script the kind runs. A sandbox is the code form of "many variants, one set of powers."

Command, observer, and event queue are three distances from a direct call. Flyweight, object pool, and spatial partition solve three different costs: repeated heavy data, allocation churn, and proximity.

## Three speeds

Long-term change speed, execution speed, and today's feature speed pull apart. Flexible code prototypes faster and costs cycles. Optimized code runs faster and locks assumptions. There is no correct point. Choose which cost you are paying, and say so in the design.
