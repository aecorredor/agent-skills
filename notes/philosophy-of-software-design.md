# Study notes: A Philosophy of Software Design

Working synthesis of John Ousterhout, *A Philosophy of Software Design* (first edition, 2018, Yaknyam Press). These notes are original operational notes for an agent skill. They are not a reproduction of the book.

The book's own test for every rule: if applying it does not reduce complexity, drop it. Extreme application of a good idea produces a bad design.

## What design is for

The limiting skill is problem decomposition: splitting a problem so each piece can be understood and changed with little knowledge of the others. Software can be reshaped after it exists, so design is continuous. Incremental slices beat a frozen early architecture that later gets patched around.

Complexity still grows with size and with the number of people. Tools help. They do not replace the design work.

## Complexity

Complexity is any structural property that makes the system hard to understand or hard to modify **while doing a particular task**. It is a cost paid by a developer at that moment.

- A large system can be simple if common work is easy.
- A small system can be complex.
- Weight by attention. A nasty corner nobody touches barely counts. The same nastiness on a hot path dominates.
- If readers find it complex, it is complex, even when the author finds it obvious.

### Three symptoms

1. **Change amplification.** One decision forces edits in many places.
2. **Cognitive load.** Completing the task requires learning a lot: many methods, globals, inconsistencies, cross-module rules. Line count is a poor proxy. A longer local implementation can be simpler than a short one whose meaning is scattered.
3. **Unknown unknowns.** You cannot tell which code must change, or even that a hazard exists, until a bug appears. This is the worst symptom. Amplification is tolerable when the edit list is obvious. Load is tolerable when you know what to read.

The opposite is an **obvious** system: a quick guess about what to change is usually right.

### Two causes

- **Dependencies.** A piece of code cannot be understood or changed alone. Dependencies are unavoidable; every API creates some. Prefer fewer of them, and make the ones that remain simple, named, and mechanically checked.
- **Obscurity.** Important facts are not apparent: vague names, missing units, one name used for two meanings, a dependency that does not show up at the declaration, inconsistency. A design that needs a lot of documentation is usually the wrong shape. Simplifying the design removes more obscurity than writing more docs.

Dependencies drive amplification and load. Obscurity drives unknown unknowns and also load. Complexity arrives as hundreds of small cuts, not one disaster. Each "tiny" compromise feels cheap. Together they make every future change expensive, and fixing one of them later barely helps. Treat new dependencies and obscurity as defects to remove now.

## Strategic versus tactical work

Tactical work optimizes the current feature or fix. Each task adds a small kludge. The next task patches around the last one. Cleanup is deferred because one fix will not move the needle. A prolific tactical programmer can look like a hero and leave a mess that makes careful engineers look slow.

Strategic work treats a good design that happens to work as the main goal, because most future code extends what already exists. Working code is required. It is not the goal.

Budget about **10–20%** of development time for continual small investments, not a big upfront design.

- Proactive: try more than one shape for a new module, imagine a few likely changes, write the comments that capture non-obvious contracts.
- Reactive: when a design mistake becomes visible, fix it instead of routing around it.

Early work runs slower. Within a few months the saved rework covers the ongoing investment. A degraded codebase commonly slows people by at least that much, so the early tactical savings are given back.

**Startups.** Shipping pressure makes the 10–20% look unaffordable, with a plan to hire people to clean up after success. A tangled codebase is nearly impossible to repair, the cost lasts for the life of the product, and the payoff shows up fast enough that the shortcut may not even move the first release. Strong engineers want a codebase they can respect. Once cleanups are postponed until "after the crunch," the next crunch arrives and the postponement becomes the culture. Investment is today's work.

Designing the whole system up front does not work. The good shape appears in pieces as you learn.

## Modules and interfaces

Modular design minimizes cross-module knowledge by splitting interface from implementation.

**An interface is everything a caller in another module must know.**

- Formal, often language-checked: parameter and return types, thrown failures, names and types of public state.
- Informal, usually larger, and only describable in comments: what the operation actually does, ordering constraints, units, ownership, performance or durability rules, "do not mutate this returned structure."

If a caller must know it, it is interface. Omitting an important fact is a false abstraction. Including an unimportant fact is needless load. The design goal is to shrink the set of facts that are important.

**Depth.** Benefit is functionality. Cost is interface complexity imposed on everyone else. A deep module hides a lot of behavior behind a small interface, so much of it can change without touching callers. A shallow module's interface is almost as hard as its implementation. The new name adds something to learn and hides little. Small modules tend to be shallow.

