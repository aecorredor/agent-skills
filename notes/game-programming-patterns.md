# Study notes: Game Programming Patterns

Working synthesis of Robert Nystrom, *Game Programming Patterns* (2009–2014). The book's samples are C++. These notes keep the ideas language-neutral so they apply in any engine. They are original operational notes, not a reproduction of the book. The book is licensed CC BY-NC-ND; do not copy its prose or listings into skills or docs.

Each pattern below is something to reach for when the symptom matches, not a default layer.

## What the catalog is for

The book sits between two other kinds of game book. Domain books teach one specialty and rarely how that chunk meets the rest of the game. Whole-engine books walk one genre end to end and go stale when the game is not that genre. This catalog is a set of independent architecture ideas for the code between those systems. Mix them for the game you have. Do not clone an engine from them.

Games make four pressures constant: things must happen in the right order and at the right time; many people must iterate behavior without trampling a shared codebase; those behaviors then interact without becoming one knot; performance is part of the product. The same ideas apply outside games.

A pattern is shapeless until it is applied. The useful reference shape is: intent, the problem, the shape of the solution, when to use it, when to skip it, the costs, and the neighbors.

## Architecture before patterns

Architecture matters only because code changes. If nobody will edit it, the design is irrelevant. The expensive step is loading the relevant code into working memory. Writing the change is often the small part.

Two pieces are coupled when you cannot understand one without the other. Split them and a change that touches only one side does not drag the other into your head. The aim is to minimize the knowledge you must hold before you can make progress.

That has a carrying cost. Every extension point is a bet that you will need that flexibility. If the bet is wrong, the extra layer is more code to debug and more scaffolding to trace before you reach the code that does something. Past a point, the abstractions themselves fill the working memory that decoupling was supposed to free.

**Three speeds conflict.**

- Long-term change speed wants a clean structure and a little extra care on every edit.
- Execution speed wants optimization time and then resists further edits.
- Today's feature speed wants the shortcut, which taxes tomorrow.

There is no correct point on that triangle. Flexibility means fewer baked-in assumptions: call through an interface, talk by message, allow one more listener later. Optimization uses the opposite: a hard cap, one concrete callee, one homogeneous contiguous collection. Flexible code prototypes faster and costs cycles. Optimized code runs faster and locks assumptions. Stay flexible until the design settles, then remove abstraction where a profile says it matters. Making a fun game quickly is the easier direction. Making a fast game fun afterward is the hard one.

**Simplicity.** Get the data layout right, then the algorithm. Less code means less to load into your head and often less to run. A short solution is distilled, so it often takes longer to find than a pile of special cases.

**When to defer.** Add an abstraction only once you are confident that spot needs to flex. Think about performance the whole time, and postpone the low-level optimizations that freeze assumptions until late enough that the design can still move and early enough that you are not discovering a slideshow near ship. Throwaway prototypes are legitimate only if they will actually be thrown away. If there is a real chance the prototype becomes the game, write it as if it will stay.

## Sequencing

### Game loop

**Intent.** Keep game time moving without waiting on input, and at a rate that does not follow the CPU.

Batch programs run and exit. Event loops block until input. A game has to keep simulating and drawing while the player does nothing. If each crank of the loop is "one unit of game time" and you spin as fast as the chip allows, the game speeds up on faster hardware and slows down in heavy scenes.

One continuous loop: non-blocking input, then advance simulation, then draw. Track real time so the advance stays consistent. An engine is the thing that owns this loop and calls your code. A library is the thing you call from a loop you own. Do not write a second loop inside an engine that already calls you.

Variants:

