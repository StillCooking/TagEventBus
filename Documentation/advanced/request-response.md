# Request-Response

*A request and a response on the same bus: four outcomes, an immediate or a delayed answer, a timeout, and cancellation.*

The `TagEventBusRequests` module **does not change the core at all**. It is built entirely
on the public API: [a broadcast under the `Stop After Handled` policy](../reference/broadcast.md), plus consuming the <!-- fact:warstwa-tageventbusrequests-nie-zmienia-rdzenia-w-ogole-jest -->
event. A request is an ordinary event, and a response is that event being consumed.

The layer lives in `UTagEventRequestSubsystem` and **needs no startup configuration** — it
works right away.

## Four outcomes

| Outcome | When |
| --- | --- |
| `Completed` | a responder answered |
| `NoResponder` | the broadcast reached **zero** listeners |
| `TimedOut` | at least one listener received the request, but none answered within the time window |
| `Cancelled` | the request was canceled before an answer arrived |

Exactly one of these outcomes reaches the requester's callback unless the owner dies first.

> [!warning]
> **`NoResponder` and `TimedOut` are two different problems.** `NoResponder` is settled
> immediately on send, by a delivery count of zero — fail-fast, with no waiting. `TimedOut`
> requires at least one listener to have received the request; the timeout countdown starts only then.
> <!-- fact:noresponder-rozni-sie-od-timedout-noresponder-rozstrzyga-sie -->
> `NoResponder` means "no listener received the request". `TimedOut` means "at least one <!-- fact:cztery-wyniki-zapytania-completed-respondent-odpowiedzial-no -->
> listener received it, but nobody answered before the timeout expired".

## Sending a request

### `RequestTagEvent(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UAsyncAction_RequestTagEvent::RequestTagEvent(this, Tag, Payload, this, 5.f)`
- **Blueprint node** — `Request Tag Event`

| Parameter | Meaning |
| --- | --- |
| `WorldContextObject` | the world context object |
| `Tag` | the request tag |
| `Payload` | optional; empty means the signal alone |
| `Sender` | the object reported as the requester |
| `TimeoutSeconds` | the time window; `5` by default, `<= 0` disables the timeout entirely |

**Returns:** `UAsyncAction_RequestTagEvent*` — the action, with four output pins

A latent node with four output pins, one per outcome: `On Completed`,
`On Timed Out`, `On No Responder`, and `On Cancelled`.

`TimeoutSeconds <= 0` disables the timeout. If no listener receives the request, it still
completes immediately with `NoResponder`. Otherwise, it waits until someone answers, it is <!-- fact:w-warstwie-request-response-timeout-0-wylacza-timeout-calkow -->
canceled, or its owner dies. **The owner's death drops the request silently**.

> [!tip] Blueprint graph to copy
> [`tageventbus-request-response.txt`](../Assets/tageventbus-request-response.txt) — a sample graph to read: the request node with all four outputs wired. It calls helper functions of the actor it was built in, so after pasting it into your own Blueprint, replace those calls with your own logic.
>
> The four request outputs. NoResponder and TimedOut lead to different diagnoses.

## Answering

You answer **from inside the listener body**, the same way you consume an event.

### `RespondToCurrentRequest(...)`

*Available in: Blueprint. In C++ use the typed call below — the node itself cannot be called from C++.*

- **Blueprint node** — `Respond To Current Request`
- **C++ equivalent** — `UTagEventRequestSubsystem::Get(this)->RespondToCurrent(Payload, this)`

| Parameter | Meaning |
| --- | --- |
| `WorldContextObject` | the world context object |
| `Responder` | the object reported as the responder |
| `Payload` | any struct, a wildcard pin |

**Returns:** nothing.

Answers the request currently being delivered. Outside a request dispatch it is a no-op and
logs a `Warning`.

### `RespondToCurrentRequestSignal(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventRequestLibrary::RespondToCurrentRequestSignal(this, this)`
- **Blueprint node** — `Respond To Current Request Signal`

| Parameter | Meaning |
| --- | --- |
| `WorldContextObject` | the world context object |
| `Responder` | the responding object |

**Returns:** nothing.

An answer with no payload — the bare "handled" signal.

## An answer that takes time

If the answer has to wait — for an animation, for the network, for a `Delay` node — you
take a ticket first.

### `GetCurrentRequestResponder(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventRequestLibrary::GetCurrentRequestResponder(this)`
- **Blueprint node** — `Get Current Request Responder`

| Parameter | Meaning |
| --- | --- |
| `WorldContextObject` | the world context object |

**Returns:** `FTagRequestResponderHandle` — the ticket you answer with; invalid outside dispatch

Returns the ticket you will use to answer later. **Call it before you start waiting** — calling it after a `Delay` returns an invalid ticket because the dispatch is already over.

### `RespondWithHandle(...)`

*Available in: Blueprint. In C++ use the typed call below — the node itself cannot be called from C++.*

- **Blueprint node** — `Respond With Handle`
- **C++ equivalent** — `Handle.Respond(Payload, this)`, on the `FTagRequestResponderHandle` from `GetCurrentResponder()`

| Parameter | Meaning |
| --- | --- |
| `WorldContextObject` | the world context object |
| `Handle` | the ticket taken earlier |
| `Responder` | the responding object |
| `Payload` | any struct, a wildcard pin |

**Returns:** nothing.

Answers the request whose ticket was taken earlier. It is a no-op if the request has already
finished — the timeout has expired, for example, or somebody else answered first.

## Cancellation

### `CancelTagRequest(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventRequestLibrary::CancelTagRequest(this, Handle)`
- **Blueprint node** — `Cancel Tag Request`

| Parameter | Meaning |
| --- | --- |
| `WorldContextObject` | the world context object |
| `Handle` | the request handle |

**Returns:** `bool` — `false` if the request has already finished

Cancels a request in flight; its outcome becomes `Cancelled`.

## An answer that outruns the handle

> [!warning]
> The entry in the correlation map is created **before** the broadcast. If a responder answers <!-- fact:wpis-w-mapie-korelacji-zapytania-powstaje-przed-broadcastem -->
> synchronously, the requester's callback fires **before the `Request` call returns the
> handle**.
>
> Code that stores the returned handle in a field cannot rely on that field containing the new handle when the callback fires. Do not assume the callback runs only after the request
> call has returned.

---

← [Advanced](README.md) · [Documentation index](../README.md)
