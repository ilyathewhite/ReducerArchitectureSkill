# Testing Patterns

ReducerArchitecture components should use three primary test kinds:

- model tests
- user event tests
- overlay tests, when the component has overlays such as sheets or alerts

Match the test kind to the behavior you are changing instead of forcing everything through one harness.

Resolve any `SyncUps/...` path in this file against the `master` branch of [SyncUpsTRA on GitHub](https://github.com/ilyathewhite/SyncUpsTRA). Resolve any ReducerArchitecture repo-relative path in this file, such as `Sources/ReducerArchitecture/...`, against the `master` branch of [ReducerArchitecture on GitHub](https://github.com/ilyathewhite/ReducerArchitecture).

## Put Tests Next To The Component

Store tests under the component's own `Tests` folder so they are easy to find while editing the feature.

Examples:

- `SyncUps/UI/SyncUpList/Tests/SyncUpListModelTests.swift`
- `SyncUps/UI/SyncUpList/Tests/SyncUpListUserEventTests.swift`
- `SyncUps/UI/SyncUpList/Tests/SyncUpListOverlayTests.swift`

Keep the tests near the feature they cover instead of collecting all feature tests in one shared top-level test folder.

## Global Test Suites

Define global test suites once, then let each component extend the appropriate suite.

Reference structure from SyncUpsTRA:

```swift
import Testing

@Suite
struct ModelTests {}

@Suite(.serialized)
struct EventTests {
    @Suite @MainActor struct UserEventTests {}
    @Suite @MainActor struct OverlayTests {}
    @Suite @MainActor struct IntegrationTests {}
}
```

Use this structure as follows:

- Model tests can run in parallel.
- User event tests must run sequentially.
- Overlay tests must run sequentially.
- `IntegrationTests` can exist as a shared serialized suite even when a specific component only needs the first three test kinds.

Per-component suite pattern:

```swift
extension ModelTests {
    @MainActor
    @Suite struct SyncUpListModelTests {}
}

extension ModelTests.SyncUpListModelTests {
    // tests...
}

extension EventTests.UserEventTests {
    @MainActor
    @Suite(.serialized) struct SyncUpListUserEventTests {}
}

extension EventTests.UserEventTests.SyncUpListUserEventTests {
    // tests...
}

extension EventTests.OverlayTests {
    @MainActor
    @Suite(.serialized) struct SyncUpListOverlayTests {}
}

extension EventTests.OverlayTests.SyncUpListOverlayTests {
    // tests...
}
```

## Model Tests

Use model tests to validate that, assuming the UI sends the right actions, the right state changes happen and the right effects fire.

Representative files:

- `SyncUps/UI/SyncUpList/Tests/SyncUpListModelTests.swift`
- `SyncUps/UI/SyncUpDetails/Tests/SyncUpDetailsModelTests.swift`
- `SyncUps/UI/RecordMeeting/Tests/RecordMeetingModelTests.swift`

Pattern:

- Create the store with `Feature.store(...)`.
- Inject a narrow test environment directly into `store.environment`.
- Send mutations synchronously and assert on `store.state`.
- Await effect sends with `await store.send(.effect(...))?.value`.
- Capture published values by creating a task with `store.firstValue()`, then call `await store.getRequest()` before triggering the publish path.
- Keep these tests focused on reducer behavior, effect behavior, and store state transitions rather than UI wiring.

## User Event Tests

Use user event tests to validate that `ContentView` wires user interactions to the correct semantic actions.

Representative files:

- `SyncUps/UI/SyncUpList/Tests/SyncUpListUserEventTests.swift`
- `SyncUps/UI/SyncUpForm/Tests/SyncUpFormUserEventTests.swift`

Pattern:

- Render `store.contentView` inside the same container the real app uses, often `NavigationStack`.
- Drive the UI with Hammer's `EventGenerator`.
- Add stable `testIdentifier` values in the UI file before writing the test.
- Keep these tests serialized.
- Validate control wiring, gestures, and published outputs from the rendered view.

## Overlay Tests

Use overlay tests to validate that sheets and alerts work as expected, and that user actions change state and child stores the right way.

Representative files:

- `SyncUps/UI/SyncUpList/Tests/SyncUpListOverlayTests.swift`
- `SyncUps/UI/SyncUpDetails/Tests/SyncUpDetailsOverlayTests.swift`
- `SyncUps/UI/RecordMeeting/Tests/RecordMeetingOverlayTests.swift`

Pattern:

- Render the real view so `.connectOnAppear` installs the production wiring.
- Override `appEnv` with `TaskIsolatedEnv` when the feature uses real environment helpers.
- Trigger the parent effect, access the child store through `store.child()`, and complete the child with `publishOnRequest(...)` or `cancelOnRequest()`.
- Wait for presentation and dismissal animations before asserting on child lifetime.
- Keep these tests serialized.
- Use them only when the component actually has overlays or overlay-backed interactions.

## Coverage Checklist

- Add at least one model test for reducer logic.
- Add user-event tests when control wiring or publish behavior is user-facing.
- Add overlay tests when the feature uses child stores, continuations, alerts, or real env hookup.