The common path should be the small part of the interface. Rare knobs stay off to the side so most callers never learn them. Effective complexity is the complexity of the features people actually use.

Some shallow modules are unavoidable when there is little to hide. Depth is not "one class for the program."

**Red flags:** interface almost as complex as the body; documentation longer than the code; more keystrokes to call than to do the work inline; splitting by a line-count rule; common behavior that callers must opt into or they silently get a bad default.

## Information hiding

Depth comes from burying a design decision inside one module so others do not depend on it: representation, algorithm, a size constant, a protocol, an assumption such as "most records are small."

Declaring members private is only a mechanism. If public accessors still force callers to understand the representation, the decision has leaked. Partial hiding still helps: rare facts behind a separate entry point, invisible on the common path.

**Leakage** means the same decision is encoded in more than one module. Leakage through an interface is at least visible. Back-door leakage (two modules that both know a format, while neither advertises it) is worse. Repair by merging the modules that share the knowledge, or by extracting that knowledge into one module whose interface truly abstracts it. If the new interface still exposes the knowledge, the leak only moved.

**Temporal decomposition** structures modules by runtime order (read, then transform, then write) even though the same knowledge is required at several times. Order belongs in control flow. Modules should be carved by knowledge. If callers truly need a fact, expose it. Prefer computing the setting inside when you can. Do not hide a requirement and call the smaller interface a win.

**Overexposure:** the common call forces you to learn a rare feature. Defaults should make the common case happen with no request. The best behavior is behavior callers do not have to know exists.

Inside a class, minimize the places that read or write each field. A few methods should own each piece of state.

## Somewhat general-purpose

The useful bias is somewhat general-purpose: behavior covers today's needs, while the interface is not specialized to today's caller. That usually yields fewer, deeper methods and a cleaner split of knowledge, even if nobody reuses the module.

Questions:

- What is the smallest interface that still covers every current use?
- Will each method be used in more than one situation?
- Can today's caller use it without a pile of extra loops, glue, or awkward performance?
- Did we reduce method count without exploding the argument list?

A method that exists for one call site is a warning. Caller vocabulary (a particular gesture, screen, or policy) inside a lower-level module is a warning.

**Too far:** a maximally general interface that is clumsy or slow for the real caller. Generality that ignores a known, central use is not deeper. Hiding a detail the caller must know is a false abstraction; make that detail obvious at the call site.

## Different layer, different abstraction

As a task moves through layers, the abstraction should change at each step. Adjacent layers with the same abstraction show up as pass-through methods, decorator stacks, or pass-through variables. Those add surface without function.

- A **pass-through method** mostly forwards arguments to a method with a similar signature. It adds interface surface and a signature dependency, and it shows that responsibility is split away from the code that does the work.
- Identical signatures are appropriate for dispatch and for sibling implementations of one interface that do not call each other. One interface is learned once.
- A **decorator** that wraps an object with the same API and a small extra behavior is often a shallow class full of forwarding. Prefer adding nearly universal behavior to the base, folding a specialized tweak into the caller, merging wrappers, or building a standalone piece that does not re-export the whole API.
- A **pass-through variable** is threaded through intermediate methods that never use it. A new cross-cutting fact then edits every signature on the path. The usual compromise is one context object per instance, held by major objects and passed at construction, so most methods do not list it. Keep it small and preferably immutable. It still has global-like hazards: unclear ownership, a grab-bag of hidden dependencies, and thread-safety trouble if its fields change.
- The interface to a capability belongs with the code that implements it. Internal representation should differ from the interface. That difference is the module's contribution.

Every new interface, argument, function, or type costs attention. It pays only if it removes more complexity than it adds.

Decision test: what is this layer responsible for that the layer below is not? If deleting it loses only a forwarding hop, it is not a layer.

## Pull complexity downward

There are more callers than authors, so a simple interface beats a simple implementation. When complexity belongs to a module's job, the module should absorb it. Throwing, or exporting a knob, multiplies the problem by the number of callers and operators.

Configuration is often an incomplete solution. Operators frequently cannot pick a better value than the module could compute, and static knobs go stale. Ask whether a higher layer can actually choose better. If you expose a knob, ship a computed default and make override rare.

Pull complexity down only when all three hold:

1. The complexity is inherent to this module.
2. Swallowing it simplifies many callers.
3. The interface gets smaller.

Pulling down caller-specific policy that is unrelated to the module's core leaks information and does not simplify callers. Pulling the whole application into one module minimizes nothing. The target is total system complexity.

## Better together or better apart

Choose the split that lowers whole-system complexity: best hiding, fewest dependencies, deepest interfaces.

