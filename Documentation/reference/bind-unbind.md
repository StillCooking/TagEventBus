# Bind and Unbind

*Listener binding and unbinding, pausing, groups, and the async node — the full listener-side API.*

The [reference introduction](README.md) covers what the shared
parameters mean.

## Binding

### `BindTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BindTagEvent(this, Tag, Event)`
- **Blueprint node** — `Bind Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag; it has to exist in the project's tag list |
| `Event` | the Custom Event to call |

**Returns:** `FTagEventHandle` — the handle you unbind with

Binds a Custom Event to a tag, much like an Event Dispatcher. It returns the handle
you later use to unbind that one listener.

A tag outside the project's tag list is rejected — the Blueprint node logs a `Warning`.

### `TryBindTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::TryBindTagEvent(this, Tag, Event, Scope, Match, bOnce, bReplaySticky, bCreate, Priority, Throttle, Query, Result)`
- **Blueprint node** — `Try Bind Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Event` | the Custom Event |
| `Result` | the operation result, including `Success`; it drives the exec pins in Blueprint |

**Returns:** `FTagEventHandle` — the handle you unbind with

The same as `Bind Tag Event`, but it tells you **why** it failed. Use it while integrating;
once everything works, go back to the plain variant.

### `BindTagEventManaged(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BindTagEventManaged(this, Tag, Event)`
- **Blueprint node** — `Bind Tag Event (Managed)`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Event` | the Custom Event |

**Returns:** `UTagEventBinding*` — the binding object; `nullptr` on invalid input

Returns a binding object instead of a raw handle. It unbinds itself on the owning actor's
`EndPlay`, and it lets you unbind explicitly at any time.

This is the default choice wherever an actor or a widget owns the listener.

### `BindTagEvents(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BindTagEvents(this, Tags, Event)`
- **Blueprint node** — `Bind Tag Events`

| Parameter | Meaning |
| --- | --- |
| `Tags` | the tag container |
| `Event` | the Custom Event bound to each of the tags |

**Returns:** `int32` — the number of tags bound

Binds the same Custom Event to every tag in the container.

Returns `0` **without binding anything** when the registry cannot be resolved — for example
in `Local` scope, when the actor has no bus component and `bCreateLocalIfMissing` is
`false`. In that case it logs a `Warning` with the actor's name and a hint on how to fix it.

## Unbinding

### `Unbind(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::Unbind(this, Handle)`
- **Blueprint node** — `Unbind`

| Parameter | Meaning |
| --- | --- |
| `Handle` | the handle returned when you bound the listener |

**Returns:** `bool` — `false` if the listener was already removed

Unbinds a single listener by handle.

### `TryUnbind(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::TryUnbind(this, Handle, Scope, Result)`
- **Blueprint node** — `Try Unbind`

| Parameter | Meaning |
| --- | --- |
| `Handle` | the handle returned when you bound the listener |
| `Result` | the operation result, including `Success` |

**Returns:** `bool` — `false` if the listener was already removed

This variant reports the operation result. `InvalidHandle` covers both a handle with `Id = 0` and a handle to a listener that has already been removed — the two cases are not distinguished.

### `UnbindTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::UnbindTagEvent(this, Tag, Event)`
- **Blueprint node** — `Unbind Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Event` | **the same** Custom Event that was bound |

**Returns:** `int32` — the number of listeners removed

Unbinds by delegate, much like an Event Dispatcher — with no handle to keep. It
requires you to pass exactly the same Event that was bound.

### `TryUnbindTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::TryUnbindTagEvent(this, Tag, Event, Scope, Result)`
- **Blueprint node** — `Try Unbind Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Event` | the same Custom Event |
| `Result` | the operation result, including `Success` |

**Returns:** `int32` — the number of listeners removed

The variant that gives a reason. **Removing zero listeners is still `Success`** — it only means no matching listeners were bound.

### `UnbindObjectFromTag(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::UnbindObjectFromTag(this, Tag, Owner)`
- **Blueprint node** — `Unbind Object From Tag`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Owner` | the object whose listeners are to be removed |

**Returns:** `int32` — the number of listeners removed

Unbinds every listener owned by `Owner` on that tag, regardless of which Event was bound.

### `UnbindObjectFromTags(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::UnbindObjectFromTags(this, Tags, Owner)`
- **Blueprint node** — `Unbind Object From Tags`

| Parameter | Meaning |
| --- | --- |
| `Tags` | the tag container |
| `Owner` | the object whose listeners are to be removed |

**Returns:** `int32` — the total number of listeners removed

The same, for a whole tag container.

### `UnbindTagEvents(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::UnbindTagEvents(this, Tags, Event)`
- **Blueprint node** — `Unbind Tag Events`

| Parameter | Meaning |
| --- | --- |
| `Tags` | the tag container |
| `Event` | the same Custom Event |

**Returns:** `int32` — the total number of listeners removed

Unbinds the Event from every tag in the container.

### `UnbindAllForObject(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::UnbindAllForObject(this, this)`
- **Blueprint node** — `Unbind All For Object`

| Parameter | Meaning |
| --- | --- |
| `Owner` | the object whose listeners are to be removed; defaults to `self` |

**Returns:** `int32` — the number of listeners removed

Unbinds **every** listener owned by `Owner` across all tags in the selected registry. This is the call for `EndPlay`
and for teardown.

When the post-GC sweep removes a listener, it logs a `Warning` because the listener was still registered when its owner was collected. Unbind during teardown to avoid relying on this fallback.

## Pause

### `PauseListener(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::PauseListener(this, Handle)`
- **Blueprint node** — `Pause Listener`

| Parameter | Meaning |
| --- | --- |
| `Handle` | the listener's handle |

**Returns:** `bool` — `false` for an unknown or removed handle, an unresolved scope, or a listener that is already paused

Suspends delivery. Broadcasts skip the listener, and `bOnce` is preserved.

Events sent while the listener is paused are **not buffered for that listener**.

### `ResumeListener(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::ResumeListener(this, Handle, true)`
- **Blueprint node** — `Resume Listener`

| Parameter | Meaning |
| --- | --- |
| `Handle` | the listener's handle |
| `bReplaySticky` | deliver the tag's retained value immediately on resume |

**Returns:** `bool` — `false` for an unknown or removed handle, an unresolved scope, or a listener that was not paused

Resumes delivery.

## Managed binding

### `Unbind()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Binding->Unbind()`
- **Blueprint node** — `Unbind`

**Parameters:** none.

**Returns:** nothing.

Unbinds the managed listener immediately. You can call it repeatedly; once the bus is gone
it is a no-op.

### `IsActive()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Binding->IsActive()`
- **Blueprint node** — `Is Active`

**Parameters:** none.

**Returns:** `bool` — `true` while the listener is bound and its bus is alive

## Listener groups

### `CreateListenerGroup(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::CreateListenerGroup(this, this)`
- **Blueprint node** — `Create Listener Group`

| Parameter | Meaning |
| --- | --- |
| `Owner` | the group's owner, `self` by default; it drives the automatic unbind on `EndPlay` and may be left empty |

**Returns:** `UTagEventListenerGroup*` — the group; `nullptr` when the registry cannot be resolved

Creates a group on the given bus. A group lets you unbind or pause all its bindings with a single call.

### `BindTagEventToGroup(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BindTagEventToGroup(Group, Tag, Event)`
- **Blueprint node** — `Bind Tag Event To Group`

| Parameter | Meaning |
| --- | --- |
| `Group` | the target group |
| `Tag` | the channel tag |
| `Event` | the Custom Event |

**Returns:** `FTagEventHandle` — the handle; invalid on bad input or a dead group

Binds the listener and adds it to the group in one step. **If the group is paused, the
new listener starts out paused.**

### `BindTagEventsToGroup(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BindTagEventsToGroup(Group, Tags, Event)`
- **Blueprint node** — `Bind Tag Events To Group`

| Parameter | Meaning |
| --- | --- |
| `Group` | the target group |
| `Tags` | the tag container |
| `Event` | the Custom Event |

**Returns:** `int32` — the number of tags bound

One shared set of options for the whole batch.

### `UnbindAll()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Group->UnbindAll()`
- **Blueprint node** — `Unbind All`

**Parameters:** none.

**Returns:** `int32` — the number of listeners removed

Unbinds every member and clears the group's pause flag. The group stays usable — a binding
made after this call starts unpaused.

### `Pause()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Group->Pause()`
- **Blueprint node** — `Pause`

**Parameters:** none.

**Returns:** nothing.

Suspends delivery to every member. New bindings added to the group start paused and stay paused until you call `Resume`.

### `Resume(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `Group->Resume(true)`
- **Blueprint node** — `Resume`

| Parameter | Meaning |
| --- | --- |
| `bReplaySticky` | also deliver the retained value of every member's tag |

**Returns:** nothing.

Resumes delivery to every member. **It overrides pauses set on individual members** —
after this call no member is left paused.

### `IsPaused()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Group->IsPaused()`
- **Blueprint node** — `Is Paused`

**Parameters:** none.

**Returns:** `bool` — the group's pause flag

The flag set by `Pause` and `Resume`. **It is not computed from the members' state**, so an
unpaused group can still have individual members paused.

### `Num()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Group->Num()`
- **Blueprint node** — `Num`

**Parameters:** none.

**Returns:** `int32` — the number of handles held

It also counts stale entries left by `bOnce` listeners after their first delivery — those entries are removed only when you call `Unbind All`.

### `IsActive()`

*Available in: C++ and Blueprint.*

- **C++ call** — `Group->IsActive()`
- **Blueprint node** — `Is Active`

**Parameters:** none.

**Returns:** `bool` — `true` while the group's bus is alive

## The async node

### `ListenForTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UAsyncAction_ListenForTagEvent::ListenForTagEvent(this, Tag)`
- **Blueprint node** — `Listen For Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |

**Returns:** `UAsyncAction_ListenForTagEvent*` — the action, which can be canceled

A latent node for Blueprints: the output pin fires on every delivery. It returns the action,
so the listener can be canceled.

In `Local` scope the bus component is created **only** when `bCreateLocalIfMissing` is
`true`.

---

← [Reference](README.md) · [Documentation index](../README.md)
