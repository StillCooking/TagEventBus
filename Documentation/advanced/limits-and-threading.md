# Limits and threading

*What the bus does not do by design, what is not implemented yet, and what disappears in a Shipping build.*

## Game Thread only

Every public function that mutates the registry contains <!-- fact:kazda-publiczna-mutacja-rejestru-zawiera-ensuremsgf-isingame -->
`ensureMsgf(IsInGameThread(), ...)`. **There are no locks and no atomics**, so concurrent
access is undefined behavior.

> [!warning]
> The `ensure` reports a violation of this rule only in development builds. **In Shipping, this check produces no warning**, so binding or broadcasting from a worker thread goes unreported in that build and can silently corrupt the registry. <!-- fact:wszystkie-ensure-znikaja-w-shipping-zlamanie-kontraktu-game -->
>
> Run the tests and the game in Development before you build for production.

If an event originates on a worker thread, schedule the broadcast on the Game Thread using
the engine's own mechanism.

## No replication

The bus is local to the game instance — events do not travel over the network. <!-- fact:szyna-nie-replikuje-jest-lokalna-dla-instancji-zdarzenia-nie -->

The registry is transient and is not serialized. Only the retained sticky values survive,
and only through an explicit dump and restore of the state.

In practice this is no obstacle in a networked game, as long as the engine's own mechanisms
handle the network boundary and the bus delivers the event within a single instance —
[Interoperability](interoperability.md).

## By design or not implemented yet

The distinction matters, because it tells you what is worth waiting for.

**By design — this is the shape of the product:**

- **Game Thread only.** Keeping the registry free of locks is a performance decision.
- **No dependencies outside the engine.** No third-party code, no `Boost`, no graphics
  requirements beyond the engine; distributing your game imposes no extra licensing
  conditions.
- **No plugin system.** You extend it using the three documented approaches, not through an extension registration mechanism.
- **No automatic deduplication.** The plugin does not merge or aggregate events on its own, <!-- fact:plugin-nie-deduplikuje-ani-nie-agreguje-zdarzen-automatyczni -->
  beyond the explicit coalescing in the deferred variant.

**Not implemented yet — what is missing today:**

- **Replication.** The bus does not replicate, and the documentation does not commit to a release date. This
  is not a design boundary; it is a feature that did not make the cut.
- **A view of local buses in the debugger panel.** [The panel resolves the global bus
  alone](../concepts/debugger.md); extending it is an open design question rather than a
  missing line of code — [Roadmap](../project/roadmap.md).

## What disappears in Shipping

| Layer                                                                            | Shipping |
|----------------------------------------------------------------------------------| --- |
| binding, broadcasting, sticky, deferred, pause, save state, and request-response | **work identically** |
| the history and the statistics                                                   | gone |
| the debugger panel and the whole editor module                                   | gone |
| contract validation                                                              | gone |
| `ensure` checks for API contracts                                                | gone |
| `Verbose` and `VeryVerbose` logs                                                 | stripped by the engine's build system |

In Shipping, `GetListenerDebugInfo` and `GetBroadcasterDebugInfo` remain callable but
return empty results, `GetTagStats` returns `false`, and `OnBroadcastRecorded` never fires.

> [!warning]
> The editor module `TagEventBusEditor` — the debugger panel and its types — is not built in Shipping. Code in a runtime module that references it will not compile there, and the error will only surface when you first build for Shipping. <!-- fact:w-shipping-cala-powierzchnia-diagnostyczna-historia-statysty -->

`Block` in contracts shares that fate: it is a dev/CI gate, not a production safeguard — in
a production build no broadcast is ever rejected by a contract.

## Things that look like limitations but are not

- **Pause works in Shipping too.** It is a runtime feature, not part of the diagnostic <!-- fact:pauza-dziala-tez-w-shipping-to-funkcja-runtime-owa-nie-czesc -->
  surface.
- **A briefly inflated channel count** during a broadcast is a consequence of deferred
  removal, not a leak.

---

← [Advanced](README.md) · [Documentation index](../README.md)
