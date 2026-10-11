# Examples

Original sketches. They show the shape of a deeper interface. They are not taken from any book.

## Deepen a common path

Shallow: every caller learns the queue, the priority, and the clock.

```text
enqueue(job, queue, priority, clock)
```

Deep: the common call submits work. Fairness, retries, and the clock stay inside. The rare path is a separate method.

```text
scheduler.submit(job)
scheduler.expedite(job)
```

## Hide a shared decision

Two stages both know the record layout, so a format change hits both.

```text
parse(bytes) -> doc
write(doc) -> bytes
```

One module owns the layout. A rare override stays off the common path.

```text
store.save(doc)
store.load(id)
store.save(doc, schemaVersion=2)
```

## Separate mechanism from one caller

The limiter also knows one dashboard.

```text
allow(key, routeName):
    ok = take(key)
    emit("dash." + routeName, ok)
    return ok
```

The mechanism does not mention the dashboard. The caller records the outcome.

```text
ok = limiter.allow(key)
dashboard.record(routeName, ok)
```

## Different abstractions in adjacent layers

A layer that only forwards is the same abstraction twice.

```text
quote(req):
    return pricer.quote(req)
```

Each layer owns a different decision.

```text
quote(req):
    priced = pricer.price(req.lines)
    return policy.decide(priced, req.customer)
```

## Define a missing case out of existence

"Release an existing hold" fails when the hold is already gone, and every caller handles that.

```text
release(lockId):
    if lockId not in held: fail NotHeld
    held.remove(lockId)
```

"Ensure no hold remains" has no error case.

```text
release(lockId):
    held.remove(lockId)
```

## A name that prevents a bug

`flag` forces the reader into the assignments. `paid` is true or false without the declaration. A pair `(sent, dropped)` forces every caller to remember the order. A `SendOutcome` with those field names is obvious at the call site.

## Pull a knob down

Callers should not pick a cache policy they cannot tune better than the module.

```text
cache.get(key, ttl, maxEntries)
```

```text
cache.get(key)
```

Expose an override only when a higher layer has information the cache cannot see, and give the computed policy as the default.

## Join conjoined methods

Every caller must open a session, write, and close, and the three only make sense together.

```text
session = open(path)
session.write(bytes)
session.close()
```

One operation owns the lifetime.

```text
store.write(path, bytes)
```

Keep `open` only if callers truly interleave other work inside the session. That is a different abstraction, and it belongs on the interface because callers must know it.
