# Overlay Patterns

Use this file when the feature presents sheets or alerts, or when overlay behavior needs reducer and environment coordination.

## Table of Contents

- Sheets
- Alerts

## Sheets

Model sheets as part of the feature architecture, not as isolated SwiftUI callbacks.

Use a child store to drive sheet presentation when the sheet is another feature.

Reference example:

- `SyncUps/UI/SyncUpList/SyncUpListUI.swift`
- `SyncUps/UI/SyncUpList/SyncUpListEnv.swift`

Pattern:

- Expose a typed child UI computed property in `ContentView`, for example `var createSyncUpUI: StoreUI<SyncUpForm>? { .init(store.child()) }`.
- Present it from SwiftUI with `.sheet(self, \.createSyncUpUI) { ui in ui.makeView() }`.
- Trigger the sheet from a semantic effect action such as `.addSyncUp`.
- Create and run the child store from an env helper with `try? await store.run(childStore)`.
- Let the child publish or cancel instead of coordinating sheet dismissal through arbitrary view state whenever the standard child-store boundary fits.
- Persist or transform the sheet result in the env helper or parent effect after the child returns.

Inline sheet example adapted from `SyncUpList`:

```swift
extension SyncUpList: StoreUINamespace {
    struct ContentView: StoreContentView {
        typealias Nsp = SyncUpList
        @ObservedObject var store: Store

        var createSyncUpUI: StoreUI<SyncUpForm>? { .init(store.child()) }

        var body: some View {
            List { ... }
                .sheet(self, \.createSyncUpUI) { ui in
                    ui.makeView()
                }
                .toolbar {
                    Button(action: { store.send(.effect(.addSyncUp)) }) {
                        Image(systemName: "plus")
                    }
                }
                .connectOnAppear {
                    store.environment = .init(
                        createSyncUp: { [weak store] in
                            guard let store else { return nil }
                            return await Nsp.createSyncUp(store: store)
                        },
                        allSyncUps: Nsp.allSyncUps
                    )
                }
        }
    }
}

extension SyncUpList {
    @MainActor
    static func createSyncUp(store: Store) async -> SyncUp? {
        let formStore = SyncUpForm.store(
            syncUp: .init(id: .init(), attendees: [Attendee(id: .init())]),
            title: "New sync-up",
            saveTitle: "Add",
            cancelTitle: "Dismiss"
        )
        if let syncUp = try? await store.run(formStore) {
            appEnv.storageClient.saveSyncUp(syncUp)
            return syncUp
        }
        else {
            return nil
        }
    }
}
```

Use this pattern when the sheet owns real feature logic and should behave like a nested feature interaction.

## Alerts

Model alerts as part of the feature architecture and bridge them through typed results.

Reference example:

- `SyncUps/UI/RecordMeeting/RecordMeetingUI.swift`
- `SyncUps/UI/RecordMeeting/RecordMeeting.swift`

Pattern:

- Define a typed result enum for each alert, such as `EndMeetingAlertResult` or `SpeechRecognizerFailureAlertResult`.
- Keep the continuation in `ContentView` using `@State private var ...: CheckedContinuation<ResultType, Error>?`.
- Present the alert with `.taskAlert(...)` and map each button to one semantic enum case.
- Install the environment closure in `.connectOnAppear` with `withCheckedThrowingContinuation`, storing the continuation into that `@State`.
- Await the environment closure from `runEffect` using `.asyncAction` or `.asyncActionSequence`.
- Use `.asyncActionSequence` when the alert needs coordination around the wait, such as temporarily suppressing timer updates before the alert and restoring them afterward.

Inline alert example adapted from `RecordMeeting`:

```swift
enum EndMeetingAlertResult {
    case saveAndEnd
    case discard
    case resume
}

struct StoreEnvironment {
    let showEndMeetingAlert: (_ discardable: Bool) async throws -> EndMeetingAlertResult
}

enum EffectAction {
    case showEndMeetingAlert(discardable: Bool)
}

@State private var endMeetingAlertResult:
    CheckedContinuation<EndMeetingAlertResult, Error>? = nil
@State private var showDiscardButton = false

var body: some View {
    content
        .taskAlert(
            "End meeting?",
            $endMeetingAlertResult,
            actions: { complete in
                Button("Save and end") { complete(.saveAndEnd) }
                if showDiscardButton {
                    Button("Discard", role: .destructive) { complete(.discard) }
                }
                Button("Resume", role: .cancel) { complete(.resume) }
            },
            message: {
                Text("What would you like to do?")
            }
        )
        .connectOnAppear {
            store.environment = .init(
                showEndMeetingAlert: { discardable in
                    showDiscardButton = discardable
                    return try await withCheckedThrowingContinuation { continuation in
                        endMeetingAlertResult = continuation
                    }
                }
            )
        }
}

@MainActor
static func runEffect(_ env: StoreEnvironment, _ state: StoreState, _ action: EffectAction) -> Store.Effect {
    switch action {
    case .showEndMeetingAlert(let discardable):
        return .asyncActionSequence { send in
            send(.mutating(.updateIgnoreTimer(true)))
            defer { send(.mutating(.updateIgnoreTimer(false))) }

            guard let result = try? await env.showEndMeetingAlert(discardable) else { return }
            switch result {
            case .saveAndEnd:
                send(.effect(.publishMeeting(transcript: state.transcript)))
            case .discard:
                send(.publish(.discard))
            case .resume:
                break
            }
        }
    }
}
```

Keep the alert semantics in the reducer layer:

- The UI should only map buttons to typed alert results.
- The reducer should decide what those results mean for state updates, publishing, cancellation, or follow-up effects.
