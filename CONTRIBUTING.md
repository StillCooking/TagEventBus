# Contributing

TagEventBus is a commercial plugin sold on Fab. This repository is its **support channel** and
its **documentation mirror** — it carries no source code, and it is generated from a private
development repository at every release.

That shape decides what is useful here.

## Issues — yes

[Open an issue](https://github.com/StillCooking/TagEventBus/issues/new/choose). Bug reports and feature
requests both belong here, whether or not you have bought the plugin.

Before you file a bug, three checks fix most reports:

1. **The tag exists in the project's Gameplay Tag list.** A tag outside it is rejected on bind and
   on broadcast alike.
2. **The sender and the receiver use the same `Scope`.** `Global` and `Local` are two disjoint
   registries.
3. **A listener on a parent tag has `Match = Partial`.**

What to attach so the problem can be reproduced:
[`Documentation/project/issues.md`](Documentation/project/issues.md).

## Questions and ideas — Discussions

"How do I…" questions and loose ideas go to
[Discussions](https://github.com/StillCooking/TagEventBus/discussions). A question is not a defect, and an
answer there stays findable for the next person who asks.

## Pull requests — no

A change committed here would be overwritten by the next generated snapshot, including a fix to a
documentation page: those pages are a render, not a source. If you have found an error in the
documentation, open an issue — a sentence quoted with the page name is enough, and the fix lands
upstream.

## Purchase, invoices and licensing — privately

Those are not public matters: use the [support form](https://stillcooking.dev/en/support/) or write to
<owner@stillcooking.dev>.

## Security

If you believe you have found a vulnerability, do not open a public issue. Write to
<owner@stillcooking.dev> with the details and the engine version.