**Bring together** when:

- the pieces share knowledge
- each is hard to understand without the other
- a combined interface can do the right thing automatically
- use is bidirectional
- both pieces sit under one simple category
- joining deletes an intermediate structure or stops a fact from living in two places

One-way use is not enough. A cache may need a map, and maps are still their own module.

**Separate** when:

- a general mechanism is mixed with one special use of that mechanism (specialization belongs higher up)
- a subtask is understandable alone and the parent does not depend on that body
- the original method did unrelated jobs and most callers need only one of the new interfaces

Special-purpose code for mechanism A may still sit inside the general module of mechanism B when that is where the knowledge lives.

**Methods.** Length is a weak reason to split. A long method with a simple signature and a straight reading order can be deep. Independent blocks can be read in place. Interacting blocks should stay together.

Extract a method when the snippet is long and the new signature stays simple. A line or two, or a snippet that closes over many locals, gains little. Sometimes the better move is to run the snippet once (a single cleanup exit).

Split into two caller-facing methods only when most callers need only one. If every caller must invoke both and shuttle state, the split made shallow methods. Joining is how you turn two shallow methods into one deep one.

**Red flags:** repetition of nontrivial code; a special-general mixture; conjoined methods (separated pieces that only make sense when read together); a tiny helper with one call site and more comment than code.

## Errors and special cases

An exception here means any uncommon condition that diverts normal control flow, including a special return value. Handling is harder than the normal path: either push through (and often invent a second anomaly, such as duplicates from a retry) or abort and restore consistency. The handling code is verbose, rarely executed, and hard to test. Code that has not run tends not to work. Throwing is cheap. Handling is the cost. An exception is a heavy interface element because it can climb through callers and widen their interfaces too.

Fewer handling sites is the goal. Four moves, in this order of preference when they fit:

1. **Define the error out of existence.** Change the contract so the former failure is ordinary success. "Make sure this name is gone" succeeds when it was already gone. "Delete the thing that exists" fails. The same move deletes special cases: design the normal algorithm so the empty or absent situation falls out with no extra branch. A user-visible idea (nothing selected) need not be a distinct internal state. Represent "nothing highlighted" as a zero-width range so copy and delete need no empty branch.
2. **Mask.** Handle the condition low, inside a widely used module, so callers never see it. Retry, wait, or repair there. This deepens the module. It fits when callers have no useful alternative.
3. **Aggregate.** One handler covers many conditions. On a request loop, a single failure type can abort the current request, clean up, and continue. Keep that distinct from faults that kill the process. Put the human-readable cause at the throw site and the response format in the one handler. Do not promote frequent, cheap faults into an expensive recovery.
4. **Crash** when the fault is rare, recovery is not real, and continuing would spread checks everywhere. Exhausted memory is the pattern: every allocation would need a check, a missed check hides the real fault, and there is usually nothing sensible to free. Centralize detection and abort with a clear diagnostic. The same often applies to hard I/O errors on an already opened file and to internal invariant breaks. Crashing is the wrong call when surviving that fault is the product.

Mask or define away a condition only when callers do not need the fact. If a lost message or a dead peer changes what the caller must do, the failure belongs on the interface.

## Design it twice

The first structure for a module, feature set, or split is unlikely to be the best. Sketch at least two genuinely different options, including one you expect to dislike, then choose or combine.

Rough out only the important operations. Then ask:

- Which is easier for the code above it? That outweighs other interface criteria.
- Which interface is simpler?
- Which is more general without becoming vague?
- Which allows a faster implementation?
- For the inside, which is simpler, and which is faster?
- Can the winner borrow one idea from the loser?
- If both are awkward for the same reason, design a third that removes that reason.

Spend a short design pass relative to the implementation. Do not exhaustively design a one-hour change. Do not treat "we considered two designs" as a substitute for picking one.

## Comments

Comments hold design knowledge the language cannot express. They complete an abstraction. A clean design that is undocumented loses most of its value for the next reader.

A comment earns its place only when a first-time reader could not write it from the adjacent code. Same-level restatement is noise. Useful comments sit either below the code (more exact) or above it (the idea).

### The four excuses

1. **"Good code documents itself."** Names and signatures carry only the formal part. They omit the meaning of results, boundary behavior, when a call is valid, and why the design exists. Telling readers to learn a method by reading its body destroys the abstraction and pushes you to split work into many shallow methods so each body stays short.
2. **"There is no time."** Something will always outrank comments, so a postponed comment is a missing comment. Interface comments written while designing pay off immediately because they are the design check.
3. **"Comments rot."** They do, when they are copied and stored far from the code. Keep one copy next to the code, and look at the diff before you commit.
4. **"Every comment I have seen was useless."** Often true, and it is a writing failure. Comments that restate the next line teach people to ignore comments.

