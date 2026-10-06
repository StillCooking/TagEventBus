# Bus state

*What you can ask the bus before broadcasting, how to access the bus object itself, and how to dump sticky state into a save game.*

Nothing on this page [broadcasts](broadcast.md) or [binds](bind-unbind.md). These functions query the bus, capture retained
state into a save struct, or restore it. Restoring retained state replaces the bus's current
retained values. The [reference introduction](README.md) covers `WorldContextObject` and
`Scope`.

## Reading state

The listener queries come in two versions. There is one difference, and it matters:

| Version | What it counts |
| --- | --- |
| `Has Listeners`, `Get Listener Count` | **the exact channel only**; `Partial` listeners on ancestors are skipped |
| `Has Matching Listeners`, `Get Matching Listener Count` | what a broadcast would actually consider: the exact channel plus `Partial` on ancestors |

> [!warning]
> To check whether any listeners match the event tag, use the `Matching` version. The version without
> `Matching` answers a narrower question and, where tags form a hierarchy, can return
> `false` where a broadcast would still deliver something.

Both versions **count paused listeners** and do not filter by payload type — they answer a
question about the route, not about the outcome.

The library exposes `Has Listeners`, `Has Matching Listeners`, and `Get Matching Listener Count`
with a `Scope` pin. `Get Listener Count` exists only on the bus object itself — see
[The global bus](#the-global-bus) below.

### `HasListeners(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::HasListeners(this, Tag)`
- **Blueprint node** — `Has Listeners`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |

**Returns:** `bool` — whether at least one listener is registered on that exact tag

### `HasMatchingListeners(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::HasMatchingListeners(this, Tag)`
- **Blueprint node** — `Has Matching Listeners`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |

**Returns:** `bool` — whether a broadcast on that tag would consider at least one listener

### `GetMatchingListenerCount(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::GetMatchingListenerCount(this, Tag)`
- **Blueprint node** — `Get Matching Listener Count`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |

**Returns:** `int32` — the number of listeners a broadcast would consider

## The global bus

### `Get(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusSubsystem::Get(this)`
- **Blueprint node** — `Get`

**Returns:** `UTagEventBusSubsystem*` — the global bus; `nullptr` during early startup and during teardown

### `GetChannelCount()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Bus->GetChannelCount()`
- **Blueprint node** — `Get Channel Count`

**Parameters:** none.

**Returns:** `int32` — the number of allocated channels: tags with listeners or with a retained value

A value read during a broadcast can briefly include channels awaiting removal — a
consequence of deferred removal, not a leak.

### `GetActiveTags()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Bus->GetActiveTags()`
- **Blueprint node** — `Get Active Tags`

**Parameters:** none.

**Returns:** `TArray<FGameplayTag>` — every tag with an active channel

The global bus also exposes its own `Has Listeners`, `Get Listener Count`,
`Has Matching Listeners`, and `Get Matching Listener Count` — with no `Scope` pin, because
the bus object already determines it. The meaning is identical to that of the library
variants.

## The local bus

`UTagEventBusComponent` exposes the same four queries as the global bus, each limited to **that** local bus. It also provides functions for saving and restoring state.

## Saving sticky state

### `CaptureRetainedState(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `Bus->CaptureRetainedState(SaveState)`
- **Blueprint node** — `Capture Retained State`

| Parameter | Meaning |
| --- | --- |
| `Out` | the save struct, filled in by the call |

**Returns:** nothing.

Dumps the retained values into a struct suitable for a save game. **Listeners are not
saved** — after a load you have to bind them again.

The struct is meant for **writing to disk, not for sending over the network**: it offers no <!-- fact:ftageventbussavestate-jest-przeznaczony-do-zapisu-na-dysk-ni -->
versioning guarantees between machines and no conflict resolution, so it is not a way to
replicate sticky values.

### `RestoreRetainedState(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `Bus->RestoreRetainedState(SaveState)`
- **Blueprint node** — `Restore Retained State`

| Parameter | Meaning |
| --- | --- |
| `In` | a previously captured save struct |

**Returns:** `ETagEventRestoreResult` — the outcome of the restore

Restores the retained values, replacing the current ones. Older dumps are migrated forward.

> [!warning]
> **A snapshot with a schema version newer than this build supports is rejected outright** — nothing is restored,
> the registry is left untouched, and the rejection is logged as an `Error`. This is not a
> partial load; it is a refusal. You will see this if you roll the plugin back and try to restore a save whose version that build does not support.

Restoring an entry with a live TTL on a local bus re-arms the component's lazy tick.

---

← [Reference](README.md) · [Documentation index](../README.md)
