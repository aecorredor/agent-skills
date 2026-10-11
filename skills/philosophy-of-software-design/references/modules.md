# Modules

Use this when the work is the shape of a module, an interface, or a split.

## Interface

An interface is every fact a caller in another module must know.

- Formal: parameter and return types, observable failures, public state.
- Informal: what the operation does, ordering, units, ownership, durability, and "do not mutate this."

Shrink the set of facts that are important. If a caller must know it, put it on the interface. If they must not, keep it inside.

## Depth

Ask:

- Is the interface much simpler than the implementation?
- Can the common caller succeed without learning rare options or assembling several objects?
- If the inside changes and the interface does not, do callers stay valid?

Small modules tend to be shallow. Some shallow modules are unavoidable when there is little to hide. Do not split by a line-count rule. A long method with a simple signature and a straight reading order can be deep.

## Information hiding

A design decision (representation, algorithm, constant, protocol, an assumption such as "most records are small") should live in one module.

Private fields are only a mechanism. Accessors that force callers to understand the representation are still the interface.

Leakage means the same decision is encoded in more than one module. Back-door leakage, where neither module advertises the shared knowledge, is worse than leakage through an interface. Repair by merging the modules that share the knowledge, or by extracting it into one module whose interface truly abstracts it. If the new interface still exposes the knowledge, the leak only moved.

Temporal decomposition groups code by runtime order (read, then transform, then write) even though the same knowledge is required at several times. Order belongs in control flow. Carve modules by knowledge.

Inside a class, a few methods should own each piece of state.

## Somewhat general-purpose

Match today's functionality. Keep the interface free of one caller's vocabulary. Stop when the result is still easy and efficient for the caller you have.

Ask:

- What is the smallest interface that still covers every current use?
- Will each method be used in more than one situation?
- Can today's caller use it without extra loops or awkward performance?
- Did the method count drop without an exploding argument list?

A method with a single call site is a warning. A maximally general interface that is clumsy for the real caller has gone too far. Hiding a detail the caller must know is a false abstraction. Make that detail obvious at the call site.

## Layers

Adjacent layers should be different abstractions. If deleting a layer loses only a forwarding hop, it is not a layer.

- A pass-through method mostly forwards arguments to a similar signature. It adds surface and shows that the work lives somewhere else.
- Identical signatures are appropriate for dispatch and for sibling implementations of one interface. They are not appropriate for a wrapper whose methods call the same methods on an inner object.
- A decorator that adds a little behavior by forwarding the whole API is usually shallow. Put nearly universal behavior in the base, push a true specialization upward, or merge wrappers.
- A pass-through variable is threaded through methods that never use it. Prefer one context object per instance, held by major objects and passed at construction. Keep it small and preferably immutable so it does not become a hidden junk drawer.

The interface to a capability belongs with the code that implements it. The representation should differ from the interface. That difference is the module's contribution.

Every new interface, argument, function, or type costs attention. It pays only if it removes more complexity than it adds.

## Pull complexity downward

There are more callers than authors. A simple interface beats a simple implementation.

Pull complexity down only when all three hold:

1. The complexity is inherent to this module.
2. Swallowing it simplifies many callers.
3. The interface gets smaller.

Configuration is often an incomplete solution. If a higher layer cannot choose a better value than the module can compute, do not export the knob. If you expose one, ship a computed default and make override rare.

Do not pull down caller-specific policy that is unrelated to the module's core. Do not pull the whole application into one module.

## Together or apart

Bring code together when:

- the pieces share knowledge, or each is hard to understand without the other
- a combined interface can do the right thing automatically
- use is bidirectional
- joining deletes an interface or stops a fact from living in two places

One-way use is not enough. A general tool used by a special module stays separate.

Separate when:

- a general mechanism is mixed with one special use of that mechanism (the special use belongs higher up)
- a subtask is understandable alone and the parent does not depend on its body
- the original operation did unrelated jobs and most callers need only one of the new interfaces

If every caller must invoke both new methods and shuttle state between them, the split made shallow methods.

Extract a repeated snippet only when it is long and the new signature stays simple. A line or two, or a snippet that closes over many locals, gains little. Sometimes the better move is to run the snippet once.

Special-purpose code for one mechanism may live inside the general module of another mechanism when that is where the knowledge lives.
