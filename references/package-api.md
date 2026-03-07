# Package API Notes

Use this file when you need the exact ReducerArchitecture concepts or want a quick reminder of which helper to reach for.

## Core Protocols

Resolve every ReducerArchitecture repo-relative path in this file against the `master` branch of [ReducerArchitecture on GitHub](https://github.com/ilyathewhite/ReducerArchitecture).

Canonical definitions live in:

- `Sources/ReducerArchitecture/StateStore.swift`
- `Sources/ReducerArchitecture/StateStore+SwiftUI.swift`

Important pieces:

- `StoreNamespace` requires `StoreEnvironment`, `StoreState`, `MutatingAction`, `EffectAction`, `PublishedValue`, `reduce`, and `runEffect`.
- `StoreUINamespace` pairs a namespace with a `ContentView` and exposes `StoreUI`.
- `StoreContentView` gives the SwiftUI view a typed `store`.
- `typealias Store = StateStore<Self>` is provided automatically through the namespace extension.

## Action and Effect Surface

`StateStore.Action` separates responsibilities:

- `.mutating(...)` changes state.
- `.effect(...)` starts side effects.
- `.publish(...)` returns a value to a parent or caller.
- `.cancel` tears down the store and cancels children/effects.

Reducer return types:

- `Store.SyncEffect.action(...)`
- `Store.SyncEffect.actions(...)`
- `Store.SyncEffect.none`

Effect helpers:

- `.action(...)` and `.actions(...)` for immediate follow-up work.
- `.asyncAction(...)` for a single async result.
- `.asyncActionLatest(key: ...)` when newer work should replace older work.
- `.asyncActionSequence { send in ... }` for streams, timers, and dialog-style multi-step interactions.
- `.asyncActionSequenceLatest(key: ...)` when only the latest long-lived task should remain active.
- `.publisher(...)` when adapting a Combine pipeline.

Reducer-initiated async work begins on the `MainActor`. If an effect needs real concurrent work, create it explicitly inside the async body with structured-concurrency tools such as `async let` or `withTaskGroup`.

## SwiftUI Helpers

- Use `store.binding(\.path, { .mutation($0) })` for simple editable fields.
- Use `store.contentView` when embedding a store directly.
- Use `StoreUI<Feature>` when storing typed child UI.
- Use `store.child()` in UI/tests to read the current child store when the feature presents another store.
- Use `.connectOnAppear` to install `store.environment` once the view has enough context.

## Child Store Helpers

For parent/child composition, the most useful helpers are:

- `store.addChild(...)` and `store.addChildIfNeeded(...)` to install child stores owned by the parent.
- `store.removeChild(...)` when a child should be torn down explicitly.
- `targetStore.bind(to: sourceStore, on: \.someState) { ... }` to turn observed state changes into target actions.
- `targetStore.bindPublishedValue(of: sourceStore) { ... }` to turn observed published results into target actions.

In parent/child composition, the most common use is the parent observing child state or published values, but parent -> child binding is also supported for simple mirrored input.

Reach for these helpers before collapsing child state into the parent store directly.

## Publish and Cancel

These methods are intentionally thin wrappers on the store action API:

- `store.publish(value)` sends `.publish(value)`.
- `store.cancel()` sends `.cancel`.

Use them in UI code or preview/test helpers instead of constructing those actions manually.

## Architectural Reminders

- Effects are tied to store lifetime and cancel when the store tears down.
- Parent stores know about child stores; child stores do not know about parent state.
- This package is intentionally not TCA. Preserve the mutating/effect split and the component-driven architecture.
