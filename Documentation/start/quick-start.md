# Quick start

*From an empty project to an actor sending an event to a widget — without a single reference between them.*

## What you need

- Unreal Engine **5.5 – 5.8**, Win64. The plugin is built and tested separately for each of those four engine versions.
- **A Blueprint-only or C++ project** — which of the two depends on
  [how you install the plugin](installation.md#requirements).
- The engine's **Gameplay Tags** plugin, enabled in the Plugins browser.

## Installation in one step

**From Fab:** install the plugin to your engine from the Epic Games Launcher, enable it in
**Edit → Plugins** and restart the editor.

**Manually from source:** copy the plugin directory into `Plugins/TagEventBus/` in your
project, with the `.uplugin` file sitting directly in that directory, then build the project.

Full installation instructions, including the `Build.cs` dependencies:
[Installation](installation.md).

## Your first event

1. Register the `Combat.Hit` tag in [**Project Settings → Project → GameplayTags**](configuration.md). A tag <!-- fact:kazdy-tag-musi-istniec-w-globalnym-slowniku-tagow-projektu-t -->
   outside the project's tag list is rejected, and the Blueprint node logs a `Warning`.
   Without this step, the event will not be delivered.
2. In the widget Blueprint, add a Custom Event named `OnHit` with a `Payload` pin of type
   `Instanced Struct`.
3. In the widget's `Event Construct`, call **Bind Tag Event**: `Tag` = `Combat.Hit`,
   `Event` = `OnHit`. Leave the remaining pins at their defaults — `Global` scope, `Exact`
   match.
4. In any actor Blueprint, call **Broadcast Tag Event**: `Tag` = `Combat.Hit`,
   `Sender` = `self`, `Payload` empty.
5. Start Play In Editor (PIE) and trigger the actor's action.

The widget's `OnHit` handler is called. You never told it who broadcasts, and the actor has no idea
the widget exists.

> [!tip] Blueprint graph to copy
> [`tageventbus-quickstart-bind.txt`](../Assets/tageventbus-quickstart-bind.txt) — open the file, copy its entire contents and paste them into any Blueprint graph.
>
> Listening inside the widget. The widget only needs the tag name to listen for the event.

> [!tip] Blueprint graph to copy
> [`tageventbus-quickstart-broadcast.txt`](../Assets/tageventbus-quickstart-broadcast.txt) — open the file, copy its entire contents and paste them into any Blueprint graph.
>
> The broadcast. The returned number tells you how many receivers got the event.

## How you know it works

**Broadcast Tag Event returns the number of deliveries.** Wire that number into a Print
String node. With the widget as the only listener, `1` means it received the event. `0` means nobody did, and that is
the moment to check whether the tag really is in the project's tag list.

**The debugger panel shows the same thing with no changes to the graph.** Open
**Tools → Debug → TagEventBus Debug** and go to the Overview tab. After a broadcast, the
tag shows up in the list with its `Broadcasts` and `Delivered` counters. A high
`Broadcasts` count next to `Delivered = 0` means you are broadcasting into the void.

## What next

- [Architecture](../concepts/architecture.md) — why the local bus and
  the global bus are two disjoint registries rather than one registry with a filter.
- [Bind and Unbind](../reference/bind-unbind.md) — the full list of
  binding parameters, including `Match` and `Priority`.
- [Debugger](../concepts/debugger.md) — the Inject tab, which lets
  you test a receiver before the sender even exists.

---

← [Start here](README.md) · [Documentation index](../README.md)
