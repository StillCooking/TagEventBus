# Tests

*The plugin passes its own automation test suite on every supported engine before each release — the suite stays with the developer and is not part of the package.*

Every release passes the plugin's own automation test suite before it is packaged. Counted by declarations in the <!-- fact:plugin-niesie-wlasny-pakiet-162-testow-automation -->
source: **288 tests across 72 files** — 266 for the core bus, contracts, and requests, and 22 for
the logic behind the debugger panel. <!-- fact:to-ze-rdzen-nie-jest-uobject-em-ma-dwie-konsekwencje-jest-w -->
The suite runs separately on each supported engine version, and a single failure stops the release.

The tests run **headless** — with no editor UI, no world, and no PIE session. That is possible
because the core is not a `UObject`: it is plain C++ that needs no world.

The same property works in your favor: **systems built on the bus can also be tested <!-- fact:rejestr-jest-plain-c-i-nie-wymaga-swiata-ani-pie-wiec-system -->
headless**, because the registry needs neither a world nor PIE.

## What the suite does not cover

- **Performance.** The tests check correctness; `stat TagEventBus` is [the tool for timing](../advanced/optimization.md). <!-- fact:pakiet-testow-biegnie-w-konfiguracji-development-poprawnosc --> <!-- fact:testy-weryfikuja-poprawnosc-nie-wydajnosc-do-pomiaru-czasow -->
- **The debugger's visual layer.** Slate cannot be tested headless — what is tested is the
  logic underneath: sorting, row synchronization, settings persistence. The view itself is <!-- fact:slate-i-warstwa-wizualna-debuggera-nie-sa-testowalne-headles -->
  verified by hand in the editor.
- **Compilation in Shipping.** The suite runs in the Development Editor configuration; Shipping compilation is checked separately by building the plugin in that configuration.

## The tests are not in the package

The test code stays in the developer's repository: **the package you receive contains no test modules and no test files**, so nothing from the suite reaches your editor, your Gameplay Tag list, or your game. <!-- fact:testy-edytorowe-zostaja-w-module-tageventbuseditor-nie-w-tag -->
To test your own systems built on the bus, write automation tests in your project's own modules —
as described above, the registry needs neither a world nor PIE.

## Reading the test results

When you run automation tests of your own — locally or on a CI server — keep one engine behavior in mind.

> [!warning]
> **`UnrealEditor-Cmd` returns exit code `0` even when the tests have failed.** You have <!-- fact:unrealeditor-cmd-zwraca-kod-wyjscia-0-nawet-gdy-testy-sie-wy -->
> to read the result from the log, looking for `Result={Success}` and `Result={Fail}` in
> `LogAutomationController`. Count the `Result={Fail}` entries to detect reported test failures.

A CI pipeline that relies on the exit code alone will report success even when tests fail.

---

← [Versions, support, and license](README.md) · [Documentation index](../README.md)
