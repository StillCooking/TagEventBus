# Diagnostics

*Procedures for diagnosing problems with no obvious cause: subscription leaks, local scope, and marshaling onto the Game Thread.*

## A subscription leak

Four steps, cheapest first: <!-- fact:wyciek-subskrypcji-diagnozuje-sie-w-czterech-krokach-pasek-p -->

1. **The summary bar** of the panel gives the `DEAD` count.
2. **Overview** shows which tags are affected — a red status and `(dead)` next to the owner.
3. **The log** provides the details in this message:

   ```log
   SweepDead: removed leaked ... — owner was GC'd without unbinding
   ```

4. **Force unbind** clears them out for the moment.

The fourth step is a stopgap; the root cause is still there in the code. The `Warning` the sweep
writes is described under [Lifecycle](../concepts/lifecycle.md).

> [!warning]
> **A growing listener count without a growing `DEAD` count is a different problem** — and it has
> a different fix. See [Common problems](common-problems.md).

## `Local` scope

The panel is no help here: it sees the global bus alone, as the
[Debugger](../concepts/debugger.md) page explains. Diagnosing `Local` scope comes down to two
things:

- **`Has Matching Listeners`** called on the same actor with the same tag. It returns what a
  broadcast would actually consider — the exact channel plus `Partial` on ancestors. The
  version without `Matching` answers a narrower question and, where tags form a hierarchy,
  can mislead you.
- **The logs**, `LogTagEventBusListeners` and `LogTagEventBusBroadcast` in particular.

The most common cause of silence in local scope is simpler than it looks: the actor has no
bus component, and only a bind can create one for you —
[Architecture](../concepts/architecture.md).

## Sender or receiver?

A manual broadcast from the panel's **Inject** tab settles this without touching the graph:
it delivers on the tag in question and tells you how many listeners it reached. If it reaches
one, the receiver works and the sender is the suspect. The tab, and how to read its reply, are
on the [Debugger](../concepts/debugger.md) page.

Two panel behaviors get mistaken for a fault while you are doing this: an event that shows up
in Overview but not in Live, and a payload row that opens empty. Both are switches rather than
faults — [Debugging](../advanced/debugging.md) for the first,
[Debugger](../concepts/debugger.md) for the second.

## Marshaling onto the Game Thread

If an event originates on a worker thread, you have to move the broadcast onto the Game <!-- fact:marshalling-na-game-thread-wymaga-trzech-ostroznosci-sprawdz -->
Thread. Three precautions:

1. **Check `Get()` again inside the lambda.** The `GameInstance` may no longer exist when the scheduled work runs.
2. **Do not capture raw `UObject` pointers.** Use `TWeakObjectPtr`, or copy the data by
   value.
3. **Copy the payload; do not hold a reference to it.** The lambda may run after the
   context it was created in has ended.

Do not rely on the `ensure` warning to enforce the "Game Thread only" contract —
[Limits and threading](../advanced/limits-and-threading.md) says why.

## When to reach for the C++ debugger

Only at the end, and when you do, put the breakpoint where every delivery passes through —
[Architecture](../concepts/architecture.md) names that place and the one listener
registration goes through.

---

← [Troubleshooting](README.md) · [Documentation index](../README.md)
