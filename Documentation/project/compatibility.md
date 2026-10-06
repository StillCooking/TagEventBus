# Compatibility

*What has been built and tested, what your project has to provide, and the two defects this release ships with.*

## Engine versions and platforms

| Plugin version | Engine | Platform |
| --- | --- | --- |
| 1.0.0 | 5.5 | Win64 (development) |
| 1.0.0 | 5.6 | Win64 (development) |
| 1.0.0 | 5.7 | Win64 (development) |
| 1.0.0 | 5.8 | Win64 (development) |

**Only these combinations have been built and tested.** The table records what has been
verified, not plans.

The plugin is built and the full test suite is run separately for each of the four engine versions, using a separate host project for each version. There is not a single version guard in the plugin's code.

The plugin is distributed as source, so even without official support you have the full code
and can adapt it yourself. In code of this kind, engine migrations between minor versions
rarely need anything beyond a recompile.

## What the plugin requires of your project

- **A C++ project — only if you install the plugin manually from source**, because it then <!-- fact:plugin-jest-dostarczany-w-zrodlach-wiec-wymaga-projektu-c-zd -->
  compiles on your side. [Installed from Fab, the plugin arrives compiled and a Blueprint-only
  project is enough](../start/installation.md#requirements).
- **Visual Studio, to package a game from a Blueprint-only project.** With the plugin enabled,
  Unreal Engine treats the project as a code project and builds a temporary game target for it.
- **Gameplay Tags** — an engine plugin; enable it in the Plugins browser.
- **`TagEventBus` and `GameplayTags` in `Build.cs`** — only if you use the API from C++.

## What the plugin does not do

Two different promises, and the difference matters to anyone planning a two-year project:
what is missing **by design** and will stay that way, and what is missing **for now** —
including [the buses the debugger panel does not show](../concepts/debugger.md).
[Limits and threading](../advanced/limits-and-threading.md) keeps those two lists apart, and
neither of them carries a date.

Two consequences of that shape have pages of their own: what shipping a game with this plugin
[does and does not oblige you to](license.md), and how the bus
[sits beside the engine's own networking](../advanced/interoperability.md).

## Known limitations

**Find in world in the debugger panel** has three independent effects under three different
conditions: framing the actor in the viewport requires an active viewport, the yellow box needs the actor to
have a world, and selection in the World Outliner needs the actor's world to be the editor
world. **In a PIE session the selection never happens**, because the PIE world is not the <!-- fact:find-in-world-ma-trzy-niezalezne-efekty-pod-trzema-roznymi-w -->
editor world, and in this release the framing does not work in PIE either — for reasons not
yet established. That is a known limitation, not a design choice.

**The `LogTagEventBus` category is not controlled by the plugin's log section settings.** `ETagEventBusLogSection`
has only seven values and that category is not among them, so neither `bMasterEnable` nor the
presets nor a section setting affects it. You silence it with the engine's own <!-- fact:umbrella-logtageventbus-jest-poza-sterowaniem-pluginu-etagev -->
`Log LogTagEventBus off` command. It is the general-purpose category; contract violations are
reported there as well.

---

← [Versions, support, and license](README.md) · [Documentation index](../README.md)
