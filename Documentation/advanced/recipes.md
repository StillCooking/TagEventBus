# Recipes

*Ready-made patterns for the situations that come up most often, in both Blueprint and C++.*

Each recipe combines things documented elsewhere: the [concepts](../concepts/README.md) explain why
they work, the [reference](../reference/README.md) lists the parameters. This page shows how to put
those pieces together.

## A system that listens for as long as it lives

```cpp
void AMySystem::BeginPlay()
{
    Super::BeginPlay();
    if (UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this))
    {
        HitHandle = Bus->AddSignalListener(
            FGameplayTag::RequestGameplayTag("Combat.Hit"), this,
            [this](const FTagEventContext& Ctx){ OnHit(Ctx); });
    }
}

void AMySystem::EndPlay(const EEndPlayReason::Type Reason)
{
    if (UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this))
    {
        Bus->UnbindAllForObject(this);   // the safety net: unbind everything
    }
    Super::EndPlay(Reason);
}
```

The version without a manual unbind keeps an
[`FTagEventSubscription`](../reference/cpp-api.md#ftageventsubscription) as a member, so the
destructor does it. For several subscriptions at once, use a group together with the group-forming
binds:

```cpp
// class member
FTagEventSubscriptionGroup Subs;

void AMySystem::BeginPlay()
{
    Super::BeginPlay();
    if (UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this))
    {
        Bus->AddSignalListenerToGroup(Subs,
            FGameplayTag::RequestGameplayTag("Combat.Hit"), this,
            [this](const FTagEventContext& Ctx){ OnHit(Ctx); });
    }
}

void AMySystem::EndPlay(const EEndPlayReason::Type Reason)
{
    Subs.Reset();   // deterministic; the destructor would do the same, later
    Super::EndPlay(Reason);
}
```

**In Blueprint**, there are two approaches. Either `Event BeginPlay` → **Bind Tag Event**, with
`Event EndPlay` → **Unbind All For Object** (`Owner` = self, the same `Scope`) — or a single
**Bind Tag Event (Managed)**, holding on to the object it returns. The managed binding unbinds
itself on the owning actor's `EndPlay`, and you can still call `Unbind` on it at any time.

## Waiting for the first event

```cpp
Bus->AddSignalListener(StartedTag, this,
    [this](const FTagEventContext&){ DoOneTimeSetup(); },
    ETagEventMatch::Exact, /*bOnce=*/true);
```

**In Blueprint:** **Bind Tag Event** with `bOnce = true`. The handler runs once and the listener
removes itself.

## State for listeners that arrive late

The sender publishes state once, whenever it changes:

```cpp
FGamePhase Phase{ EPhase::Combat };
Bus->Broadcast(PhaseTag, this, Phase, /*bRetain=*/true);   // remembered as sticky
```

A subscriber that binds afterward receives the current phase immediately:

```cpp
Bus->AddListener<FGamePhase>(PhaseTag, this,
    [this](const FTagEventContext& Ctx){ if (auto* P = Ctx.GetPayload<FGamePhase>()) ApplyPhase(*P); },
    ETagEventMatch::Exact, /*bOnce=*/false, /*bReplaySticky=*/true);
```

**In Blueprint:** the sender uses **Broadcast Tag Event Struct** with `bRetain = true`, the
subscriber uses **Bind Tag Event** with `bReplaySticky = true`.

## One handler for a tag family

```cpp
Bus->AddSignalListener(FGameplayTag::RequestGameplayTag("Combat"), this,
    [this](const FTagEventContext& Ctx){ LogCombatEvent(Ctx.Tag); },
    ETagEventMatch::Partial);
```

Broadcasts of `Combat.Hit`, `Combat.Block`, and `Combat.Hit.Critical` all reach this listener.
`Ctx.Tag` carries the concrete tag that was broadcast, not the tag you listened on.

## A typed payload in Blueprint, with a branch for the wrong type

A Blueprint listener is always signal/any: `Payload` arrives as an `FInstancedStruct` and you unpack
it yourself. A "typed subscription" — a typed data pin **plus** a separate branch for a type
mismatch — is two nodes: the bind, and the engine's own **Get Instanced Struct Value**, which gives
a wildcard pin resolved to the struct you pick and two exec outputs, **Valid** and **Not Valid**.

```
Listen For Tag Event (Tag = "Combat.Hit")
  ► On Event (Tag, Sender, Payload)
        → Get Instanced Struct Value (Payload, As = FDamageInfo)
              ► Valid      → [use the typed FDamageInfo]
              ► Not Valid  → the "wrong type" branch (another struct, or signal-only)
```

In C++, use `AddSignalListener` to receive either a payload or a signal, then call
`GetPayload<T>()` inside the handler. It returns `nullptr` for a missing or mismatched payload:

```cpp
Bus->AddSignalListener(HitTag, this,
    [this](const FTagEventContext& Ctx)
    {
        if (const FDamageInfo* Dmg = Ctx.GetPayload<FDamageInfo>())
        {
            ApplyDamage(*Dmg);   // the type matches
        }
        // else: another type, or signal-only
    });
```

> [!warning] Leaving Not Valid unconnected hides the failure
> With nothing wired to it, a payload of the wrong type ends the graph right there, silently and
> with nothing in the log. That is what lies behind
> ["the listener fires but the payload fields are zero"](../troubleshooting/common-problems.md).

> [!tip] A contract catches the mismatch at the source
> When a tag has a [contract](contracts.md), `Ensure` reports a payload type mismatch at broadcast
> time in development builds, while `Block` rejects the broadcast. The mismatch branch then remains
> as a safety net for paths without a contract and for Shipping builds, where validation is compiled
> out entirely.

## Breaking recursion

A handler that wants to broadcast something itself — on the same tag, or on one that leads back to
it — should defer, so delivery happens at the next flush instead of nesting:

```cpp
Bus->AddSignalListener(ATag, this, [this, Bus](const FTagEventContext&)
{
    Bus->BroadcastSignalDeferred(BTag, this);   // queued, not delivered here
});
```

**In Blueprint:** **Broadcast Tag Event Deferred** (payload from a pin) or **Broadcast Tag Event
Struct Deferred** (a wildcard struct) — two separate nodes. The payload is copied at the call, so
both are safe from inside a handler.

Deferring delivery avoids immediate recursion, but a cycle of deferred broadcasts still needs an
exit condition.

## Skipping an expensive payload nobody wants

```cpp
if (Bus->HasMatchingListeners(ExpensiveTag))
{
    FHeavyReport Report = BuildExpensiveReport();   // only when someone is listening
    Bus->Broadcast(ExpensiveTag, this, Report);
}
```

**In Blueprint:** a `Branch` on **Has Matching Listeners** before you build the payload.

> [!warning] `HasListeners` is the wrong guard here
> It counts the exact channel only, so it would skip a broadcast that a `Partial` listener on a
> parent tag was going to receive. `HasMatchingListeners` follows the real broadcast path — see
> [Bus state](../reference/bus-state.md).

## The local bus

Events addressed to one actor, without touching the global bus:

```cpp
if (UTagEventBusComponent* Comp = TargetActor->FindComponentByClass<UTagEventBusComponent>())
{
    Comp->Broadcast(StunTag, this, FStunInfo{2.0f});
}
```

**In Blueprint:** add the **Tag Event Bus** component to the actor, then use `Scope = Local` with a
`WorldContextObject` that points at that actor. Alternatively, the first **Bind** with an explicit
`bCreateLocalIfMissing = true` creates the component for you.

## Saving sticky state into a save game

```cpp
// Saving
FTagEventBusSaveState State;
UTagEventBusSubsystem::Get(this)->CaptureRetainedState(State);
// ... State goes into a USaveGame field marked UPROPERTY(SaveGame) ...

// Restoring, after the save is loaded
UTagEventBusSubsystem::Get(this)->RestoreRetainedState(State);
```

The whole cycle works in Blueprint too, but it needs three things the palette will not suggest on
its own: the bus object, your own SaveGame class, and the **Save Game** flag on the payload fields.

**1. The SaveGame class.** Create a Blueprint deriving from **SaveGame**, add a variable of type
**Tag Event Bus Save State**, and check **Save Game** in its Details. Without that flag the variable
never reaches the file at all.

**2. Saving.**

```
Get TagEventBusSubsystem  ──►  Capture Retained State
                                          │ (Out pin)
                                          ▼
Create Save Game Object (Class = your SaveGame BP)  ──►  Set Bus State  ──►  Save Game to Slot
```

**3. Restoring**, on a fresh session, after `Event BeginPlay`:

```
Load Game from Slot  ──►  Cast To (your SaveGame BP)  ──►  Get Bus State
                                                               │ (In pin)
                                                               ▼
Get TagEventBusSubsystem  ──►────────────────────►  Restore Retained State
                                                               │ (Return Value)
                                                               ▼
                                                    Switch on ETagEventRestoreResult
```

`Restore Retained State` returns `Restored`, `Restored With Drops`, or `Rejected Newer Version`. The
last one means the save uses a schema version newer than this build supports, and nothing was
restored — worth telling the player about. Listeners that bind **after** the restore with
`bReplaySticky = true` receive the restored values. Restoring state does not itself deliver those
values to listeners that were already bound, because it rebuilds sticky state rather than
broadcasting it.

> [!warning] The fields of your payload struct need the Save Game flag too
> Without it you get entries of the right type with **zeroed** values after loading, and no warning
> anywhere. Check **Save Game** on every variable of your user-defined struct.

Details and edge cases: [Bus state](../reference/bus-state.md).

## Muting gameplay listeners during a menu or a cutscene

A group lets you suspend a whole batch of bindings with one call — no unbinding, no rebinding —
and resume them later, optionally catching up on the state that was missed.

```cpp
// class member
FTagEventSubscriptionGroup GameplaySubs;

void AHud::BeginPlay()
{
    Super::BeginPlay();
    if (UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this))
    {
        Bus->AddSignalListenerToGroup(GameplaySubs,
            FGameplayTag::RequestGameplayTag("Combat.Hit"), this,
            [this](const FTagEventContext& Ctx){ OnHit(Ctx); });

        Bus->AddListenerToGroup<FGamePhase>(GameplaySubs,
            FGameplayTag::RequestGameplayTag("Game.Phase"), this,
            [this](const FTagEventContext& Ctx){ if (auto* P = Ctx.GetPayload<FGamePhase>()) ApplyPhase(*P); },
            ETagEventMatch::Exact, /*bOnce=*/false, /*bReplaySticky=*/true);
    }
}

void AHud::OnMenuOpened()
{
    GameplaySubs.Pause();   // gameplay events are lost while the menu is open; bOnce survives
}

void AHud::OnMenuClosed()
{
    GameplaySubs.Resume(/*bReplaySticky=*/true);   // resume and catch up on the current phase
}
```

**In Blueprint:** `Event BeginPlay` → **Create Listener Group** (`Owner` = self) → **Bind Tag Events
To Group** for the whole set of tags at once, or **Bind Tag Event To Group** for a single one.
The menu-opened event calls **Pause** on the group; the menu-closed event calls **Resume**.

> [!tip] `bReplaySticky` on resume only helps where someone retained something
> It has an effect only for tags that are broadcast with `bRetain = true`. Everywhere else there is
> nothing to replay and it behaves like `bReplaySticky = false`.

## A validator with a veto

A chain of interceptors on the `Validation` band, where any one of them can veto the event before it
reaches state modifiers and UI.

```cpp
// The validator — highest priority, vetoes what is not allowed
Bus->Listen(ActionTag)
   .Owner(this)
   .Priority(TagEventPriority::Validation)
   .HandleWith(this, &ThisClass::OnValidateAction);

ETagEventHandling AMyValidator::OnValidateAction(const FTagEventContext& Ctx)
{
    if (const FActionRequest* Req = Ctx.GetPayload<FActionRequest>())
    {
        if (!IsActionAllowed(*Req))
        {
            return ETagEventHandling::StopPropagation;   // Modifier, Default, and UI get nothing
        }
    }
    return ETagEventHandling::Continue;
}

// The emitter — the policy does not have to be StopAfterHandled:
// StopPropagation stops delivery even under BroadcastAll
FTagEventBroadcastResult Result = Bus->BroadcastWithPolicy(ActionTag, this, Req,
    ETagEventBroadcastPolicy::BroadcastAll);
if (Result.bWasStopped)
{
    // the action was vetoed — log the reason on the calling side
}
```

**In Blueprint:** the listener binds with **Bind Tag Event** at `Priority` 1000 (the `Validation`
band) and calls **Consume Tag Event** with `Handling = Stop Propagation` in the "not allowed"
branch. The emitter uses **Broadcast Tag Event (Policy)** and checks `bWasStopped` on the result.

> [!warning] The validator has to sit on the exact channel, not on a parent
> Delivery walks the exact channel in full and only then the ancestors, so a validator registered as
> `Partial` on a parent tag is **too late** to veto the exact listeners of the same broadcast. For
> "validate before anyone reacts" to hold, every listener involved has to sit on the same exact tag
> and differ only in priority. See [Priority bands](../concepts/broadcast-path.md#priorities).

## First-responder-wins

Input or UI routing where the **first** listener to handle the event ends the matter. Use the
`StopAfterHandled` policy and have each listener mark the event as `Handled` when it handles the
input.

```cpp
// Candidates, from the most specific to the most general
Bus->Listen(InputTag).Owner(Widget).Priority(TagEventPriority::UI + 100)
    .HandleWith(Widget, &UMyTopWidget::OnInput);        // returns Handled if it consumed the input

Bus->Listen(InputTag).Owner(Fallback).Priority(TagEventPriority::UI)
    .HandleWith(Fallback, &UMyFallbackWidget::OnInput); // only reached if the previous one returned Continue

FTagEventBroadcastResult Result = Bus->BroadcastWithPolicy(InputTag, this, InputData,
    ETagEventBroadcastPolicy::StopAfterHandled);
if (!Result.bWasHandled)
{
    // nobody took it — fall through to gameplay
}
```

**In Blueprint:** the candidates bind with descending `Priority` values, most specific first, and each calls **Consume Tag Event**
with the default `Handling = Handled` when it consumes the input. The emitter uses **Broadcast Tag
Event (Policy)** with `Policy = Stop After Handled`, and the `bWasHandled == false` branch is where
the fallback behavior goes.

> [!tip] Reach for `Handled` before `StopPropagation`
> Under `StopAfterHandled`, `Handled` already ends the delivery. `StopPropagation` is for when you
> want that effect **regardless** of the emitter's policy — typically because you do not control the
> emitting side.

## Testing a global-bus receiver with no real sender

1. Start PIE and open **Tools → Debug → TagEventBus Debug**.
2. Go to the **Inject** tab, pick a tag, set a payload — or leave it empty for a signal — and
   optionally check **Retain**.
3. Press **Broadcast**. The feedback line says how many listeners it reached.
4. The injected event has a sender whose name starts with **`DebugInject`**, so it is easy to tell
   apart from real traffic in the Overview and Live tabs.

The whole panel: [Debugger](../concepts/debugger.md).

---

← [Advanced](README.md) · [Documentation index](../README.md)
