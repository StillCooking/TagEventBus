# Types and settings

*Enums, result and save state structs, and project settings.*

## Scope and match

### `ETagEventScope`

Which bus the call targets.

| Value | Meaning |
| --- | --- |
| `Global` | `UTagEventBusSubsystem` on the `GameInstance`; the default value of every `Scope` pin |
| `Local` | `UTagEventBusComponent` on an actor |

This is not a filter over a shared registry — the two scopes are two disjoint registries.
See [Architecture](../concepts/architecture.md).

### `ETagEventMatch`

The match policy of a single listener.

| Value | Meaning |
| --- | --- |
| `Exact` | matches that tag alone; the default value |
| `Partial` | matches descendant tags too |

With `Partial`, matching works **in one direction only**: a listener on an ancestor matches
events from its descendants, never the other way around.

## Result and delivery control

### `ETagEventResult`

The result of a `Try*` operation. `Success` means the operation succeeded; the other seven values each identify [a reason for failure](../concepts/error-handling.md):

| Value | The usual fix |
| --- | --- |
| `Success` | — |
| `InvalidTag` | register the tag in the project's tag list |
| `InvalidWorldContext` | pass an object that has a world context |
| `LocalActorNotResolved` | check whether the context object resolves to an actor |
| `LocalBusMissing` | add the bus component, or set `bCreateLocalIfMissing` |
| `DelegateNotBound` | wire a Custom Event into the `Event` pin |
| `InvalidHandle` | check whether the handle has already been unbound or has `Id = 0` |
| `PayloadTypeMismatch` | reserved — no function returns it in this version |

### `ETagEventHandling`

The result a listener reports for one delivery. The meaning is **fixed and independent of
the sender's policy**.

| Value | Meaning |
| --- | --- |
| `Continue` | the listener makes no claim on the event; a listener with no return value is an implicit `Continue` |
| `Handled` | marks the event as handled; it ends delivery **only** under the `Stop After Handled` policy |
| `Stop Propagation` | always ends delivery |

### `ETagEventBroadcastPolicy`

The sender-side policy, chosen on every call.

| Value | Meaning |
| --- | --- |
| `Broadcast All` | deliver to everyone; the default value, equivalent to a plain `Broadcast Tag Event` |
| `Stop After Handled` | end after the first listener that returns `Handled` |

### `ETagEventRestoreResult`

The result of `Restore Retained State`. <!-- fact:restoreretainedstate-ma-trzy-wyniki-restored-wszystkie-wpisy -->

| Value | Meaning                                                                                                           |
| --- | --- |
| `Restored` | every entry restored                                                                                              |
| `RestoredWithDrops` | restored, but at least one entry was skipped because its tag was invalid or its struct no longer exists           |
| `RejectedNewerVersion` | a snapshot with a newer schema version: **nothing was restored**, the registry was left untouched, and an `Error` was logged |

Rejecting a snapshot with a newer schema version outright is a decision, not an omission. If the snapshot were restored only partially, the game would start with silently truncated state; instead the restore does nothing and says so plainly.

## Structs

### `FTagEventBroadcastResult`

Returned by `Broadcast Tag Event (Policy)`.

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `Delivered` | `int32` | `0` | the number of listeners actually called |
| `bWasHandled` | `bool` | `false` | at least one listener returned `Handled` or `Stop Propagation` |
| `bWasStopped` | `bool` | `false` | delivery ended early |

`Delivered` means exactly the same as the number returned by a plain
`Broadcast Tag Event`.

### `FTagEventHandle`

| Field | Type | Default |
| --- | --- | --- |
| `Id` | `int64` | `0` |

A handle with `Id = 0` is invalid — `Try Unbind` reports `InvalidHandle` for it.

### `FTagEventBusSaveState`

A dump of the retained values, meant for a save game. **Listeners are not saved.**

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `SaveVersion` | `int32` | `TagEventBusSave::LegacyVersion` | the dump's schema version; `0` means a dump from before versioning was introduced |
| `Retained` | `TMap<FGameplayTag, FInstancedStruct>` | — | the retained payloads |
| `RetainedContextTags` | `TMap<FGameplayTag, FGameplayTagContainer>` | — | the context tags; saved only for non-empty containers |
| `RetainedTTLRemaining` | `TMap<FGameplayTag, float>` | — | the remaining lifetime; only entries with an active TTL are saved, and a restore resumes the countdown |

The last two fields are absent from saves made before they were introduced — older dumps are
migrated forward.

## Project settings

The settings in `UTagEventBusLogSettings` are available under **Project Settings → Plugins → Tag Event Bus - Logging** and saved to `Game.ini`.

| Property | Type | Default | Controls |
| --- | --- | --- | --- |
| `bMasterEnable` | `bool` | `true` | turning it off silences all seven configurable log sections, regardless of their individual settings; it does not affect `LogTagEventBus` |
| `BroadcastVerbosity` | `ETagEventBusLogVerbosity` | `Log` | delivery and payload type mismatch |
| `ListenersVerbosity` | `ETagEventBusLogVerbosity` | `Log` | binding and unbinding |
| `StickyVerbosity` | `ETagEventBusLogVerbosity` | `Log` | setting, replaying, and clearing sticky values |
| `SaveVerbosity` | `ETagEventBusLogVerbosity` | `Log` | dumping and restoring state |
| `LifecycleVerbosity` | `ETagEventBusLogVerbosity` | `Log` | the lifecycle of the subsystem, the component, and the registry |
| `AsyncVerbosity` | `ETagEventBusLogVerbosity` | `Log` | async actions and deferred broadcasts |
| `DebuggerVerbosity` | `ETagEventBusLogVerbosity` | `Log` | the debugger panel |

Every section starts at the `Log` level,
and changing these settings is optional — the defaults are a sensible starting point.

### `ETagEventBusLogVerbosity`

The level of a single section. `Off` silences the section completely; the remaining values
map one to one onto the engine's `ELogVerbosity` levels.

### `ETagEventBusLogSection`

Seven independently controlled sections, matching the seven verbosity properties in the
table above: broadcast, listeners, sticky, save state, lifecycle, async operations, and the
debugger.

### `ETagEventBusLogPreset`

Preset levels applied to every section at once: `Off`, `CriticalOnly`, `Normal`, `Verbose`,
and `All`.
You apply them with the `TagEventBus.Log.Preset` console command — see
[Debugging](../advanced/debugging.md).

## Types from the optional modules

The [Contracts](../advanced/contracts.md) page covers the contract
types (`ETagEventContractPayload`, `ETagEventContractRetain`, `ETagEventContractScope`,
`ETagEventContractEnforcement`, `ETagContractViolation`, `FTagEventContract`).

The [Request-Response](../advanced/request-response.md) page covers
the request types (`ETagRequestOutcome`, `FTagRequestHandle`, `FTagRequestResponderHandle`).

---

← [Reference](README.md) · [Documentation index](../README.md)
