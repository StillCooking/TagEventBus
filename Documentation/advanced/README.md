# Advanced

*The two optional modules, the cost of each path, how the bus works alongside other systems, and its limitations.*

The things you do not do on day one. The first two pages cover the optional modules — you
can skip both modules entirely and the bus behaves identically.

The remaining pages answer the questions that come up as a project grows:

- what it costs,
- how it works alongside GAS,
- how to see what the bus is doing,
- and what this release does not include.

## Pages in this section

| Page | What it covers |
| --- | --- |
| [Contracts](contracts.md) | Specifying a tag's required payload struct, four enforcement levels, what validation does and does not check, and one trap tied to the tag hierarchy. |
| [Request-Response](request-response.md) | A request and a response on the same bus: four outcomes, an immediate or a delayed answer, a timeout, and cancellation. |
| [Optimization](optimization.md) | Where the costs come from on the hot path: tag depth, payload copying, the filter gates, and the sticky sweep. |
| [Interoperability](interoperability.md) | The division of labor between the bus, GAS, and Event Dispatchers, the network boundary, module dependencies, and a trap with wildcard nodes after an update. |
| [Debugging](debugging.md) | Eight log categories, four console commands, and a table showing what is available in each build configuration. |
| [Limits and threading](limits-and-threading.md) | What the bus does not do by design, what is not implemented yet, and what disappears in a Shipping build. |
| [Recipes](recipes.md) | Ready-made patterns for the situations that come up most often, in both Blueprint and C++. |

← [Documentation index](../README.md)