1. **Platform event loop.** The host calls you. You lose control of timing. Use this when the platform will not let you own the loop. Inside the callback you can still apply the timing rules below.
2. **Fixed step, no clock.** Simple, and game speed tracks hardware and scene cost. Do not ship this if the same build runs on more than one machine.
3. **Fixed step with sleep.** Sleep until the frame budget is spent. The game cannot run too fast, and spare cycles can go to sleep. If the frame runs long, the whole game slows, because simulation and drawing are still tied together.
4. **Variable step.** Scale motion by elapsed real time. It stays in real time and looks smoother on fast machines. Most of the time this is the wrong default. Floating-point error accumulates differently when one machine takes many steps and another takes few, so the same shot lands in different places. Physics damping is tuned to a specific slice. The simulation becomes non-deterministic because the clock is outside the program.
5. **Fixed simulation step, variable drawing.** Accumulate real time as lag. While lag covers one fixed slice, tick once and subtract the slice. Then draw once. If the machine is slow you drop draws instead of slowing the rules. This is the adaptable option when you own the loop and you have physics or lockstep play. Tune the slice short for fidelity, but longer than a single tick costs on the slowest target. If one tick takes longer than the slice, lag grows forever. Cap the catch-up ticks. The slice is not the display rate. Pull drawing out of the inner loop.

After the inner loop, leftover lag divided by the slice is how far you are into the next step, a fraction from 0 up to but not including 1. Draw moving things that far along their current velocity so they do not stick to the last tick. That position is a guess. Wrong guesses are usually less visible than the stutter of not guessing.

Uncapped spinning spends every spare cycle on frame rate. Clamp and sleep the rest when battery matters. Turn-based play still needs the loop: the rules wait, animation and audio do not. If the OS already has an event loop, either own the loop and poll without blocking, or let the host call you.

### Update method

**Intent.** Simulate many independent objects by asking each to run one frame of its own behavior.

A patrol written as an inner loop never returns, so input and drawing freeze. Lifting every actor into one function mashes the level into one place.

The world holds a collection of actors. Each actor advances one frame and stores whatever it needs to resume. Each frame the loop walks the collection. Actors can be added from data. They look simultaneous. Inside the frame they run one after another.

Use it when many objects must run every frame, each mostly on its own. If the pieces are like chessmen, do not tick every piece's rules every frame. Sequential rules keep a single valid world. Simultaneous rule updates create conflicts.

Costs:

- Straight-line behavior becomes explicit resume state. State machines are the usual place to keep "where I left off." Generators or coroutines can pause mid-behavior. Real OS threads are too heavy.
- During its tick an actor sees neighbors that have not ticked yet in their previous state, and neighbors that already ticked in their new state. If order must not matter, snapshot with a double buffer. If you want one valid world after another, and a serial list of moves, do not snapshot.
- Appending an actor mid-walk means it ticks on the frame it was born. Cache the count if new actors should wait. Removing an earlier slot skips the next actor. Walk backward, or mark dead and compact after the walk. If several threads tick the list, defer structural edits.
- A large variable slice overshoots patrol endpoints. Prefer a fixed slice.
- A single list of disabled actors burns time on empty checks and blows the cache. Split live versus inactive when many actors are idle. Keep one list when few are.
- The tick can live on the actor, on a component, or on a delegated state or breed. Subclass-per-behavior gets brittle. Swapping a delegate changes behavior without a tall hierarchy.

### Double buffer

**Intent.** Make a sequence of edits look instantaneous to every reader.

Two jobs:

- A display scans a frame while it is still being drawn and shows a torn picture.
- Simulation code reads the same fields it is writing in the same pass. Who reacts this frame depends on list order, so the world does not look simultaneous.

Keep two copies. Readers always use the current copy. Writers only touch the next copy. When the edit is finished, one atomic swap publishes the next copy.

- **Repoint.** Exchange references. Cheap, independent of size. Callers must not keep pointers into the buffer. Successive frames alternate copies, so leftover contents are from two frames ago, not the previous frame. That matters if you blend with the previous image. Unavailable if the visible buffer must sit at a fixed address.
- **Copy.** Write the finished next state over the current state. The published data is only one frame old. The copy can cost more than the edit. Fine for a flag. Wrong for a full frame.

For interactions, run every actor tick against the current snapshot, then swap. A reaction becomes visible on the next frame, in the same way no matter the order.

Use it only when state is edited incrementally, something may read it mid-edit, readers must not see the partial edit, and readers should not wait for the writer. If nothing reads during the edit, do not keep a second copy. If actors should resolve conflicts in order, a snapshot hides information you wanted. A swap slower than the edit erases the benefit. Almost every graphics API already swaps the display buffer.

