# Installation

*Both ways to install the plugin — from Fab or manually from source — plus the Build.cs dependencies and a check that the modules built.*

## Requirements

| Plugin version | Engine | Platform |
| --- | --- | --- |
| 1.0.0 | 5.5 – 5.8 | Win64 (development) |

The plugin is built separately for each of the four engine versions and passes the full test suite on each.
The complete matrix: [Compatibility](../project/compatibility.md).

**The kind of project you need depends on how you install the plugin.** From Fab it arrives <!-- fact:plugin-jest-dostarczany-w-zrodlach-wiec-projekt-musi-byc-zdo -->
compiled for your engine version and works in Blueprint-only and C++ projects alike. Installed
manually from source, it compiles on your side, so the project has to be a C++ project.

The plugin pulls nothing in from outside the engine: no third-party code and no dependencies on other marketplace plugins. It does depend on the engine's own `GameplayTagsEditor`.

## Installing from Fab

1. In the Epic Games Launcher, find TagEventBus in your Fab library and choose **Install to
   Engine** for the engine version you work with. The plugin lands in that engine's
   `Engine/Plugins/Marketplace/` directory, already compiled.
2. Open your project and enable **TagEventBus** in **Edit → Plugins**.
3. **Restart the editor.**

The plugin itself needs no build on your side: no project files to generate, no Development
Editor build.

## Installing manually from source

1. Copy the plugin directory into `Plugins/TagEventBus/` in your project. The `.uplugin` <!-- fact:jesli-plugin-nie-widnieje-na-liscie-katalog-jest-zly-albo-na -->
   file has to sit **directly** in `Plugins/TagEventBus/`. If the plugin does not show up
   in the Plugins browser, its directory is usually nested one level too deep, or the
   directory name contains a typo.
2. Run **Generate Visual Studio project files** on the `.uproject` file. The plugin appears <!-- fact:po-generate-visual-studio-project-files-plugin-pojawia-sie-w -->
   in the solution as a separate project under `Games/<YourProject>/Plugins/`, with full
   IntelliSense.
3. Build the project in the **Development Editor** configuration.
4. Open the editor. A plugin placed in `Plugins/` is usually enabled automatically. If you <!-- fact:plugin-umieszczony-w-plugins-projektu-zwykle-wlacza-sie-sam -->
   had to enable it by hand in the Plugins browser, **restart the editor**.

## Modules

| Module | Type | Role |
| --- | --- | --- |
| `TagEventBus` | Runtime | the core: registry, binding, broadcasting |
| `TagEventBusEditor` | Editor | the debugger panel |
| `TagEventBusRequests` | Runtime | request-response, optional |
| `TagEventBusContracts` | Runtime | tag → struct contracts, optional |

"Optional module" means that consumer code does not have to reference it in `Build.cs`.
With the plugin enabled, the runtime modules load both in the editor and in a packaged
game. The editor modules load only in the editor. The core behaves identically whether or
not you use the optional modules.

## Module dependencies

**If you use the API from C++**, add this to your own `Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "TagEventBus", "GameplayTags" });
```

`GameplayTags` is mandatory here because your own code declares and passes `FGameplayTag`. <!-- fact:gameplaytags-musi-byc-dopisany-w-build-cs-konsumenta-bo-jego -->
A plugin's dependencies do not carry over to the consumer automatically.

Leaving out either of those two entries results in an `unresolved external symbol` linker <!-- fact:blad-linkera-unresolved-external-symbol-przy-wolaniu-api-z-c -->
error as soon as your code references the API.

**If you work only in Blueprints**, you skip this step entirely — the nodes work with no <!-- fact:konsument-korzystajacy-wylacznie-z-blueprintow-moze-pominac -->
changes to `Build.cs` at all.

## Verification

Open **Tools → Debug → TagEventBus Debug**. The presence of that entry proves two things at
once: the plugin is enabled, and its modules have built successfully. A checked box in the Plugins
browser does not guarantee that on its own.

If the entry is missing, the plugin is disabled or the editor module did not build. What to check, in
order: [Verification](verification.md).

---

← [Start here](README.md) · [Documentation index](../README.md)
