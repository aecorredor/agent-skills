# Sequencing

Time, actor ticks, and who is allowed to see a half-finished change. Apply this in whatever loop the engine already has.

## Game loop

Keep game time moving without waiting on input, and at a rate that does not follow the CPU. One pass: non-blocking input, advance simulation, draw. An engine owns the loop and calls your code. A library is what you call from a loop you own. Do not write a second loop inside an engine that already calls you, and do not block inside the host callback.

Pick a variant:

1. **Host callback.** Use it when the platform will not let you own the loop. Apply the timing rules below inside the callback.
2. **Fixed step, no clock.** Game speed tracks hardware. Do not ship this if the same build runs on more than one machine.
3. **Fixed step with sleep.** The game cannot run too fast. If the frame runs long, the rules slow down, because simulation and drawing are still tied together. Sleeping spare time is the right power choice on a battery device.
4. **Variable step.** Scale motion by elapsed real time. Usually the wrong default. Different machines take different numbers of steps, so results diverge. Damping tuned for one slice blows up on another. The clock is outside the program, so the simulation is not deterministic.
5. **Fixed simulation step, draw once.** Accumulate real time. While it covers one fixed slice, tick once and subtract the slice. Then draw. Slow machines drop draws instead of changing the rules. This is the option for physics or lockstep play. Tune the slice longer than one tick costs on the slowest target. Cap the number of catch-up ticks so a slow frame cannot spin forever. The slice is not the display rate.

After the catch-up loop, leftover time divided by the slice is a fraction from 0 up to but not including 1. Draw moving things that far along their current velocity so they do not stick to the last tick. That position is a guess. Do not treat it as the authoritative state.

Turn-based rules still need the loop, because animation and audio do not wait.

## Update method

Simulate many independent objects by asking each to run one frame of its own behavior. The world holds the collection. Each object stores whatever it needs to resume. Inside the frame they run one after another. They look simultaneous.

Do not tick rule-bearing pieces every frame when the game is turn-based. Sequential rules keep one valid world. Simultaneous rule updates create conflicts.

Keep these hazards explicit:

- **Order.** An object sees neighbors that have not ticked yet in their previous state, and neighbors that already ticked in their new state. If you need one valid world after another, keep that serial order. Snapshot only when same-frame reads must not see same-frame writes.
- **Birth.** Appending during the walk ticks the newcomer on the frame it was born. Cache the count if new objects should wait.
- **Death.** Removing an earlier slot skips the next object. Mark dead and compact after the walk, or walk backward. If several threads tick the list, defer structural edits.
- **Slice.** A large variable step overshoots patrol endpoints. Prefer the fixed slice from the loop.
- **Idle objects.** A list full of disabled objects burns time and cache. Split live versus inactive when many are idle, and update that split on real create and destroy. Keep one list when few are idle.
- **Where the method lives.** On the object when there are few kinds. On a component when domains must mix. On a delegated mode or breed when behavior should change without a new subclass. A generator or coroutine can pause mid-behavior. An operating-system thread per object is too heavy.

Straight-line behavior becomes explicit resume state. A state machine is the usual place to keep "where I left off." See [behavior.md](behavior.md) only if you are choosing that shape. This file is the one to finish for tick lifetime.

## Double buffer

Make a sequence of edits look instantaneous to every reader. Two different jobs:

- A display scans a frame while it is still being drawn.
- Simulation reads the same fields it writes in the same pass, so who reacts depends on list order.

Readers always use the current copy. Writers only touch the next copy. One atomic swap publishes the next copy.

- **Repoint** by exchanging references. Cheap, and independent of size. Callers must not keep pointers into the buffer. Alternating copies means leftover contents are from two frames ago, not the previous frame. That matters if you blend with the previous image. Unavailable if the visible buffer must sit at a fixed address.
- **Copy** the finished next state over the current state. The published data is one frame old. The copy can cost more than the edit. Fine for a flag. Wrong for a full frame.

For interactions, tick everyone against the current snapshot, then swap. A reaction becomes visible on the next frame, in the same way no matter the order. That also delays every reaction by a frame and throws away the ordered updates you want for conflict resolution and online play.

Skip the second copy when nothing reads during the edit, when memory is tighter than the tearing risk and you can sync to the display refresh, or when a swap is slower than the edit. Almost every graphics API already presents a finished frame. Another buffer does not fix tearing the API already handles.