## Classic patterns, in game terms

### Command

**Intent.** Turn a call into a value you can store, pass, queue, delay, and undo.

Raw input wired straight to actions cannot be rebound. The same actions need to be issued by a player, by AI, or by a replay.

A command is a small record: the operation, the arguments, and optionally enough prior state to reverse it.

- A reusable command is the action itself, shared and invoked whenever a control fires.
- A one-shot command is a specific moment (this actor, this destination), created when someone chooses, then stored.

Input handling maps each control to a command and can return that command instead of running it. AI emits the same kind of values. A queue between producer and consumer lets you delay, serialize, or replay. For undo, each command remembers only the fields it changed. A new command after an undo drops the tail. An empty command covers an unbound control so callers need no null check. A closure is enough for a single operation. An explicit record is better when you need both do and undo.

Prefer a direct call when the binding is fixed, there is one actor, and you will never queue, undo, or replay.

Every mutation that must be undoable has to pass through a command or the history lies. Replay depends on the simulation being deterministic given those commands. Copying a whole object to undo a two-field change wastes memory.

Neighbors: a command queue is the doorway to an event queue. A stateless command shared by every binding is a flyweight.

### Flyweight

**Intent.** Share the heavy identical part of a huge number of similar objects, and store only what differs per instance.

Split each object into intrinsic state (mesh, textures, movement cost: context-free and immutable) and extrinsic state (where it is, how big, what tint). Many instances hold a reference to one shared record plus a few small fields. For a tile map, the grid stores a small id into a handful of immutable terrain records.

Use it when instance counts are large and most of the bytes are repeated. Skip it when there are few instances, each one is actually unique, or a packed table of ids is already clear. Indirection can miss cache. Measure before you adopt sharing only for style, and measure before you abandon it for a "faster" encoding.

Shared records stay immutable, because a write shows up everywhere. Their lifetime must outlive every user. Sharing must not change visible behavior. It is an optimization.

A type object also points many instances at one shared definition, but its goal is fewer classes. Flyweight is only about cost. Stateless modes can be one shared instance across many machines.

### Observer

**Intent.** Let one piece of code announce that something interesting happened without knowing the receivers.

The source, or better a small object that represents one kind of event, keeps a list of listeners. A notify walks the list and calls each listener immediately. Prefer "subscribe to the fell event" over "the physics object is itself the subject." The source still chooses which signals exist, so the split is partial.

Use it across lumps of the program that are maintained separately and only need a thin signal. Prefer a direct call when you regularly need both sides in your head. Then the link should be visible to search. Also skip it on the hottest loops, and where a listener might take a lock the source holds.

Notify is synchronous. A slow listener stalls the source. Finish quickly or hand the work to a queue. Listeners must not depend on order. Do not mutate the list inside a callback unless notify tolerates it. Unregister in the listener's teardown. With garbage collection the same bug is the lapsed listener: the source keeps the listener alive, every open adds another, and invisible widgets keep running. Who will be called is a runtime list, so a bug that crosses the signal is harder to trace than a direct call.

An event queue is the asynchronous cousin. It is a different tool.

### Prototype

**Intent.** Make new things by copying an existing example, or let a content record reuse fields by delegating to another record.

Hand-written copy of a live actor is usually the wrong tool. Copy is semantically sharp: if the creature holds a weapon, does the copy get its own? Prefer a stored spawn function, a spawner parameterized by type, components, or a type object.

The data form is the one to keep. Give a content record an optional parent. Missing fields are read from that parent. Pick the simplest real entry as the parent, not an abstract kind. A unique item is the common record plus a few overrides. The parent link is metadata about the record, not a property of the creature in the world. One level of delegation already removes duplicated tuning.

Use spawn-by-copy only when you truly need a tuned instance's state duplicated and the copy rules are obvious.

### Singleton

**Intent of the classic form.** Guarantee one instance and also offer a global way to reach it. Treat that combination as a warning. The industry kept the global half.

