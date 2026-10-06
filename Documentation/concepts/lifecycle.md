# Lifecycle

*Listener cleanup: managed bindings, explicit unbinding, and the safety nets that catch what remains.*

## Three ways to unbind

| Route | What you get | When                                       |
| --- | --- |--------------------------------------------|
| the raw `FTagEventHandle` | an explicit `Unbind` by handle | full control, in C++ and in Blueprint      |
| `Bind Tag Event (Managed)` | a `UTagEventBinding` object, `Unbind()` at any time, automatic unbind on the owning actor's `EndPlay` | a typical listener in an actor or a widget |
| listener group | one `Unbind All` for a dozen or more bindings at once, plus a shared pause | an object listening on many tags           |

A managed binding is the default choice wherever an actor is the owner. Leave the raw handle
for [the cases where you genuinely need to release one specific binding](../reference/bind-unbind.md) outside the owner's normal cleanup lifecycle.

## The post-GC sweep is a fallback

The plugin has three automatic safety nets: the post-GC sweep, an RAII handle on the C++
side, and the automatic unbind on `EndPlay`.

> [!warning]
> **Choose how each listener will be unbound.** Use a managed binding for automatic cleanup on <!-- fact:zadna-z-trzech-automatycznych-siatek-bezpieczenstwa-sweep-po -->
> the owning actor's `EndPlay`, or unbind explicitly during teardown. Do not rely on the post-GC
> sweep as your cleanup strategy. <!-- fact:kazde-sprzatniecie-listenera-przez-hook-post-gc-zapisuje-war -->

The post-GC sweep is not silent. Every cleanup it performs writes a `Warning` to
`LogTagEventBusListeners`, together with the tag, the handle, and the kind of listener.

That `Warning` means the listener was still registered when its owner was collected. With a
C++ RAII subscription, the sweep can run before the subscription's destructor. Unbinding
during teardown avoids that warning.

## How to see a leak

The debugger panel shows subscription leaks visually: a red `DEAD` in the tag tree and an
`N DEAD` counter in the summary bar.

A growing `DEAD` count within a PIE session indicates that listeners remain registered after
their owners have been collected.

## After the bus owner dies

Code holding an `FTagEventSubscription` does not have to check whether its bus is still <!-- fact:po-smierci-wlasciciela-busa-destruktor-reset-subskrypcji-jes -->
alive. The subscription's destructor and `Reset()` are safe no-ops, `IsValid()` returns
`false`, and an explicit `Reset()` in `EndPlay` is no longer a safety requirement — it
remains good practice for a deterministic teardown.

Why this had to be built at all: **the registry is not protected by GC.** Without a lifetime <!-- fact:rejestr-nie-jest-chroniony-przez-gc-bez-tokenu-zycia-uchwyt -->
token, an RAII handle holding a raw pointer could dereference freed memory during world teardown, when the destruction order is not obvious.

## Pause is not an unbind

A paused listener stays in the registry and stops receiving events. Events are not buffered
for the paused listener. On resume, `bReplaySticky` can replay the tag's retained value.

Pause is for silencing something for a while — for a cutscene, a loading screen, a
transition between states. To remove a listener permanently, unbind it.

---

← [Concepts](README.md) · [Documentation index](../README.md)
