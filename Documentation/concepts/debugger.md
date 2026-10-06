# Debugger

*The in-editor panel shows who broadcasts and who receives events — and the Inject tab lets you test a receiver before the sender exists.*

## Where it is

**Tools → Debug → TagEventBus Debug.**

The in-editor debugger works the moment the plugin is enabled, with no setup at all. <!-- fact:poza-sesja-pie-panel-debuggera-pokazuje-komunikat-start-play -->
Outside a PIE session the panel shows "Start Play In Editor to see live data." That is
correct behavior.

## Overview — who broadcasts, and who receives

A tag tree with counters. Expanding a tag shows the senders, each with its own counters.

**A high `Broadcasts` count next to `Delivered = 0` is the fastest way to find an object <!-- fact:rozwiniecie-tagu-w-overview-pokazuje-nadawcow-z-wlasnymi-lic -->
broadcasting into the void**. <!-- fact:panel-debuggera-pokazuje-wycieki-subskrypcji-wizualnie-czerw -->

Subscription leaks show up here too: a red `DEAD` in the tag tree and an `N DEAD` counter in
the summary bar.

When `Delivered` does not go up, the cause can be any of the five delivery gates, and none of
them says so in the log — [Error handling](error-handling.md) explains why. **The state of all
five is visible on the listener card in the panel**, which is the fastest way to see which one
is holding the event back.

![The Overview tab of the debugger panel during a PIE session: a 1 DEAD counter in the summary bar, a tag table with the columns Listeners, Broadcasts, Delivered, Handled, Stopped, and Retained, and beside it the selected tag's panel with three listener cards — in the last one the owner reads (dead) and the Alive field shows a red DEAD](../Assets/tageventbus-07-debugger-overview.png)

*Overview. Every listener shows its owner and its binding parameters; a red DEAD marks a listener that outlived its owner.*

## Live — the event stream and the payload

**Record history must be enabled for events to appear in Live.**

Selecting a row in Live opens the Payload panel, which requires **Capture payloads** to be <!-- fact:zaznaczenie-wiersza-live-otwiera-panel-payload-wymaga-wlaczo -->
on. It has a **Copy to Inject** button that copies the tag, the payload, and the
`ContextTags` over to the Inject tab.

That is the shortest route from "I can see the event I care about" to "I can replay it by
hand as many times as I need".

## Inject — broadcasting by hand

The Inject tab lets you **test a receiver before the sender even exists**. <!-- fact:zakladka-inject-panelu-debuggera-pozwala-przetestowac-odbior -->

After the broadcast the panel shows an inline message of the form <!-- fact:broadcast-reczny-z-zakladki-inject-potwierdza-dzialanie-odbi -->
"Broadcast Combat.Hit -> N listener(s)".
If `Delivered` increases, the injected event reached at least one listener; confirm that it <!-- fact:rosnacy-licznik-delivered-po-recznym-broadcastcie-z-inject-l -->
reached the intended receiver. If it does not increase, check the tag, the receiver's
subscription to the global bus, its pause state, and its delivery filters.

## What the panel remembers

Across editor sessions it keeps the active tab, the tag filters, the sort order in Overview,
Auto-scroll, the Retain checkbox in Inject, and Capture payloads. It **does not keep** <!-- fact:panel-zapamietuje-miedzy-sesjami-edytora-aktywna-zakladke-fi -->
session state: Pause, the selected tag, and the contents of the Inject tab.

The settings are written to `EditorPerProjectUserSettings.ini`, in the <!-- fact:ustawienia-panelu-debuggera-zapisuja-sie-w-editorperprojectu -->
`[TagEventBusDebugPanel]` section. That file is per-user and outside version control, so
your filters and sorting do not affect the rest of the team.

## What the panel will not do

> [!warning]
> **The panel sees the global bus only.** It resolves the `UTagEventBusSubsystem` of the <!-- fact:debugger-nie-widzi-busow-lokalnych-panel-rozwiazuje-utageven -->
> current PIE session; bus components on actors are **not visible** in Overview or in Live,
> and Inject targets the global bus alone.
>
> You diagnose problems in `Local` scope with logs and the `Has Matching Listeners` node, not
> with the panel.

> [!warning]
> **The panel does not exist in Shipping.** The diagnostic layer is compiled conditionally and
> the editor module is not built at all in that configuration. The diagnostic methods stay
> callable, but they report nothing: `GetListenerDebugInfo` and `GetBroadcasterDebugInfo` come <!-- fact:panel-debuggera-historia-i-statystyki-zyja-za-if-ue-build-sh -->
> back empty, `GetTagStats` returns `false`, and `OnBroadcastRecorded` never fires. Production
> logic runs unchanged.

The [Debugging](../advanced/debugging.md) page covers the differences between build
configurations in full, and what the counters do when a PIE session starts;
[Architecture](architecture.md) explains why they do it.

---

← [Concepts](README.md) · [Documentation index](../README.md)
