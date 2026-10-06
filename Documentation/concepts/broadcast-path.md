# Broadcast path

*Five broadcast steps, five gates for each listener, the priority bands, and a fixed order that new options do not change.*

## The broadcast path — five steps

Every broadcast follows the same route: <!-- fact:sciezka-broadcastu-ma-stale-piec-krokow-walidacja-tagu-hook -->

1. **Tag validation.** A tag outside the project's tag list is rejected here.
2. **The validator hook** — outside Shipping. This is where contracts run.
3. **The exact channel**, in priority order.
4. **The walk up the ancestors** — listeners with `Match = Partial` only.
5. **Retaining the value**, if `bRetain` was set to `true` for the broadcast.

Steps 3 and 4 can be interrupted by consumption: `StopPropagation`, or `Handled` under the <!-- fact:kroki-3-i-4-sciezki-broadcastu-kanal-dokladny-i-marsz-po-prz -->
`StopAfterHandled` policy.

## Delivery order

A broadcast on `Combat.Hit.Critical` first considers listeners registered on that exact tag, <!-- fact:broadcast-na-combat-hit-critical-trafia-najpierw-do-listener -->
then `Partial` listeners on `Combat.Hit`, then `Partial` listeners on `Combat`. **Always
the whole exact channel before the ancestors**.

> [!warning]
> **A listener on a parent tag does not match child tags until you set `Match = Partial`.**
> Binding on `Combat` and waiting for `Combat.Hit` with the default `Exact` ends in silence —
> no warning, no error, no log entry. That mismatch is the most common cause of an event that
> "does not arrive". <!-- fact:dopasowanie-partial-dziala-tylko-w-jedna-strone-listener-na -->

`Partial` matching works **in one direction only**: a listener on an ancestor matches events
from its descendants, never the other way around. <!-- fact:hierarchia-tagow-ma-realne-konsekwencje-dla-dostarczania-lis --> <!-- fact:kropka-w-nazwie-taga-tworzy-hierarchie-rodzic-dziecko-ktora -->

A dot in a tag name creates the parent-child hierarchy that feeds this matching.
The structure of your tag names is therefore not cosmetic — it determines who gets what.

![A broadcast on Combat.Hit.Critical enters from the left into lane 1, containing listeners registered on Combat.Hit.Critical, ordered by descending Priority, then drops to lane 2 for Combat.Hit and lane 3 for Combat, both of them for Match = Partial only. The drop to the next lane is marked as possible only after the previous lane is exhausted](../Assets/tageventbus-06-delivery-order.svg)

*The exact channel in full, and only then the ancestors. Priority works inside a lane, not between lanes.*

## Five gates on a single listener

Before an event reaches a listener's handler, it passes through five gates in a fixed order: <!-- fact:dostarczenie-do-pojedynczego-listenera-przechodzi-przez-piec -->

| # | Gate | Available from Blueprint |
| --- | --- | --- |
| 1 | `Paused` | yes |
| 2 | payload type | yes |
| 3 | `Query` | yes |
| 4 | `Predicate` | **no** — C++/fluent only |
| 5 | `Throttle` | yes |

The `Predicate` gate is the only one not available from Blueprint. <!-- fact:bramka-predicate-jest-dostepna-tylko-z-c-fluent-pozostale-cz -->

**Whichever gate blocks delivery, the effect is identical:** the handler is not <!-- fact:niezaleznie-od-tego-ktora-z-pieciu-bramek-odciela-dostarczen -->
called, `bOnce` is not consumed, and the `Delivered` counter is not updated.

That matters when you read the panel: `Delivered = 0` means no listener received the event.
It does not tell you whether there were no matching listeners or whether delivery was blocked by
a gate. <!-- fact:throttle-jest-jedyna-bramka-ktora-stempluje-lastfiretime-i-t -->

`Throttle` also tracks `LastFireTime`: it is the only gate that updates it, and only when the event actually reaches the handler. A `Query`
match and a predicate that passes do not count toward it. <!-- fact:zapauzowany-listener-jest-pomijany-a-nie-buforowany-zdarzeni -->

Delivery to a paused listener is skipped. Events are **not buffered for that listener**,
`bOnce` is preserved, and the delivery counter is not updated. On resume, `bReplaySticky` can
replay the tag's retained value.
Pause works in Shipping too, because it is a runtime feature rather than part of the
diagnostic surface.

## Priorities

Priority is a plain `int32`: a higher value gets the event earlier. The bands below are a <!-- fact:konwencja-pasm-priorytetow-stale-tageventpriority-zwykly-int -->
convention from the `TagEventPriority` constants, not a rule — in-between values are allowed,
and a band means whatever the team agrees it means. The usefulness comes from everyone using
the same numbers.

| Band | Value | For |
| --- | --- | --- |
| `Validation` | 1000 | checking before the rest |
| `Modifier` | 500 | changing the event's data |
| `Default` | 0 | reacting normally |
| `UI` | −500 | showing an effect |
| `Monitor` | −1000 | watching, with no influence |

**Priority only determines delivery order within a channel.** The "exact channel → ancestors" order takes <!-- fact:priorytet-nie-przeskakuje-miedzy-kanalami-porzadek-exact-prz -->
precedence: a `Partial` listener on `Combat` with priority 1000 still gets the event after
an `Exact` listener on `Combat.Hit` with priority −1000.

There is one case where priority is briefly suspended: **a listener added during a broadcast <!-- fact:listener-dodany-w-trakcie-trwajacego-broadcastu-tego-tagu-ni -->
on the same tag has not been sorted into the channel yet.** A nested broadcast in the same
frame delivers to it last, whatever its priority; the order returns to normal once the
outermost broadcast finishes.

## Deferred broadcast

The deferred variant copies the payload on enqueue and postpones delivery until a queue <!-- fact:wariant-deferred-kopiuje-payload-przy-enqueue-co-czyni-broad -->
flush. This avoids an immediate recursive call when broadcasting from inside a handler, but
repeated deferred broadcasts still need an exit condition.

A ticker flushes the queue: the global one once per subsystem tick, the local one on the
component's tick. The [Optimization](../advanced/optimization.md) page
covers the copying cost on each path.

---

← [Concepts](README.md) · [Documentation index](../README.md)
