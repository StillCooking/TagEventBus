# Debugging

*Eight log categories, four console commands, and a table showing what is available in each build configuration.*

## Log categories

Eight categories, one per area: <!-- fact:osiem-kategorii-logu-odpowiada-osobnym-obszarom-funkcjonalny -->

| Category | Area |
| --- | --- |
| `LogTagEventBus` | general, including contract violations |
| `LogTagEventBusBroadcast` | delivery and payload type mismatch |
| `LogTagEventBusListeners` | binding and unbinding |
| `LogTagEventBusSticky` | setting, replaying, and clearing sticky values |
| `LogTagEventBusSave` | dumping and restoring state |
| `LogTagEventBusLifecycle` | the lifecycle of the subsystem, the component, and the registry |
| `LogTagEventBusAsync` | async actions and deferred broadcasts |
| `LogTagEventBusDebugger` | the in-editor panel |

They all start at the `Log` level. <!-- fact:wszystkie-kategorie-logowania-startuja-na-domyslnym-poziomie -->

When you are hunting a subscription leak, `LogTagEventBusListeners` is the category to turn
up — [Lifecycle](../concepts/lifecycle.md) explains what the post-GC sweep writes there and
why it is a `Warning`.

## Console commands

Four commands, available right away, without going into the project settings:

| Command | What it does |
| --- | --- |
| `TagEventBus.Log.Section <Section> <Verbosity>` | sets the level of one section |
| `TagEventBus.Log.Preset <Off\|CriticalOnly\|Normal\|Verbose\|All>` | applies a preset to every section at once |
| `TagEventBus.Log.Status` | prints the current levels |
| `TagEventBus.Log.Reset` | restores the levels from the project settings |

A typical diagnostic session: `TagEventBus.Log.Preset Verbose`, reproduce the problem,
`TagEventBus.Log.Reset`.

To silence all seven configurable log sections, turn off `bMasterEnable` in the project
settings. This does not affect `LogTagEventBus`; silence that category separately with
`Log LogTagEventBus off`.

## What is available in each build configuration

| Configuration | Debugger | History and statistics | Contract validation |
| --- | --- | --- | --- |
| Development Editor | yes | yes | yes |
| DebugGame Editor | yes | yes | yes |
| Development (outside the editor) | **no** | yes | yes |
| Shipping | **no** | **no** | **no** |

In DebugGame Editor, project modules are built without optimization, making their code
easier to inspect in a C++ debugger. This does not mean the entire editor and engine are <!-- fact:konfiguracje-buildu-roznia-sie-dostepna-powierzchnia-develop -->
unoptimized.

The production logic — binding, broadcasting, sticky, deferred, pause, save state, and
request-response — behaves identically in **every** configuration.

In Shipping, every diagnostic API call is a no-op or returns empty data. <!-- fact:per-konfiguracja-buildu-bind-broadcast-sticky-deferred-pauza -->
`Verbose` and `VeryVerbose` logs are stripped there by the engine's build system.
What happens to the editor module itself, and what that costs code referencing it, is under
[Limits and threading](limits-and-threading.md).

> [!warning]
> **The Game Thread check produces no warning in Shipping** —
> [Limits and threading](limits-and-threading.md) covers what that costs you. Run the tests and
> the game in Development before a production build, while the check still speaks up.

`ensureMsgf` reports four kinds of API misuse: calls made outside the Game Thread, fluent <!-- fact:ensuremsgf-egzekwuje-cztery-kontrakty-api-game-thread-only-l -->
chains without a terminal call, `Coalesce()` used with immediate delivery, and conflicting
payload type requirements. With immediate delivery, coalescing is ignored; in a type
conflict, the payload filter's type takes precedence.

## Two problems that look the same

> [!warning]
> **"I cannot see events in Live" and "events are not arriving" are two different things.** <!-- fact:strumien-live-wymaga-wlaczonego-record-history-statystyki-w -->
> The Live stream requires **Record history** to be on; the Overview statistics work
> regardless of that switch.
>
> Before you conclude that an event is not arriving, check the counter in Overview — it does
> not lie even with recording off.

## Statistics

The panel resolves the subsystem of the current PIE session; the statistics and the sender <!-- fact:panel-rozwiazuje-subsystem-biezacej-sesji-pie-statystyki-i-l -->
list reset automatically when PIE starts.

**Reset stats** resets the broadcast statistics and clears the sender list at the same time.

For measuring time rather than counting events, use `stat TagEventBus` —
[Optimization](optimization.md).

## When the panel is not enough

There are two places worth a breakpoint — the one every delivery passes through and the one
every listener registration passes through. [Architecture](../concepts/architecture.md) names
both.

For ordinary flow tracing you do not need a C++ debugger — the panel and section logging <!-- fact:do-debugowania-przeplywu-zdarzen-zwykle-nie-jest-potrzebny-d -->
show the same thing faster, and without stopping the game.

---

← [Advanced](README.md) · [Documentation index](../README.md)
