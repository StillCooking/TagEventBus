# Updates

*A six-step update procedure, the versioning rules, and the one place where rolling back hurts.*

## Where to check the current version

In the `VersionName` field in `TagEventBus.uplugin`, and the heading in `CHANGELOG.md`. <!-- fact:wersja-biezaca-pluginu-jest-zawsze-widoczna-w-polu-versionna -->

## The update procedure

Six steps, in this order: <!-- fact:procedura-aktualizacji-ma-szesc-krokow-w-ustalonej-kolejnosc -->

1. **Read the migration notes.**
2. **Make a backup or a commit.**
3. **Replace the plugin directory and rebuild** — delete `Binaries/` and `Intermediate/` first.
4. **Run your project's own tests** and play through the systems that use the bus.
5. **Refresh the `CustomThunk` nodes**, if you need to.
6. **Check the Shipping build.**

> [!tip]
> Step four catches most problems before a player does, and spares you the "something is broken, but nobody knows what" routine. The plugin itself has already passed its own suite on every supported engine before the release — see [Tests](tests.md).

Step five applies to `Broadcast Tag Event Struct` and its deferred variant. They are
`CustomThunk` nodes with the payload on a wildcard pin — Blueprint cannot update their
signatures on its own, so a new pin does not appear on saved graphs without **Refresh Node**
or a recompile.

Step six is not a formality. A large part of the diagnostic surface is compiled <!-- fact:znaczna-czesc-powierzchni-diagnostycznej-jest-kompilowana-wa -->
conditionally, so **a reference to that surface from your own code only shows up on your
first Shipping build** — it is worth running one periodically, not once at the end of
the project.

## What the version number means

Semantic Versioning: <!-- fact:plugin-stosuje-semantic-versioning-major-dla-zmian-lamiacych -->

| Part | When it goes up                                                                                                                      |
| --- |--------------------------------------------------------------------------------------------------------------------------------------|
| Major | a change that breaks the public C++/BP API, the module layout, or the supported engine version — **always with a migration section** |
| Minor | new, backward-compatible functionality; a deprecated symbol is never removed in a minor release                                       |
| Patch | bug fixes only, with no observable API changes                                                                                       |

## The deprecation cycle

A symbol marked for removal goes through a cycle; it does not disappear overnight: <!-- fact:symbol-przeznaczony-do-usuniecia-przechodzi-najpierw-cykl-de -->

1. Marked deprecated in at least one release.
2. Removed only in a major release.
3. Documented in a migration entry.

The same discipline applies to the tags in your project: `bDeprecated` in a contract provides a warning and points to a replacement, but it **never prevents the tag from being used** —
[Contracts](../advanced/contracts.md).

## Version history <!-- fact:macierz-zgodnosci-wersji-1-0-0-2026-07-03-pierwsze-wydanie-r -->

| Version | Date | Changes                                                                 |
| --- | --- |-------------------------------------------------------------------------|
| 1.0.0 | 2026-10-05 | the first release — the core, the debugger, contracts, request-response |

This release supports UE 5.5 – 5.8. The full list is in the
[changelog](../../CHANGELOG.md).

## Rolling back to an older version

> [!warning]
> **This is the one place where a downgrade hurts.** A save written by a newer version of the <!-- fact:save-zapisany-przez-nowsza-wersje-pluginu-jest-odrzucany-w-c -->
> plugin is rejected **outright**: nothing is restored, the registry is left untouched, and an
> `Error` appears in the log.
>
> It is the only `Error`-level message anywhere in the plugin's runtime, and it is logged in
> the Save section:
>
> ```log
> RestoreRetainedState: snapshot version N is newer than supported version M — nothing restored
> ``` <!-- fact:w-calym-runtime-pluginu-jest-tylko-jedna-wiadomosc-na-poziom -->
>
> Rolling back the plugin and then loading a player's existing save triggers this error every time. The plugin never
> uses the `Fatal` level anywhere. <!-- fact:save-z-nowszej-wersji-gry-jest-odrzucany-w-calosci-zamiast-w -->

Rejecting it outright is a decision, not an omission. If the save were loaded only partially, the game would start with silently truncated state.

Saves from older versions load without trouble — they are migrated forward. An entry whose
struct no longer exists is skipped, and the result is `RestoredWithDrops`.

## Modifying the core

> [!warning]
> Modifying the core instead of building a layer over the public API means **merging changes <!-- fact:modyfikacja-rdzenia-zamiast-budowania-warstwy-nad-publicznym -->
> by hand on every update**.
>
> [Interoperability](../advanced/interoperability.md) covers three routes
> to extending the plugin without that cost.

---

← [Versions, support, and license](README.md) · [Documentation index](../README.md)
