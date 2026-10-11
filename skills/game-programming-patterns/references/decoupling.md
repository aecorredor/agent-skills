# Decoupling

Who may know whom, and when a call actually runs. Start with a direct call. Add distance only for the reason in the matching section.

## Command

Turn a call into a value you can store, pass, queue, delay, and undo.

Use it when controls must be rebound, or the same actions must be issued by a player, by AI, by a replay, or by undo. Skip it when the binding is fixed, there is one actor, and nobody needs history.

A command is a small record: the operation, its arguments, and optionally the prior fields needed to reverse it.

- A reusable command is the action itself, shared by every binding that invokes it.
- A one-shot command is a specific moment, created when someone chooses, then stored.

Input code can return the command instead of running it, so a later step picks the actor. An empty command covers an unbound control so callers need no null check. A closure is enough for one operation. An explicit record is better when you need both do and undo, or when readers must see which fields were captured.

Undo stores only the fields the command changed. A new command after an undo drops the redo tail. Every mutation that must be undoable has to pass through a command or the history lies. Replay requires the simulation to be deterministic given those commands.

A command queue is the step toward an event queue. A stateless command shared by every binding is a flyweight. Do not make the command object a global.

## Observer

Announce that something happened without naming the receivers.

Use it when a feature maintained separately must react to a thin signal, and you do not need both sides in your head to understand either one. Skip it when the link should be visible to search, on the hottest loops, or where a listener might take a lock the source holds.

Prefer a small event object ("something fell") over making the whole source the subject. The source still chooses which signals exist. Notify walks the current listeners and calls them immediately. Listeners must not depend on order. Finish quickly, or hand the work to a queue. Unregister in the listener's teardown. Do not mutate the list inside a callback unless notify is written to tolerate it.

With garbage collection, a source that retains listeners keeps them alive. Opening a screen again adds another, and invisible listeners keep running. Who will be called is a runtime list, so a bug that crosses the signal is harder to trace than a direct call.

## Event queue

Separate the moment a message is sent from the moment it is handled.

Use it when the sender must not wait, the work must run on a specific thread, or the consumer needs the whole batch (stacked sounds, a voice limit). Skip it when the caller needs a return value, or when only the receiver must change. A single bus for the whole game is a global in a protocol costume.

Send enqueues and returns. The consumer pulls on its own loop, then handles, batches, or drops. A ring of plain slots avoids per-message allocation. Leave one slot unused so empty and full differ. A full ring must refuse or overwrite. Decide which.

The world can change before the item is handled. Put the facts the handler will need into the message. Do not enqueue while handling. Pass ownership of the record to the queue, share ownership, or let the queue own the slots so the sender fills a record that already lives there.

- Direct call: cheap, same thread, result needed now.
- Observer: many listeners, and they can run now.
- Command: a request to do something later, usually one consumer.
- Queue of notifications: many listeners, deferred.

## Component

Let one entity cover several domains while those domains stay unaware of each other.

Use it when one type owns input, physics, animation, rendering, and audio, or when inheritance cannot mix those capabilities without duplication. Skip it when the type is still small. Wiring and extra pointer hops cost more than a single type.

The entity is a thin container. Each domain owns its data. Shared facts every piece needs, such as position, stay on the container. Do not push domain-specific fields up onto the container.

How pieces talk, from the simplest that works:

1. Shared container fields for universal state.
2. A direct reference for one tight pair.
3. A fire-and-forget message through the container when the sender must not name the receiver.
4. An id lookup only when the set of pieces is open. If every interaction searches a registry during update, you have rebuilt the knot.

Input runs before physics, physics before drawing, unless you intentionally render the previous frame. A stateless piece can be shared. A piece that holds identity-defining state should not.

At the extreme, the entity is an id and each domain is a collection keyed by that id. That layout is also the doorway to packing hot data. See [optimization.md](optimization.md) only when a profile says the pointer chase matters.

## Service locator

Offer one access point for a service without naming the concrete type at every call.

Passing the dependency is the default. Use a locator only when an ambient, effectively singular facility (logging, allocation, the one audio device) would otherwise put irrelevant parameters on APIs that are about something else. Skip it when the dependency is part of the real API, or when the service is valid only in one phase of the frame. A global locator cannot enforce that phase.

Startup registers a provider. Callers see the interface. If nothing is registered, return a do-nothing implementation and log the hit, so a missing registration does not crash and does not stay silent. Initialize the locator to that stand-in so the accessor does not branch.

The cost: reading a call no longer shows what it relies on, use-before-provide fails, and anyone can call it. A reflective config lookup spends complexity the player never sees. Prefer a parameter, or a service bag on the game or world you already have, over a process-wide locator. Do not invent a base type just to share the locator.

## Singleton

One instance and a global way to reach it are different problems. Keep one owned instance when an external system truly cannot have two coordinators. Do not publish a global getter.

A global makes the call harder to reason about, invites unrelated systems to reach through it, and is shared by every thread. Lazy creation hitches the frame on first use. If all you need is a group of functions, write the functions.

If a caretaker only creates and queries another object, put that behavior on the object. Pass a dependency when it is part of the operation. Hang cross-cutting facilities on the root you already have, or resolve them through the locator above. Do not add a logging parameter to every function just to avoid a global.
