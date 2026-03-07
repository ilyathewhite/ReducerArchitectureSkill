# Child Store Patterns

Use this file when the feature is composed from subcomponents, owns child stores, or needs a view-owned auxiliary store.

## Table of Contents

- Core Rules
- Inline Single-Child Example
- Inline Multi-Child Example
- View-Owned Auxiliary Store Pattern
- Compose Larger Features With Child Stores

## Core Rules

Use child stores when a parent component is made of smaller components that should keep their own state and UI.

- Let each child remain a normal `StoreNamespace` feature with its own `StoreState`, actions, reducer, and UI.
- Let the parent own child store lifetime and composition.
- Read child stores in the parent UI with typed computed properties built from `store.child()`.
- Prefer binding child state or published values back into the parent store instead of reaching into child internals or defaulting to callbacks.
- Reflect containment in the folder structure by nesting child feature folders inside the parent feature folder when the relationship is structural.

Useful patterns:

- Use `store.addChildIfNeeded(...)` in the parent view initializer when a child should always exist for the lifetime of the parent.
- Use `store.addChild(...)` when the child is conditional or created later.
- Use `store.bindPublishedValue(of: childStore) { ... }` when the parent should react to the child's published result.
- Use `store.bind(to: childStore, on: \.someState) { ... }` when the parent should react to changes in child state.

Less common but still valid:

- Use a view-owned auxiliary store via `@StateObject` when the embedded store is simple, does not need per-instance dynamic input from the parent initializer, and the parent mainly needs to bind to its published value.
- Use callbacks occasionally when `publish`, `cancel`, or bind helpers are not a good fit and the callback keeps the feature simpler without blurring store ownership.

## Inline Single-Child Example

This example is adapted from a real `NotePickerContainer` component. It shows a parent that always owns one embedded child store.

```swift
enum NotePickerContainer: StoreNamespace {
    typealias PublishedValue = ScaleNote?
    typealias StoreEnvironment = Never
    typealias EffectAction = Never

    enum MutatingAction {
        case updateScaleNote(ScaleNote?)
        case updateCanErase(Bool)
    }

    struct StoreState {
        let halfStepAlterationsOnly: Bool
        var scaleNote: ScaleNote?
        var canErase = true
    }
}

extension NotePickerContainer: StoreUINamespace {
    struct ContentView: StoreContentView {
        typealias Nsp = NotePickerContainer
        @ObservedObject var store: Store

        var notePickerStore: NotePicker.Store { store.child()! }

        init(_ store: Store) {
            self.store = store
            store.addChildIfNeeded(
                NotePicker.store(
                    halfStepAlterationsOnly: store.state.halfStepAlterationsOnly,
                    scaleNote: store.state.scaleNote
                )
            )
        }

        var body: some View {
            VStack {
                notePickerStore.contentView

                Button("Done") {
                    store.publish(store.state.scaleNote)
                }
            }
            .overlay(alignment: .topTrailing) {
                Button("Erase") {
                    notePickerStore.send(.mutating(.updateNote(nil)))
                }
                .disabled(!store.state.canErase)
            }
            .connectOnAppear {
                store.bindPublishedValue(of: notePickerStore) {
                    .mutating(.updateScaleNote($0))
                }

                store.bind(to: notePickerStore, on: \.canErase) {
                    .mutating(.updateCanErase($0))
                }
            }
        }
    }
}
```

Why this pattern works:

- The child store owns note-picking behavior and UI.
- The parent store owns only the container-level state it actually cares about.
- The parent reacts to the child's published value and derived state through bindings, not manual polling.

## Inline Multi-Child Example

This example is adapted from a real `ChordFinderContainer` component. It shows a parent coordinating several child stores with different responsibilities.

```swift
extension ChordFinderContainer: StoreUINamespace {
    struct ContentView: StoreContentView {
        typealias Nsp = ChordFinderContainer
        @ObservedObject var store: Store

        private var chordSymbolPickerStore: AnyChordSymbolPicker.Store { store.child()! }
        private var searchResultsStore: ChordFinderResults.Store { store.child()! }
        private var chordQueryStore: ChordFinderQuery.Store? { store.child() }

        init(_ store: Store) {
            self.store = store
            store.addChildIfNeeded(AnyChordSymbolPicker.store(context: .search(store.state.context)))
            store.addChildIfNeeded(ChordFinderResults.store())
        }

        var body: some View {
            MasterDetailView(
                master: {
                    if let chordQueryStore {
                        chordQueryStore.contentView
                    }
                    else {
                        chordSymbolPickerStore.contentView
                    }
                },
                detail: {
                    searchResultsStore.contentView
                }
            )
            .connectOnAppear {
                searchResultsStore.bind(to: store, on: \.searchResults) {
                    .mutating(.updateSearchResults($0), animated: true)
                }

                store.bind(to: searchResultsStore, on: \.chordSelection) {
                    .mutating(.updateChordSelection($0), animated: true)
                }

                store.bind(to: chordSymbolPickerStore, on: \.key) {
                    .mutating(.updateKey($0))
                }

                store.bind(to: chordSymbolPickerStore, on: \.navigationCount) { _ in
                    .mutating(.updateSearchResults(nil))
                }

                if let query = store.state.query {
                    store.addChild(
                        ChordFinderQuery.store(
                            guitar: store.state.guitar,
                            query: query,
                            showSearchResultsNavigation: false
                        )
                    )
                }
            }
        }
    }
}
```

Why this pattern works:

- Each child has one clear job: query entry, result display, or symbol picking.
- The parent coordinates cross-child behavior by binding state changes into parent mutations.
- Optional child stores let the parent switch layouts or modes without collapsing everything into one huge store.

Use child stores when:

- A child view has enough state or logic to deserve its own reducer.
- Several parts of the screen can evolve independently.
- The parent needs to coordinate children while still keeping each child testable and reusable.

## View-Owned Auxiliary Store Pattern

This simpler pattern is useful when the embedded store can be created directly in the view and does not depend on dynamic input from the parent's initializer.

```swift
extension NativeLanguagePicker: StoreUINamespace {
    struct ContentView: StoreContentView {
        typealias Nsp = NativeLanguagePicker
        @ObservedObject var store: Store

        @StateObject private var languagePickerStore =
            LanguagePicker.store(title: nil, recentLanguages: AppSettings.get(\.recentLanguages))

        var body: some View {
            VStack {
                languagePickerStore.contentView
                    .searchBoxContainer()
            }
            .connectOnAppear {
                store.bindPublishedValue(of: languagePickerStore) {
                    .publish($0)
                }
            }
        }
    }
}
```

Prefer parent-owned `store.child()` composition for richer containment, but this pattern is a good fit for simpler helper stores.

## Compose Larger Features With Child Stores

Break a complex component into subcomponents when the parent is becoming hard to reason about.

In that case:

- Let the parent store hold child stores.
- Let the parent `ContentView` render child content views.
- Reflect the containment in the folder structure.
- Subscribe or bind to child store state from the parent only when the parent truly needs to react to child changes.
- Run parent actions in response to child state changes when that coordination belongs to the parent feature.
