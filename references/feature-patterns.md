# Feature Patterns

Use `SyncUps/UI` in the `master` branch of [SyncUpsTRA on GitHub](https://github.com/ilyathewhite/SyncUpsTRA) as the canonical example set. Resolve any ReducerArchitecture repo-relative path in this skill, such as `Sources/ReducerArchitecture/...`, against the `master` branch of [ReducerArchitecture on GitHub](https://github.com/ilyathewhite/ReducerArchitecture).

Read this file first for the core ReducerArchitecture feature rules. Load focused references only when they match the task:

- Read `references/child-store-patterns.md` for parent/child composition, view-owned auxiliary stores, and larger features split into subcomponents.
- Read `references/overlay-patterns.md` for sheets, alerts, and continuation-based overlay wiring.
- Read `references/async-state-patterns.md` for `AsyncTaskValue`, async UI state, and async state transitions.

## Component Model

Treat each feature as one component contained in an enum namespace, even when the implementation is split across multiple files. Each component should define or provide:

- `ContentView` to render the UI and translate user interactions into semantic actions.
- `StoreState` to represent and manage internal state.
- `MutatingAction` for actions that change internal state.
- `EffectAction` for actions that do not directly mutate state.
- `StoreEnvironment` for low-level implementation details used by effects.
- `PublishedValue` for the result of interacting with the component.
- `static func reduce(_ state: inout StoreState, _ action: MutatingAction) -> Store.SyncEffect`
- `static func runEffect(_ env: StoreEnvironment, _ state: StoreState, _ action: EffectAction) -> Store.Effect`

Organize the folder around that namespace:

- Put the store namespace in `<Feature>/<Feature>.swift`.
- Put the SwiftUI code in `<Feature>/<Feature>UI.swift`.
- Add `<Feature>/<Feature>Env.swift` only when the feature needs environment helpers or default environment implementations.
- Add `<Feature>/<Feature>State.swift` when state construction or state helpers start crowding `<Feature>.swift`.
- Move helper views into `<Feature>/Views/`, one view per file, when they start crowding `<Feature>UI.swift`.
- Add tests under `<Feature>/Tests/`, and nest subcomponents inside the parent feature folder when the parent owns them conceptually.

## Store File Shape

Follow this structure:

```swift
import ReducerArchitecture

enum Feature: StoreNamespace {
    typealias PublishedValue = ...

    struct StoreEnvironment { ... }
    enum MutatingAction { ... }
    enum EffectAction { ... }
    struct StoreState { ... }
}

extension Feature {
    @MainActor
    static func store(...) -> Store {
        Store(initialState, env: nil)
    }

    @MainActor
    static func reduce(_ state: inout StoreState, _ action: MutatingAction) -> Store.SyncEffect { ... }

    @MainActor
    static func runEffect(_ env: StoreEnvironment, _ state: StoreState, _ action: EffectAction) -> Store.Effect { ... }
}
```

This is the general shape, but you do not need to add methods that ReducerArchitecture already provides by default:

- If `EffectAction == Never`, you can omit `runEffect`.
- If `MutatingAction == Void` and `EffectAction == Never`, you can omit both `reduce` and `runEffect`.

Follow these conventions:

- Add `@MainActor` to `store`, `reduce`, and `runEffect`.
- Return `.none` explicitly from reducers unless a follow-up action is required.
- Trigger side effects from `reduce` with `.action(.effect(...))`.
- Use `.action(.mutating(...), Animation.default)` when you want to animate the complete sequence of synchronous changes, including chained sync effects. Using `.mutating(..., animated: true)` animates only that mutating step.
- Guard invalid state transitions with `guard`, `assertionFailure()`, and `.none`.
- When the feature can publish multiple outcomes, make `PublishedValue` an enum. `ResultAction` is a common name, but not a required one.
- Define as much logic as possible in `reduce` and `runEffect`, leaving only low-level implementation details to `StoreEnvironment`.
- Keep workflow coordination explicit in store actions. After awaiting a dependency or overlay in `runEffect`, prefer sending follow-up mutating/effect actions over directly calling another environment closure when that next step is really feature logic. This keeps the workflow visible in reducer code, testable through store actions, and traceable.
- Manage concurrency by returning store effects instead of starting tasks directly from reducer logic.
- Reducer-initiated async work starts on the `MainActor`. When an effect needs true concurrent work, create it explicitly inside the async body with `async let`, `withTaskGroup`, or similar structured-concurrency tools.
- Keep the mutating/effect split honest. If an action mainly starts async work, touches dependencies, presents overlays, or coordinates child stores, it belongs in `EffectAction`, even if it will later trigger mutating actions.
- Add mutating helper methods to `StoreState` when they improve readability, and move larger state helpers into `<Feature>State.swift`.

## Keep View-Only State in ContentView

`ContentView` should hold the store plus only true SwiftUI-layer state such as `@FocusState`, `@Environment`, `CheckedContinuation` values used to bridge overlays, and other transient layout or view-system state. If a value affects feature behavior, business rules, async workflow, or user-visible state in a way the store should own, put it in `StoreState`.

## UI File Shape

Follow this structure:

```swift
import SwiftUI
import ReducerArchitecture

extension Feature: StoreUINamespace {
    struct ContentView: StoreContentView {
        typealias Nsp = Feature
        @ObservedObject var store: Store

        init(_ store: Store) {
            self.store = store
        }

        var body: some View { ... }
    }
}
```

Follow these conventions:

- Send user-driven state changes with `store.send(.mutating(...))`.
- Start side effects with `store.send(.effect(...))`.
- Prefer finishing child interactions with `store.publish(...)` or `store.cancel()`.
- Build form controls with `store.binding(...)` when a simple key-path mapping exists.
- Model sheets and pushed child stores as `StoreUI<ChildFeature>?` built from `store.child()`.
- Guard `.onAppear` effect sends behind `store.environment != nil` in previews or render-time setup races.
- Wire dependencies inside `.connectOnAppear { store.environment = ... }`, preferably by composing helpers from `FeatureEnv.swift`.
- Prefer reducer-driven effects over observing `store.state` in SwiftUI views when coordinating feature behavior.
- Translate UI interactions into semantic actions instead of leaking control-specific details into the store.

## Split Helper Views Across Files

If `FeatureUI.swift` only needs `ContentView` and a couple of short helper views, keeping them together is fine. Once it starts to sprawl, move feature-owned helper views into `<Feature>/Views/`, one view per file, and keep them under `extension Feature` rather than as free-floating file-private types.

## Keep Formatting Out Of ContentView

Avoid doing conversions, formatting, and display-oriented state manipulation inline in the view body. Keep raw state in `StoreState`, business logic in `reduce` and `runEffect`, and display helpers on the namespace side. Good candidates include display strings, number/date/duration formatting, presentation text chosen from enums, and small derived values that keep `ContentView` readable. Put small helpers in a separate `extension Feature`, move them into another file when the feature grows, and keep them out of `ContentView` unless the logic is purely local view-system plumbing.

## Environment Bridging

Keep `StoreEnvironment` narrow and feature-specific.

- Pass closures, not `appEnv`, into the store.
- Use `<Feature>Env.swift` for environment helpers and default implementations, but skip wrappers whose only job is to restate `StoreEnvironment(...)`.
- Let env helpers take `store` when that pulls logic out of `FeatureUI.swift`, but do not let them reference internal `ContentView` state such as `@State`, `@FocusState`, `@Environment`, or stored continuations.
- Keep unavoidable view-internal bridging in `FeatureUI.swift`. If the default `StoreEnvironment(...)` initializer is already clear at the call site, build it inline in `.connectOnAppear` and reference helpers as needed.
- If a wrapper in `FeatureUI.swift` would only repeat an existing namespace helper, skip the extra layer and reference the helper directly.
- Use `store.run(childStore)` from env helpers when a feature needs to present a child feature and await its result.
- Bridge alerts, sheets, and confirmation dialogs with `withCheckedThrowingContinuation` from the UI layer.
- Implement default environment closures as static namespace methods in `FeatureEnv.swift`.
- Prefer direct function references such as `someDependency: Nsp.someDependency` when possible; use forwarding closures only when you need adaptation or captures such as `[weak store]`.
- Keep helper methods in the same namespace extension: use `FeatureEnv.swift` for environment support, `Feature.swift` for helpers used directly by `reduce` or `runEffect`, and split them into additional feature files only when needed.

## Implement Features In This Order

1. Create the feature folder and basic placeholders in the namespace files.
2. Build `ContentView` and the UI layout first.
3. Determine the state needed to support that UI.
4. Implement `reduce` and `runEffect`, then derive the required `StoreEnvironment`.
5. Connect `ContentView` to store actions and bindings.
6. Implement environment-related namespace helpers in `FeatureEnv.swift` when needed.

## Representative Features

- `SyncUpList` for list + refresh + sheet creation.
- `SyncUpForm` for bindings + focus management.
- `SyncUpDetails` for persistence + alerts + published actions.
- `RecordMeeting` for long-lived async work + timers + transcript recording.

Load the focused references when those specialized patterns matter:

- `references/child-store-patterns.md`
- `references/overlay-patterns.md`
- `references/async-state-patterns.md`
