# Error handling

*Four mechanisms for signaling errors, seven named failure reasons, and what stays silent on purpose.*

## Signalling a problem

The plugin uses four mechanisms to signal problems, depending on the situation and the <!-- fact:plugin-sygnalizuje-bledy-przez-cztery-mechanizmy-dobrane-do -->
information the caller needs:

| Mechanism | When                                                                       |
| --- |----------------------------------------------------------------------------|
| return value | a predictable error, handled in the normal flow                            |
| `ETagEventResult` | the caller needs to know why the operation failed, not just that it failed |
| `Warning` in the log | a situation that is probably unintended                                    |
| `ensureMsgf` | a programmer breaking the API contract                                     |

## The `Try*` variants

The `Try*` variants report an `ETagEventResult` through the `Result` output parameter. <!-- fact:warianty-try-rozrozniaja-osiem-przyczyn-niepowodzenia-invali -->
Alongside `Success`, they distinguish seven failure reasons:

`InvalidTag`, `InvalidWorldContext`, `LocalActorNotResolved`, `LocalBusMissing`,
`DelegateNotBound`, `InvalidHandle`, `PayloadTypeMismatch`.

`PayloadTypeMismatch` is reserved for a later version: no function returns it yet. A broadcast
that a contract blocks reports `Success` with zero deliveries — see [Contracts](../advanced/contracts.md).

The usual fixes: <!-- fact:kazda-wartosc-etageventresult-ma-typowa-naprawe-np-invalidta -->

| Result | What to do |
| --- | --- |
| `InvalidTag` | register the tag in the project's tag list |
| `LocalBusMissing` | add the bus component, or set `bCreateLocalIfMissing` |
| `InvalidHandle` | check whether you are unbinding the same handle twice |

**Use the `Try*` variants while integrating, and the plain ones once everything works**. <!-- fact:warianty-try-warto-uzywac-podczas-integracji-a-zwyklych-wari -->

## What stays silent on purpose

This is the part that most often looks like a bug in the plugin and is not one.

**A tag outside the project's tag list.** Rejected on bind and on broadcast alike. The <!-- fact:niskopoziomowe-ftageventregistry-broadcast-wywolane-bezposre -->
Blueprint nodes log a `Warning`, but the low-level C++ API rejects it silently.
Called directly with an invalid tag, `FTagEventRegistry::Broadcast` returns `0` and logs
nothing; logging on that hot path would be a real cost, so the `Warning` comes from the
Blueprint facade and the wrappers.

Skipping delivery because a listener is paused, a `Query` does not match, a predicate returns `false`, or a throttle window is closed **logs nothing** — logging it would be spam <!-- fact:pominiecie-dostarczenia-przez-pauze-niedopasowane-query-pred -->
on the hot path. The state of all five gates is instead visible on the listener card in the
panel. <!-- fact:enqueue-wykonane-w-trakcie-flushu-deferred-trafia-na-kolejny -->

**Deferred broadcast.** The node does not return a delivery count for deferred delivery. An
event enqueued during a flush is processed in the next flush.

**A sticky value reaching its TTL.** Silent: no event goes out to the listeners, and the only
trace is a `Verbose` log entry.

> [!warning]
> A missing delivery does not always produce an error message. For the global bus, a manual
> broadcast from the Inject tab can help narrow down the cause. Check the delivery count and
> whether the intended receiver received the event:
> [Verification](../start/verification.md).

## What is not an error, although it looks like one

- **"Start Play In Editor to see live data."** in the panel outside a PIE session — correct
  behavior.
- **Channels awaiting removal in the results of `GetChannelCount()` or `GetActiveTags()` <!-- fact:odczyt-getchannelcount-getactivetags-w-trakcie-trwajacego-br -->
  during a broadcast.** These channels can temporarily remain in the count or tag list
  because removal is deferred; this is not a leak.

## Breaking the API contract

Every public function that mutates the registry contains
`ensureMsgf(IsInGameThread(), ...)`. There are no locks and no atomics, so **concurrent
access is undefined behavior**.
The `ensure` tells you about it in a development build. In Shipping it says nothing, and the
problem remains.

---

← [Concepts](README.md) · [Documentation index](../README.md)
