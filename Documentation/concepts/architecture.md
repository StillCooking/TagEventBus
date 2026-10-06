# Architecture

*The global bus and the local bus are two disjoint registries, not one registry with a filter — and that is the first thing people get wrong.*

## Two registries, not one with a filter

> [!warning]
> **Broadcasting on an actor's local bus while listening on the global one puts you on two
> different buses.** From the outside the two calls look identical: the same node, the same
> tag, the same payload. One `Scope` pin separates them, and the result of getting it wrong is
> silence with no error.

`Scope` is not a filter laid over a shared registry. It picks **which registry** the call
goes to:

| Scope | Where the registry lives | Where it comes from |
| --- | --- | --- |
| `Global` | `UTagEventBusSubsystem` on the `GameInstance` | appears on its own |
| `Local` | `UTagEventBusComponent` on an actor | you add it explicitly or enable creation when binding |

By default, calls target the global bus. To use a local bus, add a `UTagEventBusComponent` <!-- fact:domyslnie-wszystko-idzie-przez-bus-globalny-bus-lokalny-per --> <!-- fact:nic-nie-tworzy-lokalnego-busa-niejawnie-jedynym-wyjatkiem-je -->
to the actor or enable `bCreateLocalIfMissing = true` when binding. Broadcast calls, <!-- fact:w-zakresie-local-wariant-deferred-wymaga-istniejacego-kompon -->
including deferred broadcasts, require an existing component and never create one.

![The Scope pin branches into two disjoint boxes: on the left UTagEventBusSubsystem on the GameInstance, on the right UTagEventBusComponent on an actor. Each has its own sender, its own channel registry, and its own receiver; the connector between the registries is crossed out](../Assets/tageventbus-05-two-registries.svg)

*Scope picks a registry rather than filtering a shared one. An event does not cross between the boxes.*

## The lifecycle of the two buses

**The global bus** is created automatically with the `GameInstance` and goes away with it. <!-- fact:bus-globalny-powstaje-automatycznie-z-gameinstance-i-znika-w -->
`Deinitialize` detaches the hooks and clears the registry. **The local bus** is created
explicitly and goes away with the actor: `EndPlay` clears the registry and detaches the
post-GC hook.

The consequence in Play In Editor: **every PIE session has its own `GameInstance`, and <!-- fact:kazda-sesja-pie-ma-wlasny-gameinstance-i-wiec-wlasny-bus-glo -->
therefore its own global bus.** The statistics reset when PIE starts, and events from one
session are not visible in another.

The two buses also differ in what they cost when idle. The global subsystem keeps an <!-- fact:utageventbussubsystem-dodatkowo-trzyma-ftsticker-flushujacy -->
`FTSTicker` that flushes the deferred queue once per tick, a post-GC hook that sweeps out
[listeners whose owner has been destroyed](lifecycle.md), and an `AddReferencedObjects` implementation that
keeps the objects referenced by retained and deferred payloads alive.
The local component flushes the deferred queue on its own tick, which is switched on and off <!-- fact:bus-lokalny-komponent-flushuje-deferred-wlasnym-tickiem-wlac -->
automatically; the same tick sweeps out sticky values whose TTL has expired. A component
with no deferred queue and no sticky values carrying a TTL **does not tick at all**.

## The single chokepoint

`FTagEventRegistry::Broadcast` is the one point every delivery passes through, including deliveries made when the deferred queue is flushed. A breakpoint there catches everything. <!-- fact:ftageventregistry-broadcast-jest-jednym-choke-pointem-przez -->

Listener registration ultimately passes through <!-- fact:ftageventregistry-addnativelistener-i-addscriptlistener-to-w -->
`FTagEventRegistry::AddNativeHandlingListener` or `AddScriptListener`; `AddNativeListener`
delegates to `AddNativeHandlingListener`.

A breakpoint there is useful when the panel and the logs are not enough. For ordinary flow tracing you
usually do not need a C++ debugger — the in-editor panel and category-based logging show the same
thing faster, and without stopping the game.

## What the two buses do share

One thing crosses the scope boundary: **the dispatch frame stack**. There is exactly one, <!-- fact:stos-ramek-dispatchu-jest-jeden-wspoldzielony-przez-bus-glob -->
shared by the global bus and every local bus. That is why `Consume Tag Event` and
`Get Current Event Context Tags` are static and operate on the deepest active frame, no
matter which bus that frame belongs to.

The registries are disjoint. The control flow is not.

---

← [Concepts](README.md) · [Documentation index](../README.md)
