---
name: philosophy-of-software-design
description: >-
  Reviews and shapes modules, APIs, and refactors so complexity stays low:
  deep modules, information hiding, error contracts, comments, and names.
  Use when designing or reviewing modules, interfaces, abstractions, error
  handling, comments, naming, or refactors, or when the user mentions
  complexity, shallow modules, information leakage, pass-through methods,
  special cases, or designing an API twice.
license: MIT
metadata:
  version: "1.0.0"
---

# Philosophy of software design

Apply this while designing or changing a module, interface, or refactor. The goal is less complexity for the next reader, not more machinery. Ideas here are informed by John Ousterhout's *A Philosophy of Software Design*. This skill is an original procedure, not a reproduction of that book.

If a rule below does not reduce complexity in the code in front of you, do not apply it.

## What complexity means

Complexity is whatever makes a specific task hard to understand or hard to change. Weight it by how often people work in that code.

Three symptoms:

- **Change amplification.** One decision forces edits in many places.
- **Cognitive load.** The task requires learning a lot. Line count is a poor proxy.
- **Unknown unknowns.** You cannot tell what to change until a bug appears. This is the worst symptom.

Two causes: **dependencies** (code that cannot be understood or changed alone) and **obscurity** (important facts that are not apparent). Prefer fewer dependencies, and make the ones that remain obvious and mechanically checked.

## Procedure

Copy this list and work through it. Skip a step only when it does not apply to the change.

```text
Design check:
- [ ] Name the knowledge this module owns, and the facts callers must still know
- [ ] Check depth: the interface is much simpler than the implementation
- [ ] Check hiding: each design decision lives in one place
- [ ] Check layers: adjacent layers are different abstractions
- [ ] Decide together or apart using the tests in references/modules.md
- [ ] Pull inherent complexity down only when callers get simpler
- [ ] Define errors and special cases out of existence where callers do not need the fact
- [ ] If the shape is uncertain, sketch a second design before committing
- [ ] Comments and names carry what the code cannot show
- [ ] On an edit, leave the design at least as deep as you found it
- [ ] Do not add interface complexity for speed without a measurement
```

Read the matching reference when a step is the actual work, not for every task:

- Module shape, hiding, layers, splitting: [references/modules.md](references/modules.md)
- Errors and special cases: [references/errors.md](references/errors.md)
- Comments and names: [references/comments-and-names.md](references/comments-and-names.md)
- Edits, consistency, tests, and performance: [references/change-and-performance.md](references/change-and-performance.md)
- Original before/after sketches: [references/examples.md](references/examples.md)

## Investment

Working code is required. It is not the goal. Spend a small, continuous slice of the work on the design: try a second shape, fix a design mistake when it becomes visible, write the comment that captures the contract. Do not defer that to a cleanup that will not happen. Do not design the whole system up front. Build a slice, learn, and reshape while the change is still cheap.

## Depth, in one test

Benefit is functionality. Cost is interface complexity imposed on everyone else. A deep module hides a lot of behavior behind a small interface. A shallow module's interface is almost as hard as its implementation.

The interface is everything a caller in another module must know: types, and also ordering, units, ownership, and failure behavior. Omitting a fact the caller needs is a false abstraction. Putting a rare knob on the common path is overexposure.

The common path should be the small part of the interface. Rare options get a separate entry point.

## Red flags

Stop and redesign when you see these:

- Shallow module, pass-through method, or a decorator that mostly forwards
- Information leakage, or modules split by execution order rather than by knowledge
- Overexposure of rare features on the common call
- Repeated nontrivial code, or special-case policy mixed into a general mechanism
- Conjoined methods that only make sense when read together
- A comment that restates the next line, or an interface comment that describes the implementation
- A vague name, a name you cannot make precise, or a comment that must be long to be complete
- Code whose first quick reading is wrong

## Designing twice

When the module, the feature split, or the error contract is uncertain, sketch two genuinely different options. Compare them with these questions, then pick or combine:

- Which is easier for the code above it?
- Which interface is simpler, and which is more general without becoming vague?
- Which allows a simpler or faster implementation?
- If both are awkward for the same reason, design a third that removes that reason.

Spend a short pass relative to the implementation. Ship one design. Do not exhaustively design a tiny change.

## Output when reviewing

Lead with the design problem, then the change that would deepen the module. Use the red-flag names above so the comment is specific. Do not ask for more types, more wrappers, or shorter methods unless that split makes each piece a simpler abstraction and most callers need only one of them.
