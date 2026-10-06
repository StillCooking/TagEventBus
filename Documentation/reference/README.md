# Reference

*The full API surface organized by task — binding listeners, broadcasting, and reading state, each on its own page.*

The split follows **what you do**, not which language you write it in. Each card shows the C++ call and, where available, the Blueprint node name — [Native C++ API](cpp-api.md) is the one page where there is no Blueprint side at all.

## How to read a card

**The availability line** says where you can call the function from. "Available in: C++ and
Blueprint" means a Blueprint node exists; "Available in: C++" means the function is C++ only.

**`WorldContextObject`** is a hidden pin in Blueprint, filled in by `self`. In C++ you pass
it yourself — it is the `this` standing first in the calls that take it.

**The parameters the cards share** are listed here once, and the cards no longer repeat them:

| Parameter | Meaning | Default |
| --- | --- | --- |
| `WorldContextObject` | any object with a world context; the hidden Blueprint pin, `this` in C++ | — |
| `Scope` | which registry: the global one or the actor's local one | `Global` |
| `Match` | `Exact` matches the tag alone; `Partial` matches its descendants too | `Exact` |
| `bOnce` | the listener disappears after the first delivery | `false` |
| `bReplaySticky` | deliver the retained value immediately on registration | `false` |
| `bCreateLocalIfMissing` | create the local bus component if there is none | `false` |
| `Priority` | higher means earlier **within the channel**; ties preserve registration order | `0` |
| `ThrottleSeconds` | at most one delivery per window, on the leading edge, measured in real time | `0` |
| `Query` | a filter evaluated against the event tag and the broadcast's `ContextTags` | empty |

The [Broadcast path](../concepts/broadcast-path.md) page explains how these parameters affect delivery. The table above summarizes their meanings and defaults.

## What is not here

The internal delegate handlers (`HandleTagEvent`, `HandleOwnerEndPlay`,
`HandleContextEndPlay`) have no cards. They are not entry points: you do not call them from
outside, and they are not exposed to Blueprint.

The APIs of the two optional modules are documented under their respective topics:
[Contracts](../advanced/contracts.md) and
[Request-Response](../advanced/request-response.md).

## Pages in this section

| Page | What it covers |
| --- | --- |
| [Bind and Unbind](bind-unbind.md) | Listener binding and unbinding, pausing, groups, and the async node — the full listener-side API. |
| [Broadcast](broadcast.md) | All broadcast variants, including deferred delivery, and consuming an event inside a handler. |
| [Bus state](bus-state.md) | What you can ask the bus before broadcasting, how to access the bus object itself, and how to dump sticky state into a save game. |
| [Types and settings](types.md) | Enums, result and save state structs, and project settings. |
| [Native C++ API](cpp-api.md) | The subsystem and component API available only in C++: templated `AddListener<T>` and `Broadcast<T>`, `FTagEventContext`, the RAII subscription helpers, and the fluent builder. |

---

← [Documentation index](../README.md)
