# Native C++ API

*The subsystem and component API available only in C++: templated `AddListener<T>` and `Broadcast<T>`, `FTagEventContext`, the RAII subscription helpers, and the fluent builder.*

All API calls on this page must be made on the **Game Thread**. The native layer is the fastest
path — the payload travels as a `const T&` and a native listener reads the caller's own memory,
with no copy and no `FInstancedStruct` in between — and it is the only path that reaches the
registry directly. [Blueprint listeners, `bRetain` and deferred delivery still copy the
payload](../advanced/optimization.md#payload).

The parameters themselves — `Match`, `bOnce`, `bReplaySticky`, `Priority`, `ThrottleSeconds`,
`Query` — mean exactly what they mean in Blueprint, and the
[reference introduction](README.md) describes them once for both. This page covers the shape of the
native calls and the things Blueprint has no counterpart for.

## Reaching the bus

```cpp
UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this);
```

`Get` returns `nullptr` very early and very late in the game instance's life, so a null check is not
ceremony. The local bus is an ordinary component:

```cpp
UTagEventBusComponent* Local = Actor->FindComponentByClass<UTagEventBusComponent>();
```

Both expose the same surface. Where they differ is listed under
[The local bus in C++](#the-local-bus-in-c) below.

```cpp
FTagEventRegistry& Registry = Bus->GetRegistry();
```

`GetRegistry()` hands you the core the subsystem and the component are both built on. You need it
for three things: listeners that consume events, payload predicates, and the sticky and debug
surface. It is not a `UObject` and it is not thread-safe — never keep a raw pointer to it that
outlives its owner.

## Broadcasting

```cpp
// Typed payload — passed as const T&, native listeners read it without a copy
template <typename T>
void Broadcast(FGameplayTag Tag, UObject* Sender, const T& Payload, bool bRetain = false,
    const FGameplayTagContainer& ContextTags = FGameplayTagContainer(), float RetainTTLSeconds = 0.f);

// A signal with no payload
void BroadcastSignal(FGameplayTag Tag, UObject* Sender, bool bRetain = false,
    const FGameplayTagContainer& ContextTags = FGameplayTagContainer(), float RetainTTLSeconds = 0.f);

// With a ready-made FInstancedStruct — from Blueprint, or from data
void BroadcastInstanced(FGameplayTag Tag, UObject* Sender, const FInstancedStruct& Payload, bool bRetain,
    const FGameplayTagContainer& ContextTags = FGameplayTagContainer(), float RetainTTLSeconds = 0.f);

// In use:
Bus->Broadcast(Tag, this, Payload);
Bus->BroadcastSignal(Tag, this);
Bus->BroadcastInstanced(Tag, this, Instanced, false);
```

`T` has to be a `USTRUCT` — the template resolves its type through `TBaseStructure<T>::Get()`.

`ContextTags` are metadata about the broadcast, separate from the payload. A listener's `Query` is
matched against the **union** `{Tag} ∪ ContextTags`, but the listener itself reads back only the
container, without the event tag — see [Broadcast](broadcast.md).

`RetainTTLSeconds` has an effect only together with `bRetain = true`: it clears the sticky value
after that many seconds of real time. On its own it is ignored and logs a `Warning`.

### With a consumption policy

```cpp
template <typename T>
FTagEventBroadcastResult BroadcastWithPolicy(FGameplayTag Tag, UObject* Sender, const T& Payload,
    ETagEventBroadcastPolicy Policy, bool bRetain = false,
    const FGameplayTagContainer& ContextTags = FGameplayTagContainer(), float RetainTTLSeconds = 0.f);

FTagEventBroadcastResult BroadcastSignalWithPolicy(FGameplayTag Tag, UObject* Sender,
    ETagEventBroadcastPolicy Policy, bool bRetain = false, ...);

FTagEventBroadcastResult BroadcastInstancedWithPolicy(FGameplayTag Tag, UObject* Sender,
    const FInstancedStruct& Payload, bool bRetain, ETagEventBroadcastPolicy Policy, ...);

// In use:
Bus->BroadcastWithPolicy(Tag, this, Payload, ETagEventBroadcastPolicy::StopAfterHandled);
Bus->BroadcastSignalWithPolicy(Tag, this, ETagEventBroadcastPolicy::StopAfterHandled);
Bus->BroadcastInstancedWithPolicy(Tag, this, Instanced, false, Policy);
```

These return `FTagEventBroadcastResult { Delivered, bWasHandled, bWasStopped }` and honor
`ETagEventBroadcastPolicy`. The plain variants above use `BroadcastAll` behavior and return no
result. How `Policy` and a listener's `ETagEventHandling` combine is described in
[Consuming an event inside a handler](broadcast.md#consuming-an-event-inside-a-handler) and in
[Types and settings](types.md#etageventhandling).

### Deferred

```cpp
template <typename T>
void BroadcastDeferred(FGameplayTag Tag, UObject* Sender, const T& Payload, bool bRetain = false, ...);

void BroadcastSignalDeferred(FGameplayTag Tag, UObject* Sender, bool bRetain = false, ...);

void EnqueueDeferredInstanced(FGameplayTag Tag, UObject* Sender, const FInstancedStruct& Payload,
    bool bRetain, ETagEventBroadcastPolicy Policy = ETagEventBroadcastPolicy::BroadcastAll,
    bool bCoalesce = false, ...);

// In use:
Bus->BroadcastDeferred(Tag, this, Payload);
Bus->BroadcastSignalDeferred(Tag, this);
Bus->EnqueueDeferredInstanced(Tag, this, Instanced, false,
    ETagEventBroadcastPolicy::BroadcastAll, /*bCoalesce=*/true);
```

The payload is **copied on enqueue**, which is what makes these safe to call from inside a handler.
Delivery happens at the next flush — see [Deferred broadcast](broadcast.md#deferred-broadcast) for
when that is.

`Policy` is stored per entry and applied at delivery. The enqueue call does not return the eventual
delivery result. `bCoalesce` is opt-in de-duplication — if an entry for the same tag is already
queued and also marked for coalescing, this call **replaces** its payload, sender, retain flag,
policy, and context in place instead of adding a second entry. Its position in the queue
does not change; the newest payload wins. Entries without the flag never merge.

`RetainTTLSeconds` on a deferred broadcast is armed **at delivery**, not at enqueue — time spent
waiting in the queue does not count against it.

## Listening

```cpp
// Expects a payload of type T; a mismatched type is skipped with a warning
template <typename T>
FTagEventHandle AddListener(FGameplayTag Tag, UObject* Owner, TFunction<void(const FTagEventContext&)> Fn,
    ETagEventMatch Match = ETagEventMatch::Exact, bool bOnce = false, bool bReplaySticky = false,
    int32 Priority = 0, float ThrottleSeconds = 0.f, FGameplayTagQuery Query = FGameplayTagQuery());

// "Signal/any" — catches every payload and every signal
FTagEventHandle AddSignalListener(FGameplayTag Tag, UObject* Owner, TFunction<void(const FTagEventContext&)> Fn,
    ETagEventMatch Match = ETagEventMatch::Exact, bool bOnce = false, bool bReplaySticky = false,
    int32 Priority = 0, float ThrottleSeconds = 0.f, FGameplayTagQuery Query = FGameplayTagQuery());

// In use:
Bus->AddListener<FDamageInfo>(DamageTag, this,
    [this](const FTagEventContext& Ctx){ /* ... */ });
```

`Owner` may be `nullptr`. Such a listener is not swept after garbage collection and lives until you
unbind it by hand — which is a leak unless you meant it.

Both have group-forming twins, `AddListenerToGroup<T>` and `AddSignalListenerToGroup`, which take an
`FTagEventSubscriptionGroup&` as their first argument and put the resulting subscription straight
into it. Same parameters otherwise.

```cpp
bool  Unbind(FTagEventHandle Handle);       // false if it was already gone
int32 UnbindAllForObject(UObject* Owner);   // returns how many were removed

bool  PauseListener(FTagEventHandle Handle);
bool  ResumeListener(FTagEventHandle Handle, bool bReplaySticky = false);
bool  IsListenerPaused(FTagEventHandle Handle) const;

// In use:
Bus->Unbind(Handle);
Bus->UnbindAllForObject(this);
Bus->PauseListener(Handle);
Bus->ResumeListener(Handle, /*bReplaySticky=*/true);
const bool bPaused = Bus->IsListenerPaused(Handle);
```

Pause and resume return `true` only when the state actually changed. A paused listener is skipped
by `Broadcast` without consuming `bOnce` and without updating the delivery counter — see
[Pause is not an unbind](../concepts/lifecycle.md#pause-is-not-an-unbind).

> [!note] The two things these wrappers deliberately do not take
> **A payload predicate.** It is available through the fluent layer (`.When(...)` /
> `.WhenPayload<T>(...)`) and directly through `FTagEventRegistry::AddNativeListener` and
> `FTagEventRegistry::AddNativeHandlingListener`. `Query` is available everywhere instead, because
> it is a declarative filter over tags with no callable to invoke.
>
> **A handler that consumes the event.** Neither the subsystem nor the component has an `AddListener`
> whose callback returns `ETagEventHandling`. That path is
> `FTagEventRegistry::AddNativeHandlingListener`, or `Listen(...).HandleWith(...)` with a callable
> that returns the enum.

## FTagEventContext

The struct handed to every native callback:

```cpp
struct FTagEventContext
{
    FGameplayTag Tag;                    // the tag broadcast — the concrete one, not the listener's
    TWeakObjectPtr<UObject> Sender;      // may be null
    const UScriptStruct* PayloadType;    // nullptr for a signal
    const void* PayloadData;
    FGameplayTagContainer ContextTags;   // the emitter's metadata, WITHOUT the event tag

    template <typename T>
    const T* GetPayload() const;         // nullptr on a missing or mismatched payload
};
```

`GetPayload<T>()` is the whole type check in one call:

```cpp
Bus->AddSignalListener(HitTag, this, [this](const FTagEventContext& Ctx)
{
    if (const FDamageInfo* Dmg = Ctx.GetPayload<FDamageInfo>())
    {
        ApplyDamage(Dmg->Amount, Ctx.Sender.Get());
    }
    // no payload, or a payload of another type => nullptr
});
```

`PayloadData` points to memory that is valid for the duration of the callback. Copy anything you
intend to keep.

## FTagEventSubscription

A handle that unbinds **itself** in its destructor. Keep it as a class member and the subscription
lives exactly as long as the object does, with no manual cleanup on teardown.

```cpp
// class member
FTagEventSubscription HitSub;

void AMyActor::BeginPlay()
{
    Super::BeginPlay();
    if (UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this))
    {
        const FTagEventHandle H = Bus->AddSignalListener(HitTag, this,
            [this](const FTagEventContext&){ /* ... */ });
        HitSub = FTagEventSubscription(Bus->GetRegistry(), H);
    }
}
```

Non-copyable, movable. `Reset()` unbinds early, `IsValid()` reports whether it still holds a live
binding, and `GetHandle()` returns the handle. `Pause()`, `Resume(bool bReplaySticky = false)` and
`IsPaused()` forward to the registry with identical semantics.

The subscription pins a **lifetime token** for the registry before every dereference, so a
destructor that runs after the bus owner is gone is a safe no-op, and `IsValid()` returns `false`
from that moment on.

> [!warning] RAII alone does not keep the log quiet when the owner is collected
> The destructor of a member handle runs when the **engine** destroys the object, at a moment you do
> not control. Under the incremental garbage collector of the editor and the game, the purge is
> deferred by a few frames, while the `PostGarbageCollect` hook — and `SweepDead` under it — runs
> right after reachability analysis, when the weak pointer to the owner is already stale. The sweep
> therefore removes the listener first and writes:
>
> ```
> LogTagEventBusListeners: Warning: SweepDead: removed leaked native listener
>     tag='...' handle=... — owner was GC'd without unbinding
> ```
>
> Nothing breaks: the destructor runs later, calls `RemoveListener`, gets `false` back, and that is
> a state the code expects. The warning is simply misleading, because it reports a leak where the
> cleanup worked.
>
> **This is why `Reset()` in `EndPlay` is still worth writing.** Not for safety — the lifetime token
> already covers that — but so the unbind happens deterministically, before garbage collection is
> involved at all. It is the only thing that removes the warning.

## FTagEventSubscriptionGroup

An RAII container for many subscriptions. Destroying the group, or calling `Reset()`, unbinds all of
them at once; every unbind is guarded by the same token, so a group may safely outlive its bus.
Non-copyable, movable. The API is `Add(FTagEventSubscription&&)`, `Reset()`, `Num()`, `IsEmpty()`,
plus:

```cpp
void Pause();                             // pauses every member
void Resume(bool bReplaySticky = false);  // resumes every member, overriding individual pauses
bool IsPaused() const;                    // the group's own flag, not a poll of its members
```

`Add()` pauses the new member immediately if the group is paused at that moment. `Reset()` clears
the list **and** the pause flag, leaving a fresh group.

```cpp
// class member
FTagEventSubscriptionGroup Subs;

void AMyActor::BeginPlay()
{
    Super::BeginPlay();
    if (UTagEventBusSubsystem* Bus = UTagEventBusSubsystem::Get(this))
    {
        Bus->AddListenerToGroup<FDamageInfo>(Subs, DamageTag, this,
            [this](const FTagEventContext& Ctx){ /* ... */ });
        Bus->AddSignalListenerToGroup(Subs, HitTag, this,
            [this](const FTagEventContext&){ /* ... */ });
    }
}

void AMyActor::EndPlay(const EEndPlayReason::Type Reason)
{
    Subs.Reset();   // deterministic; the destructor would do the same, later
    Super::EndPlay(Reason);
}
```

A `bOnce` listener unbinds itself when it fires and leaves a dead entry in the group. Calling `Reset()`
on a group that still holds such an entry is harmless.

## The fluent layer

An optional ergonomic layer over everything above, in `TagEventBusFluent.h`. It replaces nothing —
every terminal calls the same registry paths.

### Listen

```cpp
FTagEventHandle Handle = Bus->Listen(Tag)
    .Owner(this)                      // owner, for the post-GC sweep
    .MatchPartial()                   // default: Exact
    .Once()
    .ReplaySticky()
    .Priority(TagEventPriority::UI)   // default: 0
    .Expecting<FDamagePayload>()      // default: signal/any
    .WithQuery(FGameplayTagQuery::BuildQuery(...))
    .When([](const FTagEventContext& Ctx){ return true; })
    .Throttle(0.5f)
    .InGroup(Subscriptions)           // hand the RAII subscription to a group
    .HandleWith(this, &ThisClass::OnDamage);   // TERMINAL
```

- `HandleWith(Object, &Class::Method)` makes `Object` the owner when `.Owner()` was not called, and
  captures it weakly — the callback will not fire after the object dies.
- `.When(Pred)` takes `TFunction<bool(const FTagEventContext&)>`. Returning `false` skips **that one
  delivery**: `bOnce` survives, the counter is not updated, exactly like a pause.
- `.WhenPayload<T>(Pred)` is sugar for a typed filter — `Pred` takes a `const T&` and it **implies**
  `Expecting<T>()`. A conflict with an earlier `Expecting<U>()` raises an `ensure` outside Shipping,
  and the type from `WhenPayload` wins.
- `.Throttle(Seconds)` limits this listener to one delivery per window, on the leading edge, in real
  time. A negative value is clamped to `0` and logs a `Warning`.
- **The chain must end in a terminal in the same expression.** A builder destroyed without one
  raises an `ensure` outside Shipping — and no listener is registered.

`HandleWith` also has an overload for a callable or member pointer that **returns**
`ETagEventHandling` instead of `void`. That registers a consuming listener, the same mechanism as
`AddNativeHandlingListener`:

```cpp
Bus->Listen(ValidationTag)
   .Owner(this)
   .Priority(TagEventPriority::Validation)
   .HandleWith(this, &ThisClass::OnValidate);   // ETagEventHandling OnValidate(const FTagEventContext&)
```

The priority bands (`Validation`, `Modifier`, `Default`, `UI`, `Monitor`) are constants in
`TagEventPriority` — see [Broadcast path](../concepts/broadcast-path.md).

### Emit

```cpp
Bus->Emit(Tag)
    .Sender(this)
    .Payload(Data)      // omitted = signal-only; holds a POINTER until the chain ends
    .Retain(5.f)        // sticky; > 0 auto-clears after that many seconds
    .Policy(ETagEventBroadcastPolicy::StopAfterHandled)
    .WithContext(ContextTags)
    .Now();             // TERMINAL: immediate, no copy for native listeners, returns the delivery count
// or
    .NowWithResult();   // TERMINAL: the same, but returns FTagEventBroadcastResult
// or
    .Coalesce()         // only meaningful before Deferred()
    .Deferred();        // TERMINAL: queued, payload copied on enqueue
```

- `.Retain(bool)` is **deleted** on purpose: `Retain(true)` would convert silently to `float` and
  arm a one-second TTL. Write `Retain()` or `Retain(Seconds)`.
- `.Coalesce()` together with `Now()` raises an `ensure` outside Shipping and the flag is ignored —
  an immediate broadcast has no queue to merge into.
- `Deferred()` goes through `EnqueueDeferredInstanced` and costs one extra payload copy compared to
  `BroadcastDeferred<T>`.

## FTagEventRegistry

The core, used by the subsystem and the component and useful in headless tests. Beyond the
equivalents of everything above:

```cpp
FTagEventHandle AddNativeListener(FGameplayTag Tag, UObject* Owner, FTagEventNativeDelegate Fn,
    const UScriptStruct* ExpectedType, ETagEventMatch Match, bool bOnce, bool bReplaySticky,
    int32 Priority = 0, FTagEventNativePredicate Predicate = nullptr,
    float ThrottleSeconds = 0.f, FGameplayTagQuery Query = FGameplayTagQuery());

using FTagEventNativeHandlingDelegate = TFunction<ETagEventHandling(const FTagEventContext&)>;

FTagEventHandle AddNativeHandlingListener(FGameplayTag Tag, UObject* Owner,
    FTagEventNativeHandlingDelegate Fn, const UScriptStruct* ExpectedType, ...);

static bool ConsumeCurrentEvent(ETagEventHandling Handling);
static FGameplayTagContainer GetCurrentEventContextTags();

// Sticky
bool  HasRetained(FGameplayTag) const;   bool GetRetained(FGameplayTag, FInstancedStruct&) const;
void  ClearRetained(FGameplayTag);       void ClearAllRetained();
int32 ExpireRetained();                                     // sweep, called by the owner's ticker
int32 NumRetainedWithTTL() const;                           // O(1) early-out for that sweep
bool  GetRetainedTTLRemaining(FGameplayTag Tag, float& OutSeconds) const;
void  SetClockOverride(TFunction<double()> InClock);        // TEST seam only

// Cleanup
int32 RemoveAllListenersForOwner(const UObject* Owner);
int32 RemoveListenersByScriptDelegate(FGameplayTag Tag, const FTagEventDynDelegate& Delegate);
int32 RemoveListenersForOwnerOnTag(FGameplayTag Tag, const UObject* Owner);
int32 SweepDead();                                          // called after garbage collection
```

`AddNativeHandlingListener` is a separate name rather than an overload of `AddNativeListener` on
purpose: overloading on two `TFunction` types would make a lambda returning the enum ambiguous at
the call site.

`ConsumeCurrentEvent` and `GetCurrentEventContextTags` are both **static and context-free**. They
act on the innermost active broadcast, on whichever bus — the dispatch frame stack is shared across
every registry — so call them from inside a listener body. Outside a delivery they return
`false` / an empty container and log a `Warning`.

When a native listener both returns an enum from its callback and calls `ConsumeCurrentEvent` in its
body, the stronger decision wins: `StopPropagation` > `Handled` > `Continue`. The frame only ever
moves up.

`ExpireRetained()` clears every sticky value whose TTL has expired and returns the number of values
cleared. It is a no-op **during** a broadcast — call it from a tick, never from inside a listener,
because it could free the retained payload the current callback is still reading.
`SetClockOverride` replaces `FPlatformTime::Seconds()` as the TTL clock and exists for tests only;
it does not affect `ThrottleSeconds`, which reads the clock independently.

> [!warning] The predicate contract
> `Predicate` must be **cheap and free of side effects**. It runs for every delivery attempt that
> passes the pause, payload-type, and query gates, including sticky replays. The throttle gate may
> still block delivery afterward. It **must not** mutate the bus it is called from — no bind, no
> unbind, no broadcast.

The gate order on delivery is **paused → payload type → `Query` → `Predicate` → `ThrottleSeconds`**,
and all of it applies to sticky replays too. [Five gates on a single
listener](../concepts/broadcast-path.md#five-gates-on-a-single-listener) walks through it.

The debug surface — `GetListenerDebugInfo`, `GetBroadcasterDebugInfo`, `GetTagStats`,
`SetHistoryRecording`, `SetPayloadCapture`, `OnBroadcastRecorded` — sits behind
`#if !UE_BUILD_SHIPPING` and is what feeds [the debugger panel](../concepts/debugger.md).

## The local bus in C++

`UTagEventBusComponent` carries the same API as the subsystem, scoped to one actor. The differences:

- Deferred delivery flushes on the **component's tick**, enabled automatically on the first deferred
  broadcast and disabled again once the queue is empty **and** no sticky value carries a TTL.
- That same tick runs the sticky TTL sweep.
- `EndPlay` clears the registry and detaches the post-GC hook.
- There is no legacy `BroadcastInstanced` — the component has only `BroadcastInstancedWithPolicy`.
  The asymmetry with the subsystem is deliberate.

> [!warning] Arming a TTL through the raw registry skips the component's tick
> The component's own wrappers call `NotifyRetainTTLArmed()` when a TTL is armed. Going around them
> — `Comp->GetRegistry().Broadcast(..., bRetain = true, TTL > 0)`, or `RestoreRetainedState` on the
> registry directly — leaves the component unaware, so its lazy tick never starts and that entry is
> never swept. Reads stay correct, because a lazy guard still hides an expired value; only the
> cleanup stalls. The fix is one call to `Comp->NotifyRetainTTLArmed()` right after.

> [!warning] Two bus components on one actor are two disjoint registries
> Nothing stops you from adding a second `UTagEventBusComponent`, and the plugin neither checks for
> it nor warns. The same tag then exists as two independent channels, and a listener on one will
> never see a broadcast sent on the other.
>
> The Blueprint facade resolves `Scope = Local` through a single `FindComponentByClass`, so every
> Blueprint node lands on the same one component — and **which** one is undefined, because
> `AActor::OwnedComponents` is a `TSet` and its order comes from hashing, not from the order you
> added things. From C++ there is no ambiguity: you hold a pointer to a specific instance and call
> methods on it directly.
>
> **Keep one local bus per actor.** To separate concerns inside an actor, separate them by tag or by
> `ContextTags` plus `Query` — not by a second component.

## Request-response

The optional `TagEventBusRequests` module supports requests that accept at most one response. Its
entry points:

```cpp
UTagEventRequestSubsystem* Requests = UTagEventRequestSubsystem::Get(this);

FTagRequestHandle Handle = Requests->Request(Tag)
    .Sender(this).Payload(MyPayload).Timeout(2.0f).Owner(this)
    .Then([](const FTagRequestOutcomeContext& Ctx) { /* Completed / TimedOut / Cancelled / NoResponder */ });

// from inside a listener on Tag:
Requests->RespondToCurrent(MyResponse);
FTagRequestResponderHandle Later = Requests->GetCurrentResponder();
```

The model, the outcomes and the reentrancy rules are covered in
[Request-Response](../advanced/request-response.md).

---

← [Reference](README.md) · [Documentation index](../README.md)
