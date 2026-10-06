# License

*A copy bought on Fab is governed by the Fab license. You may modify the source for your own use; you may not redistribute the source.*

## Terms of purchase

The plugin is sold through **Fab**, so a purchased copy is governed by the standard Fab
license you choose at checkout. Fab offers two tiers of the same license and a seller **has
to make both available** — it is not the author's choice. The buyer picks the tier by
**their own** gross revenue over the last twelve months:

| Tier | Who it is for |
| --- | --- |
| **Personal** | gross commercial revenue **up to** USD 100,000 in the last 12 months |
| **Professional** | gross commercial revenue **above** USD 100,000 in the same period |

Epic sets the threshold and the terms and can change them without the author's involvement,
so the authority is the
[Fab license documentation](https://dev.epicgames.com/documentation/fab/licenses-and-pricing-in-fab),
not this table.

> [!note] These are not Unreal Engine royalties
> Two different agreements with no connection between them. **Royalties** are calculated by
> Epic on your **game's** revenue under the engine's EULA. **The Fab tier** looks at your
> revenue as a **buyer** and decides only which purchase tier applies to you.

## Distributing your game

The plugin **does not link against `Boost`, does not require `DirectX` or `OpenGL` beyond what the <!-- fact:plugin-nie-linkuje-boost-nie-wymaga-directx-opengl-ponad-sil -->
engine already requires, and contains no third-party code**, so it imposes no extra licensing
conditions on your game and there is no third party whose terms carry over to it. The engine
modules it links against are listed in `THIRD-PARTY.md`, in the plugin's root directory. The
package contains no test code at all — see [Tests](tests.md).

## Source distribution

The plugin ships as source, so **you have the full source code**: even after support ends, <!-- fact:plugin-jest-dostarczany-w-zrodlach-wiec-nawet-po-zakonczeniu -->
you can keep adapting it to newer engine versions. What building it from source requires of
your project is on [Compatibility](compatibility.md).

**You may modify the source and use the modified version in your own projects**, commercial
ones included, on the same terms as an unmodified copy — without asking, and without losing
your right to updates. That permission covers **using the source, not redistributing it**:
modified or not, it may not be redistributed, published, sublicensed, or sold, and a modified
copy travels only as a compiled part of your product.

Modifying the core rather than building a layer over the public API has a price on every
update, which [Updates](updates.md) puts a number on;
[Interoperability](../advanced/interoperability.md) covers the three routes that avoid it.

## What binds

This page is a summary; nothing on it overrides **the Fab license that came with your copy**
or **the `LICENSE` file in the plugin's root directory**. `LICENSE` is where to read the
warranty disclaimer, the limit of liability, support, names and marks, the governing law —
and where the line between the two documents is drawn: what a breach of `LICENSE` ends, and
what it leaves untouched because it belongs to the licence you bought.

---

← [Versions, support, and license](README.md) · [Documentation index](../README.md)
