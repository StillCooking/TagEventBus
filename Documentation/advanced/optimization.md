# Optimization

*Where the costs come from on the hot path: tag depth, payload copying, the filter gates, and the sticky sweep.*

## Tag depth

> [!warning]
> **Walking up the ancestor chain always has a cost, even when there are no `Partial` listeners.** <!-- fact:marsz-po-przodkach-kosztuje-zawsze-nawet-bez-listenerow-part -->
> `Combat.Hit` is one extra map lookup; `Combat.Hit.Melee.Critical.Back` is four.

Every extra level of depth is one extra lookup on a broadcast, so **it pays to keep hot tags <!-- fact:glebokosc-tagu-to-liczba-dodatkowych-lookupow-mapy-przy-kazd --> <!-- fact:kazdy-dodatkowy-poziom-glebokosci-w-tagu-to-jeden-dodatkowy -->
shallow**.
That cost is incurred whether or not any listeners are registered on ancestor tags.

A `Partial` listener adds no lookups of its own beyond [that walk](../concepts/broadcast-path.md); delivering to it costs the
same as delivering to any other listener. Processing the exact channel costs one lookup plus a
linear loop over its listeners. The ancestor lookups still occur even if every registered
listener uses `Exact`.

That makes the tag hierarchy a performance decision, not just an organizational one. A tag
broadcast every frame should have two levels, not five.

## Payload <!-- fact:payload-zdarzenia-moze-przyjac-cztery-formy-signal-only-pust -->

Four forms:

| Form | Where | Allocation |
| --- | --- | --- |
| the signal alone — an empty `FInstancedStruct` | BP and C++ | none |
| a wildcard struct — `Broadcast Tag Event Struct` | BP | none |
| `FInstancedStruct` — `Broadcast Tag Event` | BP | yes |
| `Broadcast<T>` with `const T&` | C++ | none |

The native template call `Bus->Broadcast<T>(...)` allocates nothing. <!-- fact:natywne-wywolanie-szablonowe-bus-broadcast-t-z-c-jest-zero-a -->

The main payload-copying costs are: <!-- fact:koszt-kopiowania-payloadu-zalezy-od-sciezki-zero-kopii-dla-n -->

| Path | Copies                                                                  |
| --- |-------------------------------------------------------------------------|
| a native listener, immediate broadcast | zero                                                                    |
| a Blueprint listener | one into `FInstancedStruct`, **at most once per channel per broadcast** |
| a deferred broadcast | one on enqueue                                                          |
| `bRetain` | one into sticky storage                                                 |
| a sticky replay | zero                                                                    |

"At most once per channel" matters here: ten Blueprint listeners on one tag still cost one
copy, not ten.

The fluent `Deferred()` path adds one payload copy compared with `BroadcastDeferred<T>`.

`bRetain` with `RetainTTLSeconds` costs the same as `bRetain` on its own, plus one `double` and <!-- fact:bretain-retainttlseconds-kosztuje-jak-zwykly-bretain-plus-je -->
a counter increment, with no extra allocation. `ContextTags` is one container copy per
broadcast, not per listener.

## Optional filters and deferred coalescing

All four are opt-in. Their costs are summarized below: <!-- fact:cztery-bramki-filtrujace-sa-opt-in-i-nie-kosztuja-nic-gdy-ni -->

| Feature | Cost when used |
| --- | --- |
| `Query` | an `IsEmpty()` branch per listener; the target object is built lazily, once per broadcast |
| `Predicate` | one `TFunction` call per delivery attempt that passes the earlier gates — **it must be cheap and free of side effects** |
| `Throttle` | `O(1)` |
| `bCoalesce` | a linear scan of the deferred queue on every enqueue that carries the flag |

`bCoalesce` is the only one whose cost grows with load — with a long deferred queue the scan
becomes noticeable.

## Sticky values with a lifetime

There are two costs: <!-- fact:koszt-sticky-ttl-ma-dwie-sciezki-lazy-guard-przy-odczycie-to -->

- **A read** — one `double` comparison.
- **An active sweep** — `O(1)` when no sticky value carries a TTL, thanks to an incremental
  counter. Otherwise a **linear scan of every channel in the registry**, not only those with
  a TTL.

The global bus runs TTL sweeps on an always-on ticker, so that work happens within an existing tick.
The local bus ticks lazily: it starts ticking when a TTL is armed and stops only once the <!-- fact:bus-globalny-zamiata-ttl-na-zawsze-wlaczonym-tickerze-koszt -->
deferred queue is empty and no sticky value has a TTL.

A component with an empty deferred queue and no sticky values carrying a TTL **does not tick at
all**.

## Measuring

Broadcast timing is measured by the `TagEventBus Broadcast` cycle counter in the `STATGROUP_TagEventBus` group. Use the **`stat TagEventBus`** command to view the measurements. <!-- fact:broadcast-jest-objety-licznikiem-cyklu-tageventbus-broadcast -->

The tests check correctness, not performance — that command is the tool for timing.

> [!warning]
> **Turn off Record in the debugger panel before you measure.** In Development, recording the <!-- fact:przed-pomiarem-wydajnosci-trzeba-wylaczyc-record-w-developme -->
> history and leaving the Live tab open add a real cost to every broadcast. In Shipping the
> problem goes away on its own, because that layer is not there.

With recording on, the measurement includes diagnostic overhead as well as the bus's own work.

## What the optional modules add

The optional modules add no overhead to the hot delivery path while their functionality is <!-- fact:dolozenie-warstw-requests-contracts-nie-zmienia-kosztu-gorac -->
inactive. With `EnforcementLevel = Off`, the contract validator hook is not installed.

---

← [Advanced](README.md) · [Documentation index](../README.md)
