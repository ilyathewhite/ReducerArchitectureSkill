---
name: reducer-architecture
description: Use for any request related to `ReducerArchitecture` or TRA, or whenever the codebase uses the `ReducerArchitecture` package.
---

# Reducer Architecture

Mirror the conventions in the SyncUpsTRA reference app unless the target codebase already has a narrower local pattern. Resolve any `SyncUps/...` example path in this skill against the `master` branch of [SyncUpsTRA on GitHub](https://github.com/ilyathewhite/SyncUpsTRA). Resolve any ReducerArchitecture repo-relative path in this skill, such as `Sources/ReducerArchitecture/...`, against the `master` branch of [ReducerArchitecture on GitHub](https://github.com/ilyathewhite/ReducerArchitecture). Prefer the package's existing store and effect APIs over importing TCA-style conventions or inventing new abstractions.

## Quick Start

- Read `references/feature-patterns.md` before adding or refactoring a feature. It contains the core component anatomy, file split, reducer rules, view-state boundary, and environment-bridging guidance.
- Read `references/child-store-patterns.md` when the feature uses child stores, view-owned auxiliary stores, or parent/child composition.
- Read `references/overlay-patterns.md` when the feature uses sheets or alerts.
- Read `references/async-state-patterns.md` when the feature uses `AsyncTaskValue` or other async UI state patterns.
- Read `references/testing-patterns.md` before adding tests or reviewing coverage.
- Read `references/package-api.md` when you need the exact ReducerArchitecture surface area or template shape.

## Follow These Rules

- Split each screen into `<Feature>.swift` for the `StoreNamespace` and `<Feature>UI.swift` for the actual UI code for the feature. Use `<Feature>Env.swift` for environment-related namespace helpers and default environment implementations when the feature needs them. If it would otherwise be empty, do not create it.
- When `<Feature>UI.swift` grows because of helper views, move those views into a `Views/` subfolder under the same namespace and put each helper view in its own file there. Do not introduce a shared `<Feature>Views.swift` file.
- Treat each feature as one component/namespace containing `ContentView`, `StoreState`, `MutatingAction`, `EffectAction`, `StoreEnvironment`, `PublishedValue`, `reduce`, and `runEffect`, even though the UI and env helpers usually live in separate files.
- Create stores with `env: nil`, then wire `store.environment` in `.connectOnAppear`. If the default `StoreEnvironment(...)` initializer reads clearly at the call site, prefer writing that initializer inline and referencing helpers from `<Feature>Env.swift` as needed instead of adding a wrapper whose only job is to construct `StoreEnvironment`.
- Keep only true view-system state in `ContentView`, such as `@FocusState`, `@Environment`, continuations owned by the view, and other transient UI-only values. Everything else should live in `StoreState`.
- Keep views as lean as possible. Move formatting, derived display text, and state-to-UI conversion logic onto the namespace side instead of embedding it inline in `ContentView`.
- Keep synchronous state changes in `reduce`.
- Keep dependency calls, async work, alerts, sheets, timers, and persistence in `runEffect`.
- Use `MutatingAction` only for actions that directly change `StoreState`. If an action mainly starts async work, touches dependencies, presents overlays, coordinates child stores, or otherwise does not directly mutate state, make it an `EffectAction`.
- Model parent-facing outputs with `PublishedValue`. When there is more than one possible outcome, make `PublishedValue` an enum.
- Return `.action(.effect(...))` from `reduce` when a mutation should trigger follow-up work.
- Manage concurrency through `.asyncAction`, `.asyncActions`, `.asyncActionLatest`, `.asyncActionSequence`, and `.asyncActionSequenceLatest` instead of starting tasks directly from reducer logic.
- Reducer-initiated async work starts on the `MainActor`. When an effect needs true concurrent work, create it explicitly inside the async body with tools such as `async let` or `withTaskGroup`.
- Prefer `store.publish(...)` to emit a result and `store.cancel()` for dismissal/cancel paths.
- Prefer direct function references such as `liveUpdates: Nsp.liveUpdates` when a closure field already matches an existing namespace method. Use forwarding closures only when adapting arguments, capture semantics, or actor isolation.
- Guard invalid state transitions with `guard`, `assertionFailure()`, and `.none`.
- Use `store.binding(...)` for simple SwiftUI bindings and explicit `store.send(.mutating(...))` for custom updates.
- Treat `bind` as a source-to-target bridge. Most parent/child bindings should flow child -> parent so the parent can react to child state or published values.
- Parent -> child binding is also supported for simple mirrored input. Prefer an explicit parent reducer effect plus environment closure when the synchronization is really parent-owned coordination, should stay visible in reducer logic, or could create a confusing feedback loop.

## Workflow

1. Create the component folder and basic placeholders in the namespace's files.
2. Build `ContentView` first so the user interactions reveal the actions the store must support.
3. Add the `StoreState` needed to drive that UI.
4. Implement `reduce` and `runEffect`, then derive the `StoreEnvironment` surface from the low-level work still missing.
5. Connect the UI to the store actions and bindings.
6. Implement environment-related namespace helpers in `<Feature>Env.swift` when needed, then use them from `FeatureUI.swift`. Skip the file if there are no such helpers.
7. Compare the result against the closest SyncUpsTRA GitHub example and add matching tests.

## Avoid These Mistakes

- Do not call app dependencies from `reduce`.
- Do not create stores with fully wired environments at arbitrary call sites unless the target already does that consistently.
- Do not make an action mutating when it actually represents effect work.
- Do not bypass `publish` / `cancel` and binding patterns without a reason. Prefer them over ad hoc callbacks, but use callbacks occasionally when they are the simplest correct tool.
- Do not move persistence into random view callbacks when an env helper or reducer effect should own it.