One instance and global access are different problems. A wrapper around an external system that already has global state can require a single coordinator, because two wrappers would not see each other's work. That does not mean every call site should fetch it.

A global makes functions harder to reason about, invites coupling (physics reaches for audio because it can), and is shared by every thread. Encapsulating the global in a type fixes none of those. Lazy init hitches the frame the first time a sound plays. Eager static creation removes the hitch and also removes polymorphism, reliable init order, and the ability to free it. At that point a plain group of functions is simpler.

What to do instead:

- If a caretaker only creates, moves, and queries another object, that behavior belongs on the object.
- If you truly need one instance, enforce that without publishing it. The owner holds the only reference.
- Pass the dependency in when it is part of the operation.
- Do not add a logging parameter to every function. Hang log, files, and audio off the one root you already have, or resolve them through a locator.
- A subclass sandbox can share state with instances without making that state global.

One instance is fine when the external constraint is real and the reference stays in the owner's hands. The harmful half is the global reach.

### State

**Intent.** Let an object change behavior as its mode changes, with each mode's behavior and mode-only data kept together.

A pile of booleans allows combinations that should be impossible, and a timer that exists only while crouching still lands on the character.

A finite state machine is a fixed set of modes, exactly one current mode, a stream of inputs, and transitions. Draw the boxes before writing the branches. The smallest implementation is one enumeration and a branch on the mode first. That is enough for many machines.

When a mode grows its own data and its own per-frame work, give each mode an object. The character holds the current mode and forwards calls. Swapping the reference is the transition. Put setup that every entry should run in an enter action on the destination, so inbound arrows do not each repeat it. Do not destroy the current mode inside its own method. Return the next mode and let the owner replace it after the call returns.

Independent axes (what the body is doing, and what it is holding) should be two machines, so counts add instead of multiplying. Shared transitions belong in a superstate. A temporary mode such as "firing" can be pushed and popped so you do not clone firing-while-standing, firing-while-running, and firing-while-jumping.

Use it when behavior switches among a small set of exclusive modes. Prefer a direct branch when one or two flags still match the design. Prefer a planner when the mode set is really open-ended AI. A finite machine is not a general computer.

Stateless modes are flyweights. The per-frame call is an update method. Strategy also swaps a delegate, in order to detach a behavior from the host. State swaps the delegate so that one host behaves differently over time.

## Behavior as content

### Type object

**Intent.** Represent each kind as an instance of one type record, so new kinds are data rather than new types in the language.

One subclass per breed produces hundreds of tiny types and blocks designers from changing starting health without a rebuild.

Two objects. The breed record holds what every instance of that kind shares. The instance holds what varies per individual, plus a reference to its breed. Load breeds from a file.

Use it when kinds are numerous or unknown until runtime, or they must change without rebuilding. A handful of kinds whose behavior is real code belongs in ordinary types or in components. Type records make data easy and behavior hard.

You now manage the type records yourself: create them, keep them alive as long as any instance points at them. Per-kind behavior does not fall out of an overridable method. The breed can store a choice among a fixed set of functions, or hold a script.

Hiding the breed lets the instance override and keeps the pattern local, at the cost of a forwarder per public field. Exposing it lets other systems use breeds with no live instance, and widens the instance's interface.

Changing breed on a live instance avoids allocating a replacement. It is harder to debug, and the new breed's assumptions need a check. Single inheritance between breeds is the practical maximum: either walk the parent chain on each read, or copy inherited fields down at construction. Multiple parents can remove almost all duplication and is hard enough to skip.

A breed is what the thing is. A state is what it is right now. Making the breed also the only constructor is how you force a pool.

### Subclass sandbox

**Intent.** Let many variants implement one hook by calling operations the base type provides, so they never touch the rest of the engine.

Dozens of powers, each a subclass, duplicate effect code and call into layers they were never meant to see. There is no single place to enforce "all audio from powers is queued."

The base has one abstract method and a set of protected operations: move, play a sound, spawn particles. A new power overrides the one method and implements it only with those operations. Duplicate sequences get lifted into a new operation.

