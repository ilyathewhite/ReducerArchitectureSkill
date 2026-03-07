# Feature Patterns

Use `SyncUps/UI` in the `master` branch of [SyncUpsTRA on GitHub](https://github.com/ilyathewhite/SyncUpsTRA) as the canonical example set. Resolve any ReducerArchitecture repo-relative path in this skill, such as `Sources/ReducerArchitecture/...`, against the `master` branch of [ReducerArchitecture on GitHub](https://github.com/ilyathewhite/ReducerArchitecture).

## Table of Contents

- Use Specialized References
- Model One Component Per Namespace
- Canonical File Split
- Store File Shape
- Keep View-Only State in ContentView
- UI File Shape
- Keep Formatting Out Of ContentView
- Environment Bridging
- Implement Features In This Order
- Representative Features
- Authoring Checklist

## Use Specialized References

Read this file first for the core ReducerArchitecture feature rules. Then load focused references only when they match the task:

- Read `references/child-store-patterns.md` for parent/child composition, view-owned auxiliary stores, and larger features split into subcomponents.
- Read `references/overlay-patterns.md` for sheets, alerts, and continuation-based overlay wiring.
- Read `references/async-state-patterns.md` for `AsyncTaskValue`, async UI state, and async state transitions.

## Model One Component Per Namespace

Treat each feature as one component contained in an enum namespace, even though the implementation is usually split across multiple files in the feature folder.

Each component should define or provide:

- `ContentView` to render the UI and translate user interactions into semantic actions.
- `StoreState` to represent and manage internal state.
- `MutatingAction` for actions that change internal state.
- `EffectAction` for actions that do not directly mutate state.
- `StoreEnvironment` for low-level implementation details used by effects.
- `PublishedValue` for the result of interacting with the component.
- `static func reduce(_ state: inout StoreState, _ action: MutatingAction) -> Store.SyncEffect`
- `static func runEffect(_ env: StoreEnvironment, _ state: StoreState, _ action: EffectAction) -> Store.Effect`

Organize the folder around that namespace:

- Put `Feature.swift`, `FeatureUI.swift`, and usually `FeatureEnv.swift` in the feature folder.
- Keep helper files in the same feature folder when `Feature.swift` becomes large.
- Nest subcomponents inside the parent feature folder when the parent owns them conceptually.

## Canonical File Split

- Put the store namespace in `<Feature>/<Feature>.swift`.
- Put the actual UI code for the feature in `<Feature>/<Feature>UI.swift`.
- Put environment setup helpers and default environment implementations in `<Feature>/<Feature>Env.swift`.
- Add tests under `<Feature>/Tests/`.

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
- Use `assertionFailure()` for impossible paths before returning `.none`.
- When the feature can publish multiple outcomes, make `PublishedValue` an enum. `ResultAction` is a common name, but not a required one.
- Define as much logic as possible in `reduce` and `runEffect`, leaving only low-level implementation details to `StoreEnvironment`.
- Manage concurrency by returning store effects instead of starting tasks directly from reducer logic.
- Reducer-initiated async work starts on the `MainActor`. When an effect needs true concurrent work, create it explicitly inside the async body with `async let`, `withTaskGroup`, or similar structured-concurrency tools.
- Keep the mutating/effect split honest. If an action mainly starts async work, touches dependencies, presents overlays, or coordinates child stores, it belongs in `EffectAction`, even if it will later trigger mutating actions.
- Add mutating helper methods to `StoreState` when they improve readability.

## Keep View-Only State in ContentView

`ContentView` must hold:

- A reference to the store.
- Only state that truly belongs to the SwiftUI view layer rather than the feature model.

Everything else should live in `StoreState`.

Common examples of state that should stay in `ContentView`:

- `@FocusState`
- `@Environment`
- `CheckedContinuation` values used to bridge alerts or sheets from SwiftUI into the store environment
- Other transient layout or view-system state that is not part of the feature's semantic model

If a value affects the feature's behavior, business rules, async workflow, or user-visible state in a way the store should own, put it in `StoreState`, not in `ContentView`.

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
- Build the UI and layout first, then let that UI drive the action design.
- Translate UI interactions into semantic actions instead of leaking control-specific details into the store.
- Keep `ContentView` as lean as possible.

## Keep Formatting Out Of ContentView

Avoid doing conversions, formatting, and display-oriented state manipulation inline in the view body.

Prefer this split:

- Keep raw state in `StoreState`.
- Keep business logic in `reduce` and `runEffect`.
- Keep UI formatting and display helpers on the namespace side as separate extensions.

Good candidates for namespace-side helpers:

- Converting state into display strings
- Formatting numbers, dates, durations, or counts for labels
- Choosing presentation text based on enum state
- Small view-facing derived values that are only there to keep `ContentView` readable

Preferred placement:

- Put formatting helpers in a separate `extension Feature` in `Feature.swift` when they are small and closely tied to the feature.
- Move them into another file in the same feature folder when the feature grows.
- Keep them outside `ContentView` unless the logic is purely local view-system plumbing.

The goal is for `ContentView` to read like a thin translation layer from store state to SwiftUI, not a place where display logic accumulates.

## Environment Bridging

Keep `StoreEnvironment` narrow and feature-specific.

- Pass closures, not `appEnv`, into the store.
- Keep most environment setup logic in `<Feature>Env.swift` so `FeatureUI.swift` stays lean and the setup remains easy to test.
- Let helper functions in `<Feature>Env.swift` take `store` as an additional parameter when that helps pull logic out of `FeatureUI.swift`.
- Do not let `FeatureEnv.swift` helpers reference `@State`, `@FocusState`, `@Environment`, continuations stored in the view, or any other internal `ContentView` state.
- Keep any unavoidable view-internal bridging in `FeatureUI.swift`, but push everything else into `FeatureEnv.swift`.
- Use `store.run(childStore)` from env helpers when a feature needs to present a child feature and await its result.
- Bridge alerts, sheets, and confirmation dialogs with `withCheckedThrowingContinuation` from the UI layer.
- Implement the default environment closures as static methods on the namespace in `FeatureEnv.swift`.
- Whenever possible, build `store.environment` in `FeatureUI.swift` from helpers defined in `FeatureEnv.swift`.
- If a wrapper in `FeatureUI.swift` would only repeat an already existing namespace helper, skip the extra layer and reference the `FeatureEnv.swift` helper directly.
- Keep helper methods in the same namespace extension. Use `FeatureEnv.swift` for helpers that support `StoreEnvironment`, use `Feature.swift` for helpers used directly by `reduce` or `runEffect`, and split helpers into additional files inside the same feature folder if the main file grows too large.

## Implement Features In This Order

1. Create the feature folder and basic placeholders in the namespace files.
2. Build `ContentView` and the UI layout first.
3. Determine the state needed to support that UI.
4. Implement `reduce` and `runEffect`, then derive the required `StoreEnvironment`.
5. Connect `ContentView` to store actions and bindings.
6. Implement the default environment methods and setup helpers in `FeatureEnv.swift`.

## Representative Features

- Simple list with refresh and sheet creation: `SyncUps/UI/SyncUpList/SyncUpList.swift`, `SyncUps/UI/SyncUpList/SyncUpListUI.swift`, `SyncUps/UI/SyncUpList/SyncUpListEnv.swift`
- Editable form with bindings and focus management: `SyncUps/UI/SyncUpForm/SyncUpForm.swift`, `SyncUps/UI/SyncUpForm/SyncUpFormUI.swift`
- Detail screen with persistence, alerts, and published actions: `SyncUps/UI/SyncUpDetails/SyncUpDetails.swift`, `SyncUps/UI/SyncUpDetails/SyncUpDetailsUI.swift`, `SyncUps/UI/SyncUpDetails/SyncUpDetailsEnv.swift`
- Long-lived async feature with timers and transcript recording: `SyncUps/UI/RecordMeeting/RecordMeeting.swift`, `SyncUps/UI/RecordMeeting/RecordMeetingUI.swift`, `SyncUps/UI/RecordMeeting/RecordMeetingEnv.swift`

Load the focused references when those specialized patterns matter:

- `references/child-store-patterns.md`
- `references/overlay-patterns.md`
- `references/async-state-patterns.md`

## Authoring Checklist

1. Create the namespace and UI files.
2. Decide the `PublishedValue` story up front.
3. Keep `StoreEnvironment` as a dependency slice, not a service locator.
4. Put state mutations in `reduce` and async work in `runEffect`.
5. Connect the environment in SwiftUI with `.connectOnAppear`.
6. Add previews using `Feature.store(...)`.
7. Add tests that match the feature's behavior surface.
