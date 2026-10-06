# Common problems

*From symptom to cause — silence after a broadcast, a linker error, a missing menu entry, and a queue that never empties.*

## An event does not arrive, and there is nothing in the log

The most common symptom, and the one with the most possible causes. **Check in this order** —
from the least to the most expensive check: <!-- fact:tag-spoza-slownika-gameplay-tags-jest-cicho-odrzucany-zarown -->

1. **Is the tag in the project's Gameplay Tag list?** A tag outside it is rejected on both
   sides, and not always loudly — [Configuration](../start/configuration.md).
2. **Are the sender and the receiver in the same `Scope`?** `Global` and `Local` are two <!-- fact:nadawca-i-odbiorca-musza-uzyc-tego-samego-scope-dla-tego-sam -->
   disjoint registries — a global broadcast **never** reaches a local listener, even on an
   identical tag.
3. **Is the listener waiting for a parent tag without `Partial`?** A listener on `Combat`
   with the default `Exact` will not see `Combat.Hit`.
4. **Is the listener paused?** Pause skips; it does not buffer.
5. **Is one of the remaining gates blocking delivery?** `Query`, the predicate, the
   throttle — none
   of them logs, because that would be spam on the hot path.
6. **Does the listener still exist at all?** See the section on leaks below.

> [!tip]
> **The shortest route to a verdict:** [a manual broadcast from the Inject tab](../concepts/debugger.md). If `Delivered` increases, the receiver works — check the sender. If it does not, check the receiver.

## `unresolved external symbol` when building

Either `"TagEventBus"` or `"GameplayTags"` is missing from your module's `Build.cs`.

Both entries are needed: `GameplayTags` because your own code declares and passes
`FGameplayTag`, and a plugin's dependencies do not carry over to the consumer.

If you work only in Blueprints, you will never see this error — the nodes work with no
changes to `Build.cs` at all.

## The Tools → Debug → TagEventBus Debug entry is missing

The plugin is disabled, or the editor module did not build.

1. Check whether the plugin shows up in the Plugins browser. If it does not, the `.uplugin`
   file is in the wrong directory or the folder name has a typo; the file has to sit
   directly in `Plugins/TagEventBus/`.
2. If you enabled the plugin by hand, restart the editor.
3. Check whether the build in the Development Editor configuration succeeded.

## A new pin did not appear after an update

This applies to `Broadcast Tag Event Struct` and its deferred variant. They are
`CustomThunk` nodes with the payload on a wildcard pin; Blueprint **cannot update their
native signatures on its own**.

Run **Refresh Node** on the existing nodes, or recompile the Blueprint.
Ordinary `UFUNCTION` nodes do not need this.

## The `Q deferred` counter never drops to zero

> [!warning]
> A handler that queues a deferred broadcast **on a tag it is itself bound to** creates an
> infinite loop. An enqueue performed during a flush lands on the next flush, and the queue has <!-- fact:handler-enqueue-ujacy-broadcast-deferred-na-tag-ktorego-sam -->
> no limit — so the loop does not blow up with an error; it just grinds on forever. The
> symptom is exactly that: a `Q deferred` that never falls to zero.

A deferred broadcast is safe from inside a handler for a **different** tag. On the same tag
you need an exit condition.

## A growing listener count on a tag

This is not the kind of leak tracked by the `DEAD` counter. The listener is not dead — **the receiver is a
different object now**. The typical case: a widget was recreated and bound a second time,
while the old binding is still in place. <!-- fact:wariant-mylacy-z-wyciekiem-listener-nie-jest-martwy-tylko-od -->

The symptom is a growing listener count on the tag, **not the `DEAD` counter**.

The fix: unbind in `EndPlay` or `Destruct`, or use a managed binding, which does the same
thing for you.

## Behaviors that look like a fault <!-- fact:osiem-zachowan-brzegowych-warto-znac-get-zwracajacy-nullptr -->

| What you see | What it means                                                             |
| --- |---------------------------------------------------------------------------|
| `Get()` returns `nullptr` | legitimate during early startup and during teardown                       |
| a listener skipped, a `Warning` in the log | the owner was collected by GC — an explicit unbind is missing             |
| a restore did nothing, an `Error` in the log | a dump from a newer version; the registry remains unchanged              |
| a restore skipped one entry | an unknown struct; the result is `RestoredWithDrops`                      |
| `Consume Tag Event` did nothing, a `Warning` | called outside dispatch, after a `Delay`, for example                     |
| `RetainTTLSeconds` had no effect, a `Warning` | passed without `bRetain` — it is ignored                                  |
| a negative `ThrottleSeconds` behaves like `0` | it is clamped; the fluent layer also logs a `Warning`                     |
| a deferred broadcast in `Local` rejected on enqueue, a `Warning` with the actor's name | there is no local bus component; a deferred broadcast will not create one |

---

← [Troubleshooting](README.md) · [Documentation index](../README.md)
