# Changelog

All notable changes to the **TagEventBus** plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] — 2026-10-05

First release. Unreal Engine 5.5 – 5.8, Win64.

### Added

- `FGameplayTag`-addressed event bus with a global scope (`UTagEventBusSubsystem`) and a
  local, per-actor scope (`UTagEventBusComponent`).
- C++ API: typed `AddListener<T>` / `Broadcast` / `Unbind`, RAII subscriptions
  (`FTagEventSubscription`, `FTagEventSubscriptionGroup`) and a fluent `Listen(...)` /
  `Emit(...)` builder.
- Blueprint API: Bind / Broadcast / Unbind nodes with struct payloads, `Try*` variants
  returning `ETagEventResult`, and an async `Listen For Tag Event` node.
- Sticky (retained) payloads with replay on bind, plus versioned save-state capture and
  restore.
- Deferred broadcasts flushed on the Game Thread, with optional keep-latest coalescing.
- Delivery control: event consumption and stop-propagation (`ETagEventHandling`,
  `ETagEventBroadcastPolicy`), per-listener payload predicates, gameplay tag query filters
  with broadcast context tags, and a leading-edge throttle.
- Editor debugger with Overview, Live, Inject and Contracts tabs; panel state persists per
  user.
- Per-section logging with Project Settings and `TagEventBus.Log.*` console commands.
- `TagEventBusContracts` (optional): declarative `tag → payload type` contracts authored as
  Data Assets, validated per broadcast outside Shipping (Off / Warn / Ensure / Block). Every
  level but Off logs each violation as a `Warning` naming the rule, the expected and actual
  payload types, the scope, the sender and the level; `Ensure` also trips `ensureMsgf`.
- `TagEventBusRequests` (optional): request-response over the bus, first responder wins,
  with timeout and cancellation, and an async `Request Tag Event` node.
- Documentation in `Documentation/`, also published at
  <https://stillcooking.dev/en/products/tageventbus/docs/>.