Comments do not, by themselves, stop a change from touching many places. That is a structure problem.

### What to write

**Interface comments** are the abstraction.

- A type comment says what the type is for, what one instance represents, and which limits a caller must respect.
- A method comment starts with caller-visible behavior, then states each argument and the result with constraints, every side effect, every exception the caller can observe, and any precondition. The reader should call the method without opening the body. If the interface comment has to explain the implementation, the module is shallow.

**Implementation comments** sit inside the body. Most short methods need none. For a long method, put a high-level note before each major block and before a loop whose iteration is not obvious: what this block accomplishes, and why this path exists. Say what and why. The code already says how.

**Precision versus intuition.**

- Lower-level comments add precision the declaration lacks: units, whether an end index is included, what null means, who frees a resource, invariants. Write them as nouns.
- Higher-level comments give a frame: the one or two ideas that make the block intelligible.

Adopt one comment convention so people know what to write and readers know where to look. Cross-module decisions need a single obvious home, with a short pointer at each other site. Duplicating the essay at every site rots.

### Write comments first

Comments written after the code stabilizes are often never written, and when they are, they repeat the code because the design reasoning has faded.

1. Write the type comment.
2. Write comments and signatures for the important public methods. Leave bodies empty.
3. Revise until the structure is easy to say.
4. Declare the important fields with their comments.
5. Fill in bodies, adding implementation comments only where a block needs a frame.
6. For every method or field discovered later, write its comment before or with the declaration.

Decision test: can you describe how to use it, completely, in a short comment? A long comment that is still incomplete means the interface is too complex or the comment is cryptic. If the comment must mention the major implementation pieces, the method is shallow. Compare comment and body: a deep method has a small interface description and a lot of work behind it.

Revise the comment when coding reveals a better abstraction. Do not freeze it.

### Maintaining comments

- Put each comment at the narrowest place that covers everything it describes. The farther a comment sits from the code, the more abstract it should be.
- If a future developer needs a fact, it belongs in the code. The commit log is a poor place to hide it.
- Record each design decision once. A broken pointer is obvious. A stale duplicate is not.
- Do not document one module's decisions inside another.
- Before commit, read the whole diff for comment drift.

## Names

A name is a small abstraction. In isolation it should make a reader picture what the thing is and what it is not. One ambiguous name can cause a bug because people trust the name and stop checking the value.

- **Image.** Pick the few words that highlight what matters. Names longer than two or three words get unwieldy.
- **Precision.** Vague names (`count`, `status`, a boolean that is not a predicate) invite the wrong assumption. Over-specific names lie (an argument called `selection` that accepts any range). Generic `i` and `j` are fine when the whole loop is visible. `result` is acceptable on a real return path when the method comment defines it.
- **Consistency.** For a recurring concept, one name everywhere, that name nowhere else, and a purpose narrow enough that every use behaves the same way. When you need two, add a prefix (`src` / `dst`) rather than inventing a second root.

If you cannot find a precise, intuitive name, the thing may be doing two jobs or have no crisp purpose. Split it or redefine it. A short name can be clear when its declaration is nearby and the project uses that short name for exactly one meaning. Distance from declaration to use calls for a longer name. Readers decide. If review calls the names cryptic, lengthen them.

## Modifying existing code

A system's design is mostly the history of its changes. Each "smallest patch" adds a special case. After the change, the code should look as if this requirement had been part of the original design. If you are not improving the design a little, you are probably making it worse.

If a full reshape takes months and a hack takes hours, look for a design that is nearly as clean in a few days. If you must ship the hack, schedule the cleanup. Do not leave the cleanup as an intention.

Do not redesign on every bugfix. Do not gold-plate when a smaller improvement was available and a deadline is real.

## Consistency

If similar things work the same way, and different things look different, a reader can reuse a conclusion. Inconsistent systems make false assumptions.

Keep consistent: names for the same concept; style; one interface with several implementations; a known design pattern when the problem actually matches it; invariants such as "every stored line ends with a newline."

If you want a new convention, you need real new information, and the new way must be enough better to update every old use so the old way disappears. Otherwise follow the existing convention. Do not force unlike things into one pattern so that "it looks like X" no longer means it is X.

Write global rules where newcomers will see them. Enforce mechanical rules with a check that blocks the commit. Review the rest.

## Obvious code

