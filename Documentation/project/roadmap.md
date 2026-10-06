# Roadmap

*Features deliberately deferred and one design decision still to be made — with no dates.*

## What this page is not

**There are no dates here.** An entry on this list means "designed so it can be added later",
not "in the next release".

What is missing by design, and will stay that way, lives elsewhere:
[Limits and threading](../advanced/limits-and-threading.md).

## Deferred features

Seven features are deliberately deferred to future versions and **designed so that adding <!-- fact:siedem-funkcji-jest-swiadomie-odlozonych-do-przyszlych-wersj -->
them later will not break the API**:

| Feature                                            | What it covers                                                                                  |
|----------------------------------------------------|-------------------------------------------------------------------------------------------------|
| scatter-gather in requests                         | collecting answers from several responders at once, rather than just from the first responder   |
| `Reject(reason)` error channel                     | a negative answer with a reason, alongside `Completed` and `TimedOut`                           |
| `Local` scope for requests                         | request-response on an actor's local bus                                                        |
| notification when a sticky value expires           | today a TTL expiry is silent; the only trace is a `Verbose` log entry                           |
| TTL measured in game time                          | currently, TTL is measured in real time, so it is unaffected by game pause or time dilation     |
| detecting deprecated replacement tags in contracts | today validation does not check whether `ReplacementTag` is itself deprecated                   |
| detecting duplicates across schemas                | today duplicates are detected only inside a single schema                                       |

Two of them have consequences you can observe today:

- When a sticky value expires, no event is sent — code that needs to react to "the value no longer applies" has to check for that itself. <!-- fact:ttl-sticky-liczy-czas-realny-fplatformtime-seconds-ten-sam-z -->
- Two schemas can declare a contract for the same tag with no validation error; at runtime <!-- fact:duplikaty-kontraktow-sa-wykrywane-tylko-wewnatrz-jednego-sch -->
  the first schema loaded wins.

## The unresolved design question

**Extending the debugger panel to local buses.** Currently, the panel shows only the global bus.

This is not a question of effort but an unresolved design problem: how to present N <!-- fact:rozszerzenie-panelu-debuggera-o-busy-lokalne-jest-otwarta-de -->
independent registries in one tree, and how to pick which one you are looking at. The
change concerns the interface, not the core.

Until that is settled, diagnosing `Local` scope comes down to the logs and the
`Has Matching Listeners` node —
[Diagnostics](../troubleshooting/diagnostics.md).

---

← [Versions, support, and license](README.md) · [Documentation index](../README.md)
