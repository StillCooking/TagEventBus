# TagEventBus documentation

*An event bus for Unreal Engine 5 that uses Gameplay Tags as addresses — the sender and the receiver hold no references to each other.*

The sender broadcasts an event on a tag. The receiver listens on the same tag. Neither
side knows the other, and neither holds a pointer to the other.

## Find the right API for your task

| I need                                                                   | Use | Where |
|--------------------------------------------------------------------------| --- | --- |
| A widget to respond to an event from the world                           | `Bind Tag Event` + `Broadcast Tag Event` | [Quick start](start/quick-start.md) |
| A listener to be removed automatically when its actor is destroyed       | `Bind Tag Event (Managed)` | [Lifecycle](concepts/lifecycle.md) |
| One object to listen on a dozen or more tags                             | listener group | [Bind and Unbind](reference/bind-unbind.md) |
| A receiver to get a value that was broadcast before the receiver existed | sticky (`bRetain`) | [Broadcast](reference/broadcast.md) |
| To see who broadcasts an event and who receives it                       | debugger panel | [Debugger](concepts/debugger.md) |
| The payload type to be enforced                                          | tag → struct contracts | [Contracts](advanced/contracts.md) |
| An answer, not just a notification                                       | request-response | [Request-Response](advanced/request-response.md) |

## Limits worth knowing before your first line of code

- **Game Thread only.** The registry is not thread-safe. You bind and broadcast from the
  Game Thread.
- **No replication.** The bus is local to the game instance. An event broadcast on a client
  does not appear on the server.
- **Every tag has to exist in the project's Gameplay Tag list.** A tag outside that list is
  rejected, both on broadcast and on bind — the Blueprint nodes log a `Warning`; the low-level C++ API stays silent.
- **The diagnostic layer is excluded from Shipping builds.** The debugger panel, the history, and
  the statistics live behind `#if !UE_BUILD_SHIPPING`. Production logic runs unchanged.

The full set of limits:
[Limits and threading](advanced/limits-and-threading.md).

## Glossary

**Global bus** — `UTagEventBusSubsystem` on the `GameInstance`. It is created automatically and works as soon as the plugin is enabled, with no configuration.

**Local bus** — `UTagEventBusComponent` on an actor. A separate registry, not a filter over
the global one. It is created only when you explicitly add the component or enable `bCreateLocalIfMissing` when binding.

**Listener** — a single subscription: an object, a tag, and a handler, tied together by one
`Bind` call.

**Broadcast** — sending an event on a tag. The sender does not call the receiver directly and
finishes its work whether or not anyone is listening.

**Channel** — the set of listeners registered on one specific tag.

**`Partial` match** — the listener also matches child tags. Without it, a listener on
`Combat` will not see `Combat.Hit`.

**Sticky** — a broadcast with `bRetain`, retained in the channel. A new listener with
`bReplaySticky` gets the last retained value immediately after it registers.

**Handle** — the `FTagEventHandle` returned when you bind, and the only way to unbind one
specific listener.

## Documentation sections

| Page | What it covers |
| --- | --- |
| [Start here](start/README.md) | Installation, the one mandatory configuration step, and a check that the plugin really did build. |
| [Concepts](concepts/README.md) | The conceptual model of the bus: two disjoint registries, a fixed delivery path, and cleanup responsibilities and timing. |
| [Reference](reference/README.md) | The full API surface organized by task — binding listeners, broadcasting, and reading state, each on its own page. |
| [Advanced](advanced/README.md) | The two optional modules, the cost of each path, how the bus works alongside other systems, and its limitations. |
| [Troubleshooting](troubleshooting/README.md) | The order in which you use the tools here is the reverse of the usual debugging workflow: the panel first, then the logs, and the C++ debugger last. |
| [Versions, support, and license](project/README.md) | What is tested, what an update looks like, what is planned, and the terms of use. |

Release-by-release history: [`CHANGELOG.md`](../CHANGELOG.md) in the plugin root.
