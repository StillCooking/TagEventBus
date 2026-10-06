<img src="Resources/Icon128.png" alt="" width="96" align="right">

# TagEventBus

Lightweight `FGameplayTag`-addressed event bus for Unreal Engine. The sender broadcasts on a tag,
the receiver listens on the same tag, and neither side ever holds a reference to the other. Works
from C++ and Blueprints, with global (GameInstance) and local (per-actor) scopes, sticky payloads,
deferred dispatch, save-state capture, and an in-editor debugger.

| | |
|---|---|
| **Version** | 1.0.0 — see [`CHANGELOG.md`](CHANGELOG.md) |
| **Engine** | Unreal Engine 5.5 - 5.8 |
| **Platform** | Win64 |
| **Distribution** | Through Fab — compiled for each engine version, full C++ source included. Works in Blueprint-only and C++ projects; a manual installation from source needs a C++ project. |
| **Dependencies** | None outside the engine |

**[Get the plugin](https://www.fab.com/)** · [Product page](https://stillcooking.dev/en/products/tageventbus/) · [Documentation](https://stillcooking.dev/en/products/tageventbus/docs/) · [Report an issue](https://github.com/StillCooking/TagEventBus/issues/new/choose)

## What this repository is

This is the **support and documentation** repository. It exists so that you can report a bug,
request a feature, and read the full documentation without a Fab account and without waiting on
an email.

**It contains no source code.** The plugin is commercial; the code ships with your purchase. What
you get here is the documentation exactly as it ships with the plugin, the changelog, the license,
and the issue tracker.

## What you get in the plugin

- **Addressing by tag.** No cast, no interface, no `GetAllActorsOfClass`, no manually wired
  delegate. A widget, an actor and a subsystem meet on `Combat.Hit` and know nothing else about
  each other.
- **Two scopes.** A global bus on the `GameInstance`, created automatically, and a local bus as a
  component on an actor — two disjoint registries, not one registry with a filter.
- **Typed payloads.** Any `USTRUCT`, as `Instanced Struct` in Blueprints and as a `const T&` in
  C++ — no allocation on the native path.
- **Sticky events.** A retained broadcast is replayed to listeners that register later and ask for
  it, with an optional TTL.
- **Automatic cleanup.** Managed listeners unbind when their owner is destroyed; the native
  `FTagEventSubscription` unbinds in its own destructor.
- **Delivery control.** Exact or partial tag match, priority, one-shot, throttling, tag queries,
  pause/resume, consumption policies, deferred and coalesced dispatch.
- **Save-state capture.** Retained sticky payloads survive a save/load cycle.
- **An in-editor debugger.** Who broadcasts, who receives, how many deliveries — plus an Inject tab
  that lets you test a receiver before the sender exists.
- **Optional layers.** Request-response and declarative `tag → payload type` contracts, both opt-in
  and both built on the public API.

## Documentation

The full documentation is in [`Documentation/`](Documentation/README.md) — plain Markdown, the same
tree that ships with the plugin. The same pages are online at <https://stillcooking.dev/en/products/tageventbus/docs/>.

| Group | Covers |
|---|---|
| [`start/`](Documentation/start/README.md) | Installation, the one mandatory configuration step, the first event, how to verify the build. |
| [`concepts/`](Documentation/concepts/README.md) | The model: two disjoint registries, the delivery path, listener lifetime, error handling, the debugger. |
| [`reference/`](Documentation/reference/README.md) | The full API — binding, broadcasting, bus state, types, and the native C++ surface. |
| [`advanced/`](Documentation/advanced/README.md) | The optional modules, costs, interoperability, large projects, recipes. |
| [`troubleshooting/`](Documentation/troubleshooting/README.md) | Problems by symptom, diagnostic procedures, FAQ. |
| [`project/`](Documentation/project/README.md) | Compatibility matrix, updates, roadmap, test coverage, license, how to report an issue. |

## Limits worth knowing before you buy

- **Game Thread only.** The registry is not thread-safe; bind and broadcast from the Game Thread.
- **No replication.** The bus is local to the game instance. An event broadcast on a client does
  not appear on the server.
- **Every tag must exist in the project's Gameplay Tag list.** That is the engine's rule, and it is
  the one mandatory configuration step.
- **The diagnostic layer is excluded from Shipping.** The debugger, broadcast history and
  statistics live behind `#if !UE_BUILD_SHIPPING`. Production logic runs unchanged.

Details: [`Documentation/advanced/limits-and-threading.md`](Documentation/advanced/limits-and-threading.md).

## Reporting an issue

Open an issue here: [**New issue**](https://github.com/StillCooking/TagEventBus/issues/new/choose). Three things fix most "it does not
work" reports before they are written — a tag missing from the project's tag list, a different
`Scope` on the sender and the receiver, and a listener on a parent tag without `Match = Partial`.
The checklist and what to attach: [`Documentation/project/issues.md`](Documentation/project/issues.md).

"How do I…" questions and ideas go to [Discussions](https://github.com/StillCooking/TagEventBus/discussions).

**Purchase and licensing questions do not belong in a public issue** — use the
[support form](https://stillcooking.dev/en/support/) or write to <owner@stillcooking.dev>. So do security reports: e-mail,
never a public issue.

Pull requests are not accepted here: this repository is generated from the private development
repository, so a change made here would be overwritten by the next release. Bugs and feature
requests travel through issues, questions and ideas through Discussions, and that is the whole
intake.

## License

Copyright © 2026 Hubert Knochowski, code.Matter(). All rights reserved.

Copies obtained through Fab are governed by the Fab standard license selected at purchase, together
with the Fab Marketplace terms. You may modify the source and use the modified plugin in your own
projects, commercial ones included; the source may not be redistributed. Full terms:
[`LICENSE`](LICENSE) and [`Documentation/project/license.md`](Documentation/project/license.md).