If every power is the same control flow with different ids and magnitudes, skip the hierarchy and store those fields. The sandbox earns its place when the method needs real branching.

The brittle base is the failure mode: every subclass reaches the engine only through it, so the base couples to everything. Operations used by one or two powers do not belong on the base. If the operation list outgrows the base, move operation groups into helper objects the base hands out.

State the operations need can arrive as constructor arguments (every subclass must forward them), two-stage init (easy to forget unless creation is wrapped), static shared state (a singleton in disguise), or a locator.

This is the reverse of template method: the primitive steps live on the base and the varying method lives on the subclass. Components are the escape hatch. Type object and bytecode are the data-driven alternatives when the subclass count explodes.

### Bytecode

**Intent.** Encode behavior as data: a flat list of instructions a small machine in the game executes.

Spells written in the engine language need an engineer and a rebuild to change, and a bad spell can crash someone else's game. You still want a linear instruction stream, not a tree of objects.

Define a tiny instruction set that is the only door out of the script. A spell is a byte array. A loop reads one instruction and dispatches. Intermediate values travel on a stack so nested expressions become a flat sequence. Without jumps the script cannot loop, so a size limit also bounds runtime. Start with a stack machine. One value type if you can. Otherwise a tagged union. An untagged union is unsafe unless you verify the bytes at load.

Someone must produce the bytes. A text language needs a parser and real syntax errors. A constrained tool can refuse invalid programs as they are built. The authoring experience is the actual product. The machine is how you run it quickly and safely.

Use it when a lot of behavior must live outside the engine language, you can afford a front end, and this code is not on the hottest path. If the variety is only a few numbers, store breed data. If you cannot build the authoring tool or a way to see what the script is doing, do not invent the machine. If authors are already programmers inside a fast language, call operations from ordinary code.

Slower than native code. The instruction set grows into an accidental language. Stepping the machine shows the machine, not the high-level source. Plan debug views before the content scale makes them mandatory. An expression tree is the simpler model and the wrong runtime: many small objects and indirect calls.

## Decoupling

### Component

**Intent.** Let one entity cover several domains while those domains stay unaware of each other.

One entity type accumulates input, physics, animation, rendering, and audio. An inheritance tree for "visible," "collidable," and "both" either duplicates a domain or drags unused data into the base.

The entity is a thin container. Each domain lives in its own piece. Shared facts that every piece needs (position, velocity) stay on the container. At the extreme, the entity is only an id and each domain is a collection keyed by that id.

Use it when one type touches several domains you want isolated, or you need many combinations of the same capabilities. Skip it when the type is still small. Wiring and pointer hops cost more than a single class.

How pieces talk, from simplest to most flexible:

- Shared container fields for universal state.
- A direct reference only for a tight pair.
- A message through the container when the note is fire-and-forget.
- An id lookup for siblings only when the set of pieces is open. It becomes a ball of lookups if every interaction searches a registry during update.

Start with shared state and add a path only when a pair actually needs it. Input must run before physics, physics before drawing, or you render last frame's position. A stateless piece can be shared and then behaves like a strategy object. Do not push domain-specific fields up onto the container.

### Event queue

**Intent.** Separate the moment a notification or request is sent from the moment it is handled.

A direct call means "do this now," on the caller's thread. A service then cannot see the whole set of requests in a frame, so the same sound stacks, or the work runs on whichever thread happened to call it.

Send enqueues and returns. The consumer pulls when its own loop is ready, then handles, batches, or drops. A ring of plain slots is the usual store: contiguous, no per-message allocation. One slot stays unused so empty and full can be told apart.

Use it to decouple *when*, not only *who*. Skip it when the sender needs a return value, or when you only need a different receiver (an observer or a command is simpler). A single central bus for the whole game is a global in a protocol costume.

The world can change before the item is handled, so the message must carry the facts that will be gone later. Do not enqueue while handling. A full fixed ring must refuse or overwrite. Aggregation only sees the current depth. Pass ownership of the message to the queue, share ownership, or let the queue own the slots.

- Direct call when the work is cheap, same thread, and the caller needs the result.
- Observer when many listeners must hear something that already happened, and they can run now.
- Command when the payload is a request to do something later, usually with one consumer.
- A notification often has many listeners.

