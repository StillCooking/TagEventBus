# Contracts

*Specifying a tag's required payload struct, four enforcement levels, what validation does and does not check, and one trap tied to the tag hierarchy.*

The `TagEventBusContracts` module defines requirements for each tag's payload, scope, and sticky behavior, and detects violations of those requirements. The core has no dependency on this module: adding the layer
required **exactly one** change in the core — a generic validator hook checked once per <!-- fact:warstwa-tageventbuscontracts-wymagala-dokladnie-jednej-zmian -->
broadcast, and only outside Shipping.

## Configuration — three steps <!-- fact:konfiguracja-kontraktow-wymaga-trzech-krokow-utworzyc-schema -->

1. Create a **Tag Event Contract Schema** Data Asset (`UTagEventContractSchema`).
2. Fill in the `Contracts` list.
3. **Register the schema in Project Settings.** <!-- fact:tageventbuscontracts-wymaga-rejestracji-schematow-w-project -->

Without the third step the schema does nothing.
Only the schemas listed in the settings are active.

![Two panels stacked: at the top Project Settings → Plugins → Tag Event Bus - Contracts, with two schemas in the Schemas array and the enforcement level set to Warn; at the bottom the Data Asset editor with one contract entry expanded — Tag QA.Damage, Payload Required, Payload Struct F_QA28_Payload, Scope Any, Retain Allowed, Replacement Tag QA.Ping](../Assets/tageventbus-08-contract-schema.png)

*The contract schema and where it is registered. Until the Data Asset is added to the Schemas list in Project Settings, the contract does not apply.*

## What a single contract describes

`FTagEventContract`:

