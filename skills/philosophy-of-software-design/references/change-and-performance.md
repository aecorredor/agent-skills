# Change, consistency, and performance

## Modifying existing code

After the change, the code should look as if this requirement had been part of the original design. If you are not improving the design a little, you are probably making it worse. Each smallest-possible patch adds a special case, and the system's design is mostly the history of those patches.

If a full reshape takes months and a hack takes hours, look for a design that is nearly as clean in a few days. If you must ship the hack, schedule the cleanup as real work. Do not leave it as an intention.

Do not redesign on every bugfix. Do not gold-plate when a smaller improvement was available and a deadline is real.

## Consistency

Similar things should work the same way, and different things should look different, so a reader can reuse a conclusion.

Keep consistent: names for the same concept, local style, one interface with several implementations, a pattern that actually matches the problem, and invariants such as a single representation for "no value."

Follow the existing convention unless you have new information and the new way is enough better to update every old use so the old way disappears. Do not force unlike things into one pattern so that "it looks like X" no longer means it is X.

Write global rules where newcomers will see them. Enforce mechanical rules with a check that blocks the commit.

## Obvious code

Obvious code is code whose first quick guess is right. A reader decides, especially in review.

What helps: precise names, consistency, whitespace that shows where one part ends, a comment aimed at the exact confusion.

What obscures, and the fix:

- A handler that is not called at the use site. Say on the handler when it runs and on which thread.
- A generic pair, so callers remember an order. Define a small named type with field names.
- A declared type that is broader than the performance or safety the reader must assume. Align the declared type with that assumption.
- Control flow that violates a platform expectation. Say so at the point the reader will stall.

Do not ban events, generics, or interfaces to chase obviousness.

## Practices, judged by complexity

- Interface inheritance (shared signatures, separate implementations) is useful when one set of caller knowledge covers many mechanisms. Implementation inheritance couples parent and children. Prefer a small helper each type uses. If you inherit implementation, keep parent state behind parent methods. Privacy does not fix a shallow type.
- Build in increments. Delay an abstraction until a feature needs it. Once it is needed, design it as a whole and make it somewhat general. The increment is the abstraction, not a pile of special cases.
- Unit tests make structural change safe enough to attempt. Without them, people minimize the diff and the design decays. Tests do not choose the design.
- Writing tests before production code has no natural moment to design the abstraction. For a bug, write a failing test that exhibits it, then fix it.
- Use a design pattern when the problem matches it. A pattern that does not fit is extra machinery.
- Prefer operations that hide data. A getter or setter that publishes a field adds a shallow method. Use an accessor when you need a place to enforce a constraint later, not as a default.

## Performance

Clean design and speed usually move together. Deeper modules do more per call and cross fewer layers.

1. Know the costly operations from measurement: network round trips, durable I/O, allocation, cache misses. Do not tune from a hunch.
2. Do not tune every statement, and do not ignore cost. A system full of small inefficiencies has no single fix later.
3. Hidden implementation complexity can be worth it on a measured hot path. Interface complexity is a high price. A better algorithm or one less layer beats micro-tuning.
4. Write the shortest common-case sequence you can imagine, with special cases ignored. Call that the ideal.
5. Redesign so the real code stays close to that ideal. One test at the start should reject uncommon cases. Handle those cases off to the side, written for clarity.

If the number does not move, remove the complexity unless the change also simplified the code. Start with the simple design unless you already know this path dominates.
