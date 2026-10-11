# Errors and special cases

An exception here means any uncommon condition that diverts normal control flow, including a special return value. Throwing is cheap. Handling is the cost. Handlers are verbose, rarely executed, and hard to test. An exception is also interface surface, because it can climb through callers and widen their interfaces.

Fewer handling sites is the goal.

## Choose a move

Apply the first move that fits.

### 1. Define it out of existence

Change the contract so the former failure is ordinary success, when callers do not need to know it happened.

- "Ensure this name is absent" succeeds when it was already absent.
- "Delete the thing that exists" fails if it is already gone.

Do the same for ordinary special cases. Design the normal algorithm so the empty or absent situation falls out with no extra branch. A user-visible idea such as "nothing selected" need not be a distinct internal state. A zero-width range can mean "nothing highlighted," so copy and delete need no empty branch.

### 2. Mask

Handle the condition inside a widely used module so callers never see it. Retry, wait, or repair there. This deepens the module. Use it when callers have no useful alternative. Do not mask a failure the caller must act on, such as lost work or a dead peer.

### 3. Aggregate

One handler covers many conditions. On a request loop, a single failure type can abort the current request, clean up, and continue. Keep that distinct from faults that kill the process. Put the human-readable cause at the origin and the response format in the one handler. Do not promote a frequent, cheap fault into an expensive recovery.

### 4. Crash

Crash when the fault is rare, recovery is not real, and continuing would spread checks everywhere. Exhausted memory is the pattern: every allocation would need a check, a missed check hides the real fault, and there is usually nothing sensible to free. Centralize detection and abort with a clear diagnostic. The same often applies to hard I/O errors on an already opened file and to broken invariants.

Do not crash when surviving that fault is the product. Do not crash for frequent, recoverable faults.

## Decision test

- Can a slightly different contract make this case success?
- If the fact must remain, where can one handler cover the most occurrences: low and shared (mask) or high and shared (aggregate)?
- Can the caller do anything useful? If not, is a crash with a clear diagnostic more honest than a check at every site? If yes, is the failure visible on the interface?

## Do not

- Report every suspicious condition.
- Put the same handler at every call site.
- Invent a distinct empty state the main algorithm could treat as a normal zero.
- Mask a fact the caller must act on.
- Check every allocation for a condition the process cannot survive.
