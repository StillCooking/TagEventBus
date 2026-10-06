# Concepts

*The conceptual model of the bus: two disjoint registries, a fixed delivery path, and cleanup responsibilities and timing.*

Four things help explain why an event did not arrive: which bus was targeted, the order in which listeners are
considered, what can prevent delivery or stop further propagation, and who unbinds a listener when its owner
dies.

Read the pages in this group in order. The API reference is separate — this group is about
how the bus works, not about exactly what a given pin does.

## Pages in this section

| Page | What it covers |
| --- | --- |
| [Overview](overview.md) | What the plugin is made of, which modules are optional, and what sets it apart from free tag-routing alternatives. |
| [Architecture](architecture.md) | The global bus and the local bus are two disjoint registries, not one registry with a filter — and that is the first thing people get wrong. |
| [Broadcast path](broadcast-path.md) | Five broadcast steps, five gates for each listener, the priority bands, and a fixed order that new options do not change. |
| [Lifecycle](lifecycle.md) | Listener cleanup: managed bindings, explicit unbinding, and the safety nets that catch what remains. |
| [Error handling](error-handling.md) | Four mechanisms for signaling errors, seven named failure reasons, and what stays silent on purpose. |
| [Debugger](debugger.md) | The in-editor panel shows who broadcasts and who receives events — and the Inject tab lets you test a receiver before the sender exists. |

← [Documentation index](../README.md)
