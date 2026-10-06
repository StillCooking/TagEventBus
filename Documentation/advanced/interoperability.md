# Interoperability

*The division of labor between the bus, GAS, and Event Dispatchers, the network boundary, module dependencies, and a trap with wildcard nodes after an update.*

## GAS

GAS and the bus solve different problems, despite sharing the same address type,
`FGameplayTag`. <!-- fact:gas-i-tageventbus-rozwiazuja-rozne-problemy-mimo-wspolnego-t -->

**How to choose:** activating an ability on a specific actor → GAS. Notifying whichever systems are interested → the bus.

GAS routes a `GameplayEvent` to a specific actor with an `AbilitySystemComponent`. The bus
handles communication where **neither side requires an ASC**.

> [!tip]
> Split the tag namespaces — `Ability.*` for GAS and `Event.*` for the bus, for example. Both <!-- fact:gas-i-szyna-korzystaja-z-tego-samego-slownika-tagow-wiec-war -->
> systems read the same list, so without that separation, the tag picker mixes tags intended for the two systems, making it hard to tell which tag serves which purpose.

## Event Dispatchers

The bus **does not replace** Event Dispatchers.

| Situation | Tool |
| --- | --- |
| the reference exists anyway; the dependency is local and obvious | Event Dispatcher |
| the receiver would have to go looking for the sender | the bus |
| there are many receivers and they are not known up front | the bus |

A dispatcher on a component you already reference through a field is simpler and cheaper. The bus starts paying off only when you would otherwise need a reference solely for event delivery. <!-- fact:szyna-nie-zastepuje-event-dispatcherow-dispatcher-jest-lepsz -->

## The network

The bus does not replicate. That is no obstacle in a networked game, **as long as you treat <!-- fact:brak-replikacji-nie-przeszkadza-w-grze-sieciowej-o-ile-szyna -->
it as communication inside a single instance** and let the engine's own mechanisms handle
the network boundary — RPCs and property replication. Within each instance, the bus
distributes the event to systems such as UI, audio, and VFX, keeping them decoupled from
network communication.

The pattern is this: an RPC carries the event across the network boundary, and a bus
broadcast distributes it to systems in the receiving instance.

## Module dependencies

**The `TagEventBus` core depends on none of the other modules.** The dependencies are <!-- fact:rdzen-tageventbus-nie-zalezy-od-zadnego-z-pozostalych-modulo -->
strictly one-way: everything points to the core, never the other way around.

The descriptor registers four modules: the core, the editor debugger, and two optional Runtime <!-- fact:deskryptor-rejestruje-piec-modulow-rdzen-editor-debugger-dwa -->
modules. All four are built in an editor configuration; "optional" means you
**do not have to use them or link against them**.

To use the core bus API from C++, add `TagEventBus` and `GameplayTags` to the consuming <!-- fact:gameplaytags-jest-potrzebny-w-module-konsumenta-bo-jego-kod -->
module's `Build.cs`. The second one is necessary because your own code declares and passes
`FGameplayTag`, and a plugin's dependencies do not carry over automatically.

## The trap: wildcard nodes after an update

> [!warning]
> `Broadcast Tag Event Struct` and its deferred variant are `CustomThunk` nodes with the <!-- fact:wezly-customthunk-z-payloadem-na-pinie-wildcard-broadcast-ta -->
> payload on a wildcard pin. Blueprint **cannot update their native signatures on its own** —
> after [a plugin update](../project/updates.md) the new pin does not appear on saved graphs until you run
> **Refresh Node** or recompile the Blueprint.
> Ordinary `UFUNCTION` nodes do not need this.

## Extending it

The plugin **has no plugin system**. You can extend it in three ways — the first two are the approaches its own modules use: <!-- fact:plugin-nie-ma-systemu-wtyczek-rozszerza-sie-trzema-drogami-u -->

1. **A layer over the public API** — that is how `TagEventBusRequests` came about:
   a broadcast with a policy, plus consuming the event.
2. **The validator hook** — that is how `TagEventBusContracts` came about.
3. **Composition in your game code.**

## Deliberate asymmetries

A few things look like omissions but are deliberate choices:

- **There is no typed listen node in Blueprint.** The async node's output delegate cannot <!-- fact:nie-ma-dedykowanego-typowanego-wezla-nasluchu-w-blueprint-bo -->
  carry a wildcard pin; a typed pin would need a `K2Node` of its own rather than the plain
  `CustomThunk` used on the sending side. The two-node pattern — the bind node plus the
  engine's own `Get Instanced Struct Value`, whose `Valid` and `Not Valid` outputs stand in
  for the branch — gives the same result without the maintenance burden of a custom
  `K2Node`.
- **The payload predicate is C++ only.** The cost of calling a Blueprint delegate on every <!-- fact:predykat-payloadu-jest-tylko-c-bo-natywny-callable-nie-ma-se -->
  matching broadcast would contradict the "cheap, no side effects" contract.
  `ThrottleSeconds` is in Blueprint because it is a simple timestamp comparison.
- **`UTagEventBusComponent` has no `BroadcastInstanced` variant** — it provides <!-- fact:utageventbuscomponent-swiadomie-nie-ma-legacy-broadcastinsta -->
  `BroadcastInstancedWithPolicy` instead.
  That is an accepted asymmetry between the component and the global bus, not an oversight.
- **`ContextTags` is not a parameter of the listener delegate.** The `FTagEventDynDelegate` <!-- fact:sygnatura-ftageventdyndelegate-delegat-listenera-bp-nie-zmie -->
  signature did not change when that feature was added — you read the context with a
  separate node.

---

← [Advanced](README.md) · [Documentation index](../README.md)