| Field | Type | Default | Meaning                                                                                                                                                        |
| --- | --- | --- |----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Tag` | `FGameplayTag` | — | the tag the contract covers; **descendants inherit it** unless they have their own                                                                             |
| `Payload` | `ETagEventContractPayload` | `Either` | whether a broadcast has to carry a payload: `Either` allows broadcasts with or without a payload, `PayloadRequired` requires one, and `SignalOnly` forbids one |
| `PayloadStruct` | `UScriptStruct*` | `nullptr` | the only permitted struct, checked exactly; empty means "the signal alone" or "any"                                                                            |
| `Scope` | `ETagEventContractScope` | `Any` | the permitted bus scope: `Any`, `GlobalOnly`, `LocalOnly`                                                                                                      |
| `Retain` | `ETagEventContractRetain` | `Allowed` | whether sticky retention is allowed, forbidden, or required for broadcasts on this tag: `Allowed`, `Forbidden`, `Required`                                     |
| `Description` | `FText` | — | a human-readable description of the event                                                                                                                      |
| `DomainOwner` | `FGameplayTag` | — | the owning domain, for example `Domain.Combat`; when empty, the schema's `DefaultDomainOwner` applies                                                          |
| `bDeprecated` | `bool` | `false` | with validation active outside Shipping, using the tag warns and points to `ReplacementTag`; **deprecation alone never blocks a broadcast**                     |
| `ReplacementTag` | `FGameplayTag` | — | the suggested replacement                                                                                                                                      |

The global settings (`UTagEventContractSettings`):

| Field | Default | Meaning |
| --- | --- | --- |
| `Schemas` | — | the schemas aggregated into the catalog; **only the listed ones are active** |
| `EnforcementLevel` | `Warn` | the reaction to a violation |
| `bRequireContractForAllBroadcasts` | `false` | when `true`, a broadcast on a tag with no contract is itself a violation |

## Enforcement levels

| Level | What it does | Broadcast |
| --- | --- | --- |
| `Off` | the validator **is not installed** — zero cost | goes through |
| `Warn` | a `Warning` in the log | goes through |
| `Ensure` | a `Warning` in the log for every violation, plus `ensureMsgf` — which fires only once per process | goes through |
| `Block` | a `Warning` in the log, and a rejection | **0 deliveries**; a `Try*` variant still reports `Success` — the `Warning` and the debugger's Live tab are the signals |

Every level except `Off` writes the same line to `LogTagEventBus`, with the fields the debugger's
Live tab shows for a violation:

```text
LogTagEventBus: Warning: Contract violation on Combat.Hit: rule=PayloadTypeMismatch expected=DamageInfo actual=SignalInfo scope=Global sender=BP_Enemy_2 level=Warn
```

- `rule` is the broken rule by name: `Uncontracted`, `PayloadRequiredButSignal`,
  `SignalOnlyButPayload`, `PayloadTypeMismatch`, `ScopeNotAllowed`, `RetainForbidden` or
  `RetainRequired` (the values of `ETagContractViolation`; `LexToString` returns the same name in C++).
- `expected` and `actual` are struct names without the `F` prefix, as Unreal reports them. A side
  with no type prints `-` — a signal has no `actual`, and `Uncontracted`, `SignalOnlyButPayload`
  and `ScopeNotAllowed` have no `expected`.
- `sender` prints `?` when the broadcast named no sender, or the sender is already gone.

Under `Ensure` the `ensureMsgf` message carries the same fields. It does not replace the line:
an `ensure` fires once per process, so from the second violation on, the log is the only trace.

> [!warning]
> **`Block` is a dev/CI gate, not a production safeguard.** All contract validation lives
> outside Shipping, so in a production build no broadcast is ever rejected by a contract.
> <!-- fact:block-to-bramka-dev-ci-nie-zabezpieczenie-produkcyjne-cala-w -->
> A contract is a tool for catching mistakes before they get out, not a safety layer running
> in the game on a player's machine.

At `EnforcementLevel = Off` the validator hook is not even installed, so adding this module <!-- fact:cztery-poziomy-egzekucji-kontraktow-off-walidator-nieinstalo -->
does not change the cost of the hot path.

## What validation checks, and in what order

The rules are checked in a fixed order: no contract (only under
`bRequireContractForAllBroadcasts`) → deprecation → payload requirement → payload type → <!-- fact:reguly-walidacji-kontraktow-sprawdzane-sa-w-stalej-kolejnosc -->
scope → retain rule.

None of those rules looks **inside** the payload. Contract validation checks the payload's <!-- fact:szyna-sprawdza-tylko-typ-payloadu-nigdy-jego-zawartosc-broad -->
type, never its contents, so `Amount = -9999` alone is not a contract violation. Validating
values that arrive from outside your code — a save game, a player's configuration, mods —
belongs on the consumer side, ideally in a listener with `Validation` priority.

## The trap: a validating listener on a parent tag is too late

This limitation applies to listeners that validate and consume events, not to inherited
contracts. The contract validator runs before listener delivery.

> [!warning]
> The "validate before anyone reacts" pattern **does not work through a parent tag**. Delivery to all listeners on the exact channel completes before delivery to ancestor channels begins, so a `Partial` listener on the parent —
> even with `Validation` priority — is considered only after all listeners on the exact
> channel, whatever match mode those exact-channel listeners use.
> <!-- fact:walidator-na-tagu-rodzicu-nie-zdazy-zawetowac-kanal-exact-do -->
> [That pattern](recipes.md#a-validator-with-a-veto) requires every listener involved to listen on the same exact tag.

It is the same rule that
[Broadcast path](../concepts/broadcast-path.md) sets out: priority only applies within a channel.

## Tag ownership

`DomainOwner` costs nothing to fill in and answers a question that gets expensive later: in a
project with several teams, **who may change what a tag means**. Set it per contract, or let
the schema's `DefaultDomainOwner` fill it in.

Without it, a year on, nobody can say whether `Combat.Hit` belongs to the combat system or to
the UI that latched onto it.

## Deprecating tags

With validation active outside Shipping, `bDeprecated` triggers a warning and points to a
replacement. Deprecation alone never blocks a broadcast. This is how you rename a tag
without breaking other people's graphs overnight.

## API

### `FindContractForTag(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventContractLibrary::FindContractForTag(Tag, OutContract, bFound)`
- **Blueprint node** — `Find Contract For Tag`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the tag to resolve |
| `OutContract` | the contract found |
| `bFound` | `false` when no contract applies |

**Returns:** nothing.

Resolves the contract **hierarchically**: it checks the tag itself first, then walks up its
ancestors until it finds a contract. **A child with an entry of its own overrides the <!-- fact:rozwiazywanie-hierarchiczne-kontraktow-lookup-sprawdza-dokla -->
parent's contract — the two are never merged.** The result is cached per tag and the cache is
cleared when the catalog is rebuilt.

### `IsTagDeprecated(...)`

*Available in: C++ and Blueprint.*

- **C++ call** — `UTagEventContractLibrary::IsTagDeprecated(Tag, OutReplacement)`
- **Blueprint node** — `Is Tag Deprecated`

| Parameter | Meaning |
| --- | --- |
| `Tag` | the tag to check |
| `OutReplacement` | the suggested replacement |

**Returns:** `bool` — whether the tag, or its nearest ancestor with a contract, is deprecated

## The panel view

The **Contracts** tab of the debugger panel is visible only when the module is active. It <!-- fact:zakladka-contracts-jest-widoczna-tylko-gdy-modul-kontraktow -->
shows a row for **every tag broadcast during the current PIE session** — whether or not
that tag has a contract.

It is the fastest way to answer "which tags are we broadcasting on, and how many of them have contracts?"

## Extending it

The core exposes one generic validator hook — `SetTagEventValidator` and <!-- fact:rdzen-wystawia-jeden-generyczny-hook-walidatora-settageventv -->
`ClearTagEventValidator` in `TagEventBusTypes.h`. It is checked once per broadcast,
including on a deferred flush; you install it in `StartupModule` and clear it in
`ShutdownModule`.

The contracts module is one user of that hook, not its owner — and **there is only one <!-- fact:jest-tylko-jeden-slot-walidatora-na-proces-instalacja-wlasne -->
validator slot per process**. Installing your own validator overrides contracts, and
installing contracts overrides yours. To have both, call both checks from a validator of
your own.

---

← [Advanced](README.md) · [Documentation index](../README.md)