Obvious code is code whose first quick guess is right. Obviousness is judged by a reader, especially in review, not by the author.

What helps: precise names, consistency, whitespace that shows where one parameter or block ends, a comment aimed at the exact confusion.

What obscures:

- Event-driven flow, where the handler is not called at the use site. Say, on the handler, when it runs and on which thread.
- Generic pair-like containers, so the caller sees key and value instead of the domain. Define a small named type.
- A declaration whose static type is a broad interface while the allocated object's performance or safety differs and matters.
- Control flow that violates a platform expectation, such as a main routine that returns while other threads keep the process alive.

Do not ban events, generics, or interfaces. Do not comment code a reader already reads correctly.

## How to judge common practices

Judge a practice by whether it reduces complexity in a large system.

- **Interface inheritance** (shared signatures, separate implementations) lets one set of caller knowledge cover many mechanisms, provided the interface keeps only what all implementations share. **Implementation inheritance** can remove duplicated methods and couples parent and children through shared fields. Prefer a small helper that each class uses. If you still inherit implementation, keep parent state behind parent methods. Classes and privacy do not fix a shallow class.
- **Agile increments** match the fact that you cannot see the whole design at the start. The risk is a feature treadmill: special-case code now, a general mechanism later, which often means never. Delay an abstraction until a feature needs it. Once it is needed, design it as a whole and make it somewhat general. The increment is the abstraction.
- **Unit tests** make structural change safe enough to attempt. Without them, people minimize the diff and the design decays. Tests do not choose the design.
- **Test-driven development** keeps attention on the next behavior and has no natural moment to design the abstraction, so the result tends to be tactical. The exception is a bug: write a failing test that exhibits it, then fix it.
- **Design patterns** that fit are usually a better bet than a from-scratch mechanism, and readers recognize them. A pattern that does not fit is extra machinery. More patterns are not better.
- **Getters and setters** can later enforce a constraint without changing call shape, but exposing the field, even through a one-line accessor, publishes implementation and adds shallow methods. Prefer operations that hide the data.

## Performance

Clean design and speed usually move together. Simpler code does less redundant work. Deep modules do more per call and cross fewer layers.

1. Know which operations are expensive from measurement, not a hunch: network round trips, durable I/O, allocation and collection, cache misses.
2. Do not tune every statement, and do not ignore cost. A system full of small inefficiencies has no single fix later.
3. When speed needs complexity, hidden implementation complexity can be worth it. Interface complexity is a high price. A fundamental change (a cache, a better algorithm, bypassing a known-slow layer) beats micro-tuning.
4. If you must reshape a hot path, ignore the current structure and write the shortest common-case sequence you can imagine: one place, only the data that path needs, special cases ignored. Call that the ideal.
5. Redesign so the real code stays close to that ideal and still has a clean structure. One test at the start should reject every uncommon case. Handle special cases off to the side, written for clarity.

If a change does not move the number, remove the complexity unless the change also simplified the code. Start with the simple design unless you already know this path dominates.

## Principles

- Complexity arrives through small decisions, so small decisions count.
- Code that merely works is below the bar.
- Keep spending small amounts of effort on the design.
- Modules should be deep: a small interface in front of a lot of work.
- Shape the interface so the common case is the easy case.
- A simple interface matters more than a simple implementation.
- General-purpose modules go deeper than one-off ones.
- Keep general-purpose code separate from special-purpose code.
- Adjacent layers should offer different abstractions.
- Push complexity downward, into the implementation.
- Define errors and special cases out of existence when callers do not need the fact.
- Design each important decision at least twice.
- Comments should add what the code does not already show.
- Design for the reader, not for the speed of writing.
- Grow the system by abstractions, not by a pile of features.

## Red flags

- **Shallow module.** The interface is almost as complex as the implementation.
- **Information leakage.** One design decision shows up in several modules.
- **Temporal decomposition.** The structure follows execution order instead of hiding information.
- **Overexposure.** Callers must learn rare features in order to use common ones.
- **Pass-through method.** A method mostly forwards arguments to a similar method.
- **Repetition.** A nontrivial stretch of code appears again and again.
- **Special-general mixture.** Special-purpose code is mixed into the general mechanism.
- **Conjoined methods.** You cannot understand one method's implementation without the other's.
- **Comment repeats the code.**
- **Interface documentation describes implementation** the caller does not need.
- **Vague name.** The name could mean too many things.
- **Hard-to-choose name.** No short name creates a clear picture, which suggests a muddy concept.
- **Hard to describe.** A complete comment has to be long.
- **Nonobvious code.** A quick reading does not reveal the behavior.