### Service locator

**Intent.** Offer one access point for a service while callers stay independent of the concrete implementation and of how the instance is found.

Passing the dependency is the default. The coupling is visible. Use a locator when threading an ambient, effectively singular facility (logging, allocation, the one audio device) through many layers would put irrelevant parameters on APIs that are about something else.

A registered pointer returned by a function is more configurable than a singleton: startup constructs the provider and registers it. You can swap providers while running. A null object that implements the interface and does nothing is the safest missing-registration policy on a large team, and it should log when it is hit. Initialize the locator to that null object so the accessor does not branch.

The chapter still dislikes the pattern. Dependencies move from the source text to runtime. Registration is temporal: use before provide fails. Anyone can call it, so the service must tolerate every context. A reflective config-file locator spends complexity on something the player never sees. A global locator cannot enforce that a service is only valid in one phase of the frame.

Prefer a parameter, or a service bag hung on the game instance, over a process-wide locator when only a subtree needs the service. Do not invent a common base just to share the locator.

## Optimization

Reach for these after a profile, or when the failure mode is structural (fragmentation, quadratic pair tests) rather than a hunch.

### Data locality

**Intent.** Speed memory access by placing data contiguously in the order a hot loop actually walks it.

A fetch from main memory costs on the order of hundreds of cycles. Hardware fills that gap by pulling a whole cache line of neighboring bytes. Pointer-heavy entity graphs miss constantly. The same work, laid out differently, can differ by a large factor. The ratio depends on the machine.

- Store the pieces themselves in a contiguous array, not pointers to them. The frame walks the physics array, then the render array.
- Splitting the entity into domain arrays matches a loop that touches one domain's fields for many entities. A single array of fat entities pulls cold bytes into a hot walk.
- Fields touched every frame stay in the walked array. Rare data moves to a side structure.
- Keep live items packed at the front of a pool and walk only that count. Swapping a dead item with the boundary is often cheaper than chasing pointers through holes. Profile before trusting that.

Use it when a measured hot path is missing cache. During design, prefer layouts a hot loop can scan linearly, without rewriting cold corners in advance. You give up virtual calls on that path. Entity pointers into a packed array break when you swap for locality. Store an id. Two threads writing the same line bounce it between cores.

Components make this easier because the loop already updates one domain at a time. A pool of one type is the contiguous array. Packing the live prefix is what makes the pool actually local.

### Dirty flag

**Intent.** Skip expensive derived work until something actually needs the result, and then do it once.

A hierarchical scene stores a local transform. Drawing needs the world matrix, the chain of locals combined from the root. Most objects do not move. Eager refresh is worse when several locals on one chain change in one frame: each change recalculates that node and every descendant, and the renderer only needed the last result.

Each node stores the last world matrix and a dirty bit. Writing the local transform sets the bit and does not walk children. The consumer, walking parent before child, recomputes if this node or any ancestor is dirty, stores the matrix, and clears the local bit. One bit collapses a chain of edits into one combine.

Use it when derived data is expensive, primary data changes more often than the result is read, and you cannot cheaply patch the derived value in place. Skip it when you read the derived value after every write, when the value can be maintained incrementally (add and subtract a cargo total), or when the deferred spike would hitch.

The bit must be set on every primary write. The safe shape is one API that both mutates and dirties. A branch around the recompute can mispredict. The parent-dirtiness trick works only when a single traversal is the only reader. Clear the bit at the read, at a checkpoint, or on a background timer, and know which cost you accepted.

### Object pool

**Intent.** Reuse objects from a fixed collection so creating and destroying them does not allocate or free each time.

Particles, projectiles, and voices are created and killed constantly. A fragmented heap can have enough total free bytes and still fail a contiguous allocation. Holding one block for the session avoids that.

At init, create the whole set, all marked unused. Acquire finds a free slot, initializes it, and marks it in use. Release marks it unused. A linear scan is enough for a small pool. When the pool is large and mostly full, a free list threaded through the unused slots makes acquire constant time. Packing live slots at the front optimizes the per-frame walk. The free list optimizes finding a slot.

