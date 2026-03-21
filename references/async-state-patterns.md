# Async State Patterns

Use this file when `StoreState` holds asynchronously loaded values, when the UI is driven by `AsyncTaskValue`, or when effect naming and async state transitions need to follow the TRA conventions.

## Model Async State Explicitly

When `StoreState` holds a value received asynchronously:

- Use `AsyncTaskValue` from `FoundationEx`.
- Prefer switching over `AsyncTaskValue` directly in `ContentView` for the primary async state of the screen.
- It is also acceptable to use convenience properties such as `.isInProgress` or `.error` for secondary async states when that keeps the UI clearer.
- Handle both success and failure through the same mutating action by passing `Result` as the payload when that keeps the state transition unified.

Prefer these action patterns:

- Use `.asyncAction`, `.asyncActions`, `.asyncActionSequence`, and related helpers for async work.
- Use `.asyncActionLatest` or `.asyncActionSequenceLatest` when a new request should cancel the previous one, such as search-as-you-type.
- Remember that reducer-initiated async work begins on the `MainActor`. If the effect needs real parallelism, use `async let` or `withTaskGroup` inside the effect body.
- When a mutation is followed by async work, use the same base name for the mutation and the effect so their relationship is obvious.
  Example: the mutation updates state to an in-progress value, then returns `.action(.effect(.search(query)))`.
- Keep the success and error branches in the same mutating action when both update the same `AsyncTaskValue`.

## Inline AsyncTaskValue Example

The pattern below is adapted from a private `TranslationResultContainer` feature in LinguaSense. It is included inline here so the skill does not depend on a non-public repo.

Store shape:

```swift
import FoundationEx
import ReducerArchitecture

enum TranslationResultContainer: StoreNamespace {
    struct StoreEnvironment {
        let cachedResult: (TranslationQuery) async -> TranslationResult?
        let translate: (TranslationQuery) -> AsyncThrowingStream<TranslationEntry.Value, Error>
        let saveResult: (TranslationResult) async throws -> Void
    }

    enum MutatingAction {
        case translate
        case didGetCachedResult(TranslationResult)
        case didGetTranslation(TranslationQuery, TranslationEntry.Value)
        case didFinishTranslation(Result<Void, Error>)
        case updateReadingTextState(AsyncTaskValue<Void, Error>)
    }

    enum EffectAction {
        case translate(TranslationQuery)
        case saveResult(TranslationResult)
    }

    struct StoreState {
        let queryOrResult: TranslationQueryOrResult
        var partialTranslationResult: TranslationResult?
        var translationResultState: AsyncTaskValue<TranslationResult, Error> = .notStarted
        var readingTextState: AsyncTaskValue<Void, Error> = .notStarted

        var translationResult: TranslationResult? {
            switch translationResultState {
            case .success(let result):
                return result
            default:
                return partialTranslationResult
            }
        }
    }
}

extension TranslationResultContainer {
    @MainActor
    static func reduce(_ state: inout StoreState, _ action: MutatingAction) -> Store.SyncEffect {
        switch action {
        case .translate:
            state.partialTranslationResult = nil
            state.translationResultState = .inProgress
            guard case .query(let query) = state.queryOrResult else { return .none }
            return .action(.effect(.translate(query)))

        case .didFinishTranslation(let result):
            switch result {
            case .success:
                if let translationResult = state.partialTranslationResult {
                    state.translationResultState = .success(translationResult)
                    state.partialTranslationResult = nil
                }
                else {
                    state.translationResultState = .failure(BackendAppError.noTranslations)
                }
                return .none

            case .failure(let error):
                state.translationResultState = .failure(error)
                return .none
            }

        case .updateReadingTextState(let value):
            state.readingTextState = value
            return .none

        default:
            return .none
        }
    }
}
```

Effect shape:

```swift
extension TranslationResultContainer {
    @MainActor
    static func runEffect(_ env: StoreEnvironment, _ state: StoreState, _ action: EffectAction) -> Store.Effect {
        switch action {
        case .translate(let query):
            return .asyncActionSequence { send in
                do {
                    if let cachedResult = await env.cachedResult(query) {
                        send(.mutating(.didGetCachedResult(cachedResult)))
                    }

                    for try await value in env.translate(query) {
                        send(.mutating(.didGetTranslation(query, value)))
                    }

                    send(.mutating(.didFinishTranslation(.success(()))))
                }
                catch {
                    send(.mutating(.didFinishTranslation(.failure(error))))
                }
            }

        case .saveResult(let translationResult):
            return .asyncAction {
                do {
                    try await env.saveResult(translationResult)
                    return nil
                }
                catch {
                    return .mutating(.updateReadingTextState(.failure(error)))
                }
            }
        }
    }
}
```

This example shows a primary async state, a secondary async state, and partial streamed results without inventing custom loading enums.

UI pattern:

```swift
var body: some View {
    VStack {
        if case .failure(let error) = store.state.readingTextState {
            TranslationErrorView(
                error: error,
                retry: {
                    store.send(.mutating(.updateReadingTextState(.notStarted), animated: true))
                }
            )
        }
        else if let result = store.state.translationResult {
            TranslationResultView(
                translationResult: result,
                onReadingError: { error in
                    store.send(.mutating(.updateReadingTextState(.failure(error)), animated: true))
                }
            )
        }
        else {
            switch store.state.translationResultState {
            case .notStarted:
                EmptyView()

            case .inProgress:
                ActivityIndicator(symbolName: "icloud", message: "Translating via AI service")

            case .success(let result):
                TranslationResultView(translationResult: result)

            case .failure(let error):
                TranslationErrorView(
                    error: error,
                    retry: {
                        store.send(.mutating(.translate))
                    }
                )
            }
        }
    }
}
```
