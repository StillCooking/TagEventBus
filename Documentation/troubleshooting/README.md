# Troubleshooting

*The order in which you use the tools here is the reverse of the usual debugging workflow: the panel first, then the logs, and the C++ debugger last.*

> [!tip]
> **Start with the panel, not with the C++ debugger.** A plain call stack shows only half the <!-- fact:kolejnosc-narzedzi-diagnostycznych-jest-odwrotna-niz-w-zwykl -->
> story, because the sender and the receiver do not know each other — you see who broadcast
> the event but not who was supposed to receive it, or the other way around.

The order: panel → logs → C++ debugger.

The [Common problems](common-problems.md) page starts with the symptom.
[Diagnostics](diagnostics.md) provides procedures for problems with no obvious cause.

## Pages in this section

| Page | What it covers |
| --- | --- |
| [Common problems](common-problems.md) | From symptom to cause — silence after a broadcast, a linker error, a missing menu entry, and a queue that never empties. |
| [Diagnostics](diagnostics.md) | Procedures for diagnosing problems with no obvious cause: subscription leaks, local scope, and marshaling onto the Game Thread. |

← [Documentation index](../README.md)