Use it when you create and destroy often, instances are similar in size, and allocation is slow or fragments, or each object wraps an expensive resource. Skip it when creation is rare, sizes vary wildly, or a compacting collector already removes the fragmentation reason. A simple free list of recycled nodes is enough when you do not need a hard cap or a contiguous walk.

The cap is also a feature: explosions cannot consume the heap needed for a new enemy. On failure, size the pool so important objects never fail, skip a spawn the user will not notice, steal the least important live object, or grow an overflow region. Slots are fixed size, so a pool of subclasses wastes space on the smaller ones. Split into size classes.

Reuse does not clear memory. Forgotten initialization looks almost correct. Clear slots in debug on release. If the object knows it lives in a pool, an existing field (lifetime remaining) can mean "free." If callers keep a reference, the pool must not silently reuse that slot under them. Under a collector, clear references inside a released object or the collector keeps the rest of the graph alive.

### Spatial partition

**Intent.** Find objects near a position by storing them in a structure ordered by position.

An unordered list makes every pair an explicit test, quadratic in the number of units. A flat grid of fixed cells, each a list of objects whose centers fall in that cell, lets a query look only at the object's cell and, if the radius is larger than zero, at neighboring cells. Test each unordered pair once. Movement that stays in the cell only updates coordinates. Crossing a boundary unlinks and relinks.

A hierarchical structure splits a region only while it holds too many objects. Exact distance tests still run on the candidates. The partition is the broad filter. It does not decide the hit.

Use it when location queries or pair tests are actually hot and the set is large. Skip it when the set is small, objects move so often that the bookkeeping dominates, you are tighter on memory than on time, or the radius spans so many cells that you have rebuilt the full pair test.

- **Flat grid** when you want simple code, a memory cost fixed up front, and fast moves. Accept imbalance and empty cells.
- **Quadtree or octree** when density varies and you still add and move objects one at a time. One insertion triggers at most one split.
- **A batch-built tree** when the set is static or rebuilt as a batch and you want each region to hold a similar count. Better for level geometry than for units that stream in.

A second flat list of live objects makes the "update everyone" pass cheap when the grid is full of empty cells, at the cost of keeping the two in sync. "Move" in a tree is a remove plus an add.

## How the patterns compose

- The game loop calls the update walk. The display flip is the graphics form of double buffer. Double-buffer actor state only when same-frame reads must not see same-frame writes.
- Components are the usual home of the per-frame update once domains must be mixed. Data locality then stores each domain in its own packed array.
- State, type object, and bytecode are three places behavior can live: what the actor is doing right now, what kind it is, and a script the kind runs. A sandbox is the code form of "many variants, one set of powers."
- Command, observer, and event queue are three distances from a direct call: a value you can store, a synchronous fan-out, and a deferred handoff. Do not build all three for one signal.
- Flyweight, object pool, and spatial partition solve three different costs: repeated heavy data, allocation churn, and "who is near me." A pool that is also packed is a locality win. A breed that allocates from a pool is a type object plus a pool.
- Singleton is the pattern to refuse by default. A locator is the narrower escape for ambient facilities. Passing the dependency is the default.

## Cautions

- Do not add a second unbounded loop inside an engine callback, and do not block inside the callback the host uses to keep the process alive.
- Do not pass raw frame delta into physics or lockstep play and call it done. Accumulate time, run fixed ticks, cap the catch-up, and treat interpolated draw positions as guesses.
- Do not buffer actor state "to be safe." It costs memory, delays every reaction by a frame, and throws away ordered updates.
- Do not treat tick order as irrelevant without that snapshot. Do not spawn or destroy in the middle of the walk without a rule for who runs and who is skipped.
- Do not make one subclass per spell, item, or breed that reaches into audio and rendering. If they differ by data, use a breed record. If they differ by a short script, a bytecode machine is justified only with an authoring front end and a way to see what the script did.
- Do not grow a component, a locator, or an event bus because the pattern is available. Add the indirection when a concrete coupling or a measured cost is already in the way.
