# Optimization

Use these when a profile, a fragmentation failure, or a quadratic query says the simple version is the bottleneck. During design, prefer layouts a hot loop can scan. Do not rewrite cold code in advance.

## Flyweight

Share the heavy identical part of a huge number of similar objects. Store only what differs per instance.

Split intrinsic state (mesh, material, movement cost: context-free and immutable) from extrinsic state (position, size, tint). Many instances hold a reference to one shared record plus a few small fields. Upload the shared mesh once and draw instances. A grid can store a small id into a handful of immutable records instead of copying those bytes into every cell.

Use it when instance counts are large and most of the bytes are repeated. Skip it when counts are small, each instance is actually unique, or a packed table is already clear. Indirection can miss cache. Measure before adopting sharing for style, and measure before abandoning it.

Shared records must outlive every user. A write to a shared record shows up everywhere, so keep them immutable. Sharing must not change visible behavior.

A type object also points many instances at one shared definition. Its goal is fewer types, and any memory win is incidental. Flyweight is only about cost. See [behavior.md](behavior.md) when the goal is authoring kinds.

## Data locality

Place data contiguously in the order a hot loop walks it. A fetch from distant memory costs on the order of hundreds of cycles. Hardware pulls a whole neighboring line. Pointer-heavy graphs miss constantly. The same work in a different layout can differ by a large factor. The ratio depends on the machine. Use a cache profile, not only a timer.

- Store domain pieces in a contiguous array, not an array of pointers. Walk physics, then rendering.
- A fat record that mixes hot and cold fields pulls cold bytes into the hot walk. Keep per-frame fields in the walked array. Move rare data to a side structure.
- Pack live items at the front of a pool and walk only that count. Swapping a dead item with the boundary is often cheaper than chasing holes. Profile before trusting that.

You give up virtual calls on that path. A pointer hop plus an indirect call is a miss. Separate arrays per concrete type restore a static call and force you to know the types. That trade is for the hot path only.

Pointers into a packed array break when you swap for locality. Store an id. Two threads writing the same line bounce it between cores. Data they both mutate should sit on different lines.

Components make this easier because the loop already updates one domain at a time. A pool of one type is the contiguous array. Packing the live prefix is what makes the pool local.

## Dirty flag

Skip expensive derived work until something needs the result, then do it once.

The usual case is a local transform whose world matrix is the chain of locals from the root. Most objects do not move. Recomputing on every edit is worse when several locals on one chain change in one frame: each edit rebuilds that node and every descendant, and the reader only needed the last result.

Store the last derived value and a dirty bit. The write sets the bit and does not walk children. The consumer, walking parent before child, recomputes if this node or any ancestor is dirty, stores the value, and clears the local bit.

Use it when the derived work is expensive, primary data changes more often than the result is read, and you cannot patch the derived value in place. Skip it when you read the result after every write, when an incremental update is easy (add and subtract a total), or when doing the work at the moment of use would hitch.

Set the bit on every primary write. The safe shape is one API that both mutates and dirties. The parent-dirtiness trick works only when that single traversal is the only reader. Anything else must share the path or mark descendants itself. Clear the bit at the read, at a checkpoint, or on a background timer, and accept the cost of that choice. A missed write serves stale data that is hard to find.

## Object pool

Reuse objects from a fixed collection so create and destroy do not allocate each time.

Use it when you create and destroy often, instances are similar in size, and allocation is slow, fragments, or wraps an expensive resource such as a voice. Skip it when creation is rare, sizes vary wildly, or a compacting collector already removes the fragmentation reason and allocation is cheap. A free list of recycled nodes is enough when you do not need a hard cap or a contiguous walk.

Create the whole set up front, unused. Acquire initializes a free slot and marks it in use. Release marks it unused. Scan for a free slot while the pool is small. When it is large and mostly full, thread a free list through the unused slots. Packing live slots at the front optimizes the per-frame walk. The free list optimizes finding a slot.

The cap is a feature when an effect must not consume memory the game needs. On failure: size the pool so important objects never fail, skip a spawn the user will not notice, steal the least important live object, or grow an overflow and decide whether to shrink it. Per-level sizes are reasonable when one level is effects-heavy and another is not.

Slots are fixed size. A pool of variants must fit the largest and wastes space on the smaller ones. Split into size classes. Reuse does not clear memory. Forgotten initialization looks almost correct. Clear slots in debug on release, and initialize every field. If callers keep a reference, do not silently reuse that slot under them. Under a collector, clear references inside a released object or the collector keeps the rest of the graph alive.

## Spatial partition

Find objects near a position by storing them according to position. The structure is a broad filter. Exact distance tests still run on the candidates it returns. It does not decide the hit.

Use it when pair tests or neighbor queries dominate and the set is large. Skip it when the set is small, objects move so often that bookkeeping dominates, memory is tighter than time, or the query radius covers so many cells that you still test almost everyone.

- **Flat grid.** Fixed cells, each a list of objects whose centers fall there. A query looks at the object's cell and, if the radius is larger than zero, at neighbors. Test each unordered pair once. Moves inside a cell only update coordinates. Crossing a boundary unlinks and relinks. Memory is fixed up front. Accept clumps and empty cells.
- **Quadtree or octree.** Split a region only while it holds too many objects, on fixed halfway planes. One insertion triggers at most one split. Use it when density varies and objects are added and moved one at a time.
- **A tree built in a batch.** Use it when the set is static or rebuilt together and you want each region to hold a similar count. Better for level geometry than for units that stream in.

Keep a second flat list of live objects when you also walk everyone regardless of place, and the grid is full of empty cells. Keep the two in sync on create and destroy. A move in a tree is a remove plus an add. If cell boundaries depend on the object set, moving one object can reshuffle others.

Live objects and static geometry often want different partitions.
