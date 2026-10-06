# Issues

*What to include in a report so the problem can be reproduced — and which channel to send it through.*

## Before you report

You can fix the three most common causes of "it does not work" yourself, without submitting a report:

1. **A tag outside the project's tag list** — rejected both on broadcast and on bind.
   Blueprint nodes log a `Warning`; the low-level C++ API rejects it silently.
2. **A different `Scope` on the sender and on the receiver** — `Global` and `Local` are two
   disjoint registries.
3. **A listener on a parent tag without `Match = Partial`.**

The full list of symptoms:
[Common problems](../troubleshooting/common-problems.md).

## What to include

To help reproduce the problem, include:

- **The plugin version** — the `VersionName` field in `TagEventBus.uplugin`, or the heading
  in `CHANGELOG.md`.
- **The engine version and the build configuration.** They matter: the available features
  and diagnostic tools differ between Development Editor, Development, and Shipping.
- **The scope** — `Global` or `Local`. For `Local` scope, the debugger panel does not show the local bus, so you need logs.
- **A log from the right category.** Eight categories map to separate areas, so it is worth
  raising the verbosity of the right one instead of sending everything.
- **For the global bus, the result of a manual broadcast from the Inject tab** — include the
  delivery count and whether the intended receiver received the event.

> [!tip]
> The fastest way to a complete log:
>
> ```text
> TagEventBus.Log.Preset Verbose
> ```
>
> Reproduce the problem, then run `TagEventBus.Log.Reset`. A `Verbose` log on its own is
> usually enough for a diagnosis.

## Which channel

**Bugs and feature requests — [GitHub Issues](https://github.com/StillCooking/TagEventBus/issues).**
Each issue is public, supports file attachments, and has a number that ends up in the changelog.
A report others can see saves time for whoever runs into the same problem.

**"How do I…" questions and ideas — [GitHub Discussions](https://github.com/StillCooking/TagEventBus/discussions).**
A question is not a defect, and an answer there stays findable for the next person who asks.

**Private matters — purchase, invoices, licensing — the [support form](https://stillcooking.dev/en/support/)
or [owner@stillcooking.dev](mailto:owner@stillcooking.dev).**

**Security issues — [owner@stillcooking.dev](mailto:owner@stillcooking.dev), not a public issue.**
A vulnerability described in public is open to everyone until a fix ships.

---

← [Versions, support, and license](README.md) · [Documentation index](../README.md)
