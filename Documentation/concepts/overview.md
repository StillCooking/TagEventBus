# Overview

*What the plugin is made of, which modules are optional, and what sets it apart from free tag-routing alternatives.*

## The four modules

| Module | Type | Role | Needs configuration |
| --- | --- | --- | --- |
| `TagEventBus` | Runtime | registry, binding, broadcasting, sticky values, deferred delivery | no |
| `TagEventBusEditor` | Editor | the debugger panel | no |
| `TagEventBusRequests` | Runtime | request-response | no |
| `TagEventBusContracts` | Runtime | tag → struct contracts | yes, schema registration |

`TagEventBusRequests` and `TagEventBusContracts` are optional in the sense that **consumer code does not have to reference them in `Build.cs`**.
With the plugin enabled, the runtime modules load both in the editor and in a packaged game.
The editor modules load only in the editor. The core behaves identically whether or not you use <!-- fact:modul-opcjonalny-znaczy-ze-kod-konsumenta-nie-musi-go-refere -->
the optional modules.

The optional modules add no overhead to the hot delivery path while their functionality is
inactive. With `EnforcementLevel = Off`, the contract validator hook is not installed.

The `TagEventBusRequests` layer does not change the core at all: it is built entirely on the
public API. A request is an ordinary event, and a response is that event being consumed.

## What you get beyond tag routing

The bare "broadcast on a tag, listen on a tag" mechanism is already covered by the free
`GameplayMessageSubsystem` and by a few approaches in Lyra. This plugin adds four things on
top of it:

- **Visibility.** The in-editor panel shows who broadcasts, on which tag, and how many
  receivers the event reaches, together with counters. A high `Broadcasts` count next to `Delivered = 0` reveals an object broadcasting into the void, without the need for a breakpoint.
- **Payload type contracts.** A tag can have a struct assigned to it. With contract validation
  enabled in a non-Shipping build, a payload type mismatch is detected when the event is
  broadcast.
- **Request-response.** A request with a timeout, cancellation, and a separate path for the
  "nobody answered" case.
- **Sticky values with state saving.** A value retained in the channel, replayed immediately
  after registration to listeners that enable `bReplaySticky`, and carried over by
  Capture/Restore.

## When this is not what you are looking for

If all you need is basic broadcasting and listening on a tag, and figuring out what is going on
gives you no trouble, the free equivalents cover it. The plugin starts paying off once there
are so many events that reading the code no longer keeps you on top of them.

Three hard limits hold whatever the size of the project, and no plan changes them.
[Limits and threading](../advanced/limits-and-threading.md) lists them, and keeps them apart
from what is missing only for now.

---

← [Concepts](README.md) · [Documentation index](../README.md)
