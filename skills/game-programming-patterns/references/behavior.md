# Behavior

Where variety lives: the mode right now, the kind of thing, a shared set of operations, or a script. Pick one home. Do not build all four for the same content.

## State

Let an object change behavior as its mode changes, and keep each mode's behavior and mode-only data together.

Use it when behavior switches among a small set of exclusive modes over a series of inputs: a character, a menu, input handling, a parser, a protocol. Skip it when one or two flags still match the design, or when the behavior is open-ended planning. A finite machine is not a general computer. Do not turn every conditional into an object.

Draw the modes and transitions before writing branches. Start with one enumeration and a branch on the mode, so everything about one mode sits together. Move to a mode object when that mode owns data or a per-frame tick. The owner holds the current mode and forwards calls. Swapping the reference is the transition.

Put entry work on the destination so every inbound transition does not repeat it. Do not destroy the current mode inside its own method. Return the next mode and let the owner replace it after the call returns. Modes with no fields can be one shared instance. Modes with fields are created per owner.

Two independent axes (what the body is doing, and what it is holding) should be two machines, so the counts add instead of multiplying. Shared transitions belong in a superstate. A temporary mode can be pushed and popped so you do not clone "firing while standing," "firing while running," and "firing while jumping."

A cross-check of the other machine's mode is crude and usually enough. Stateless modes are flyweights. The per-frame call is the update method in [sequencing.md](sequencing.md).

## Type object

Represent each kind as data, so new kinds are records rather than new types in the language.

Use it when kinds are numerous, unknown until runtime, or must change without a rebuild. Skip it when a handful of kinds have real, different code. Type records make data easy and behavior hard. Those kinds belong in ordinary types or in components.

The breed record holds what every instance of that kind shares. The instance holds what varies per individual, plus a reference to the breed. Load breeds from content. Keep the breed alive as long as any instance points at it. The language will not do that for you.

Per-kind behavior does not fall out of an overridable method. Store a choice among a fixed set of functions, or a script, when a breed needs real behavior. Hiding the breed lets the instance override a field and adds a forwarder per public field. Exposing the breed lets other systems use it with no live instance, and widens the instance's interface.

Changing breed on a live instance avoids allocating a replacement. Check the new breed's assumptions. Single inheritance between breeds is the practical maximum: walk the parent on each read, or copy inherited fields down at construction. Do not give a breed multiple parents. Making the breed the only constructor is how you force spawns through a pool.

A breed is what the thing is. A state is what it is doing right now. Do not make one record do both jobs unless you mean to.

Content families can also share fields through one parent link on the record: missing fields are read from the parent. Pick a real simple entry as the parent, not an abstract kind. Copying a live actor is a different, sharper tool. Use it only when the copy rules are obvious, including what happens to owned objects. Otherwise pass a type, a spawn function, or a breed.

## Subclass sandbox

Let many variants implement one hook by calling operations the base provides, so they never touch the rest of the engine.

Use it when you already have many derived types, they overlap, and the base can offer every operation they need. Skip it when the variants are the same control flow with different ids and magnitudes. Store those fields on a breed instead.

The base has one overridable method and a small set of protected operations: move, play a sound, spawn an effect. A variant implements the method using only those operations. Lift a duplicated sequence into a new operation. An operation used by one or two variants does not belong on the base.

The failure mode is the brittle base: it couples to every system, and every variant couples to it. When the operation list outgrows the base, hand out helper objects (a sound player with play, stop, and volume) instead of adding more methods. A pure forwarder is still useful when it hides a handle the variant must not touch.

Dependencies for those operations should not become a singleton hiding in the base. Pass them at construction if every variant can forward them, or wrap creation so a second init step cannot be skipped. A locator is the last resort, described in [decoupling.md](decoupling.md).

## Bytecode

Encode behavior as data: a flat list of instructions a small machine in the game executes.

Use it when a lot of behavior must live outside the engine language, you can build the authoring tool, and this code is not on the hottest path. Skip it when the variety is a few numbers (use a breed), when authors are already programmers in a fast language (call the operations directly, or use the sandbox), or when you cannot show what the script did.

The instruction set is the only door out of the script. Intermediate values travel on a stack so nested expressions become a flat sequence. Start with a stack machine and one value type. If you need more types, tag them. Do not add jumps until you also have a step limit. Without jumps, a size limit bounds runtime.

The authoring experience is the product. A text language needs real syntax errors. A constrained tool can refuse invalid programs as they are built. Plan a way to see the high-level script, not only the machine, before the content scale makes that mandatory.

A tree of behavior objects is the simpler model and the wrong runtime: many small objects and indirect calls. The flat stream is how you get density. It is still slower than native code. Do not let the instruction set grow into an unplanned language, and do not run it on the hot path.

A breed can point at a script when the kind needs behavior and not only fields. That is the step past the sandbox, not a replacement for data that has no control flow.
