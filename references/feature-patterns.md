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
- Move helper views into `<Feature>/<Feature>Views/`, one view per file, when they start crowding `<Feature>UI.swift`.
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
- Keep feature behavior in `reduce` and `runEffect`, leaving only low-level implementation details to `StoreEnvironment`. Views must not decide or coordinate that behavior.
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

- UI event handlers only dispatch semantic store actions: use `store.send(.mutating(...))` for state changes and `store.send(.effect(...))` for side effects. Put validation, branching, calculations, and follow-up work in the store rather than in button, gesture, or control callbacks.
- Reusable presentation views may expose event closures; feature call sites must wire those closures to store actions without adding feature logic.
- Prefer finishing child interactions with `store.publish(...)` or `store.cancel()`.
- Build form controls with `store.binding(...)` when a simple key-path mapping exists.
- Model sheets and pushed child stores as `StoreUI<ChildFeature>?` built from `store.child()`.
- Wire dependencies inside `.connectOnAppear { store.environment = ... }`, preferably by composing helpers from `FeatureEnv.swift`.
- Use `.connectOnAppear` for first actions that depend on the store being connected. Set `store.environment` first, then send the initial mutating/effect actions in that same closure.
- Use `.onAppear` only for additional view lifecycle work that cannot be part of the one-time connection. If it sends store actions and the store has an environment, guard those sends with `store.environment != nil` so previews and render-time setup races do not run effects before the store is ready.
- Do not use `.onChange()` in views for logic or animations, even when its callback only sends another store action. Return follow-up actions from reducers/effects, or use store bindings for cross-store observation.
- Translate UI interactions into semantic actions instead of leaking control-specific details into the store.

## Animate Through Store Actions

Every user-initiated animation must originate from an animated store action. Choose the action animation API described above according to whether the animation covers one mutation or the complete synchronous action sequence. An interaction that should update immediately should still send a store action without animation; this rule does not require animating every interaction.

Views may declare transitions and other rendering details that consume the action's animation transaction. Do not wrap event handlers in `withAnimation`, attach `.animation(..., value:)` to initiate user-driven state animations, or use `.onChange()` to start or coordinate them. Any asynchronous follow-up state change that should animate needs its own animated action.

Keep unavoidable SwiftUI bridges limited to reporting events or typed results back to the store. For example, an animation completion reports a completion action; the store decides whether it is still relevant and what happens next. The typed alert continuation pattern in `overlay-patterns.md` similarly returns a result to an awaiting store effect without making workflow decisions in the view.

## Compose Small Views And Split UI Files

Build screens from small, focused `View` types with clear inputs and event outputs. Extract meaningful sections, rows, and controls into their own views instead of accumulating computed `some View` properties, `@ViewBuilder` functions, or other UI-building methods inside a large `ContentView`. Simple store accessors and typed child-store presentation properties remain wiring, not substitutes for extracting UI sections. Use a child store when the extracted component owns feature behavior; a presentation-only view does not need its own store.

Keep UI files small as well as individual views. If `FeatureUI.swift` only needs `ContentView` and a couple of short helper views, keeping them together is fine. Once the file grows difficult to scan, move feature-owned helper views into `<Feature>/<Feature>Views/`, one view per file, and keep them under `extension Feature` rather than as free-floating file-private types. Treat file growth and multiple UI responsibilities as extraction signals; there is no fixed line-count threshold, and merely moving a large body into same-file properties or functions does not satisfy this rule.

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
