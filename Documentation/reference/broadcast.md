# Broadcast

*All broadcast variants, including deferred delivery, and consuming an event inside a handler.*

The [reference introduction](README.md) covers `WorldContextObject` and `Scope`. The parameters
shared by the broadcast calls are these:

| Parameter | Meaning | Default |
| --- | --- | --- |
| `Sender` | the object reported as the sender; visible in the debugger panel | — |
| `Payload` | optional; an empty `FInstancedStruct` means "the signal alone" | empty |
| `bRetain` | retain the value in the channel (sticky) | `false` |
| `RetainTTLSeconds` | above `0`, clears the sticky value after that many seconds, counted from **delivery** rather than from enqueue time | `0` |
| `ContextTags` | additional tags evaluated together with the event tag by a listener's `Query` | empty |

## Broadcasting

### `BroadcastTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BroadcastTagEvent(this, Tag, this, Payload)`
- **Blueprint node** — `Broadcast Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |

**Returns:** `int32` — the number of deliveries

The basic broadcast. The returned delivery count is the simplest way to check whether any listeners received the event.

It is equivalent to a call with the `Broadcast All` policy.

### `TryBroadcastTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::TryBroadcastTagEvent(this, Tag, this, Payload, Scope, bRetain, TTL, ContextTags, Result)`
- **Blueprint node** — `Try Broadcast Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Result` | the operation result, including `Success`; it drives the exec pins in Blueprint |

**Returns:** `int32` — the number of deliveries

This variant reports whether the operation succeeded and, if it failed, why. The delivery count still comes out on the return
pin.

### `BroadcastTagEventWithPolicy(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BroadcastTagEventWithPolicy(this, Tag, this, Payload, ETagEventBroadcastPolicy::StopAfterHandled)`
- **Blueprint node** — `Broadcast Tag Event (Policy)`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Policy` | `Broadcast All` or `Stop After Handled` |

**Returns:** `FTagEventBroadcastResult` — the delivery count, plus whether any listener handled the event and whether any listener stopped it

Broadcasts under an explicit policy and **tells you what became of the event**: whether any
listener marked it handled, and whether any listener stopped propagation.

This is the variant you need when the sender cares whether someone handled the event, rather than just whether it was broadcast.

### `BroadcastTagEvents(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BroadcastTagEvents(this, Tags, this, Payload)`
- **Blueprint node** — `Broadcast Tag Events`

| Parameter | Meaning |
| --- | --- |
| `Tags` | the tag container |
| `bRetain` | retain the value in every channel |

**Returns:** `int32` — the total number of deliveries across all tags

Broadcasts the same event on every tag in the container.

### `BroadcastTagEventStruct(...)`

*Available in: Blueprint. In C++ use the typed call below — the node itself cannot be called from C++.*

- **Blueprint node** — `Broadcast Tag Event Struct`
- **C++ equivalent** — `Bus->Broadcast(Tag, this, Payload)` on `UTagEventBusSubsystem` (Global) or `UTagEventBusComponent` (Local), with any `USTRUCT` as `Payload`; see the [C++ API](cpp-api.md)

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `Payload` | any struct, a wildcard pin |

**Returns:** `int32` — the number of deliveries

Carries any struct through a wildcard pin **without wrapping it in an `FInstancedStruct` at the call site**. This is the variant for [hot paths](../advanced/optimization.md) where the cost of that wrapper matters. A Blueprint listener on the channel still gets a copy — [the payload costs](../advanced/optimization.md#payload) list every path.

## Deferred broadcast

### `BroadcastTagEventDeferred(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::BroadcastTagEventDeferred(this, Tag, this, Payload)`
- **Blueprint node** — `Broadcast Tag Event Deferred`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `bCoalesce` | merge with a pending entry on the same tag only if that entry also has `bCoalesce` enabled; keep the newest values |

**Returns:** nothing.

Queues the broadcast for delivery on the next tick and copies the payload on enqueue. Deferring delivery avoids an immediate recursive call, but repeated deferred broadcasts still need an exit condition.

The node does not report the delivery outcome or return a delivery count.

In `Local` scope it needs an existing bus component and **will not create one itself**.

### `TryBroadcastTagEventDeferred(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::TryBroadcastTagEventDeferred(this, Tag, this, Payload, Scope, bRetain, TTL, bCoalesce, ContextTags, Result)`
- **Blueprint node** — `Try Broadcast Tag Event Deferred`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `bCoalesce` | merge with a pending entry on the same tag only if that entry also has `bCoalesce` enabled; keep the newest values |
| `Result` | the enqueue result, including `Success`; **not the delivery result** |

**Returns:** nothing.

`Result` describes the outcome of enqueueing, not delivery. Delivery happens on the next tick, and its outcome is not reported.

### `BroadcastTagEventStructDeferred(...)`

*Available in: Blueprint. In C++ use the typed call below — the node itself cannot be called from C++.*

- **Blueprint node** — `Broadcast Tag Event Struct Deferred`
- **C++ equivalent** — `Bus->BroadcastDeferred(Tag, this, Payload)`; for `bCoalesce`, the fluent builder: `Bus->Emit(Tag).Payload(Payload).Coalesce().Deferred()`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the channel tag |
| `bCoalesce` | merge with a pending entry on the same tag only if that entry also has `bCoalesce` enabled; keep the newest values |
| `Payload` | any struct, a wildcard pin |

**Returns:** nothing.

The deferred wildcard variant. The payload is copied into the queue and flushed on the next
tick.

## Consuming an event inside a handler

### `ConsumeTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::ConsumeTagEvent(ETagEventHandling::StopPropagation)`
- **Blueprint node** — `Consume Tag Event`

| Parameter | Meaning |
| --- | --- |
| `Handling` | `Handled` or `Stop Propagation` |

**Returns:** nothing.

Call it **from inside the listener body**. `Handled` marks the event as handled and ends
delivery only under the `Stop After Handled` policy; `Stop Propagation` always ends
delivery.

Outside dispatch — after a `Delay` node, for example — it is a no-op and logs a `Warning`.

It operates on the deepest active frame, no matter which bus that frame belongs to: there
is one frame stack, shared by the global bus and every local one.

### `GetCurrentEventContextTags()`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventBusLibrary::GetCurrentEventContextTags()`
- **Blueprint node** — `Get Current Event Context Tags`

**Parameters:** none.

**Returns:** `FGameplayTagContainer` — the context tags of the event currently being delivered

Call it from inside the listener body. Outside dispatch it returns an empty container and
logs a `Warning`.

On a sticky replay, it returns the context tags from the **retained** broadcast, not from the time of replay.

---

← [Reference](README.md) · [Documentation index](../README.md)
