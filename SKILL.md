---
name: reducer-architecture
description: Use for any request related to `ReducerArchitecture` or TRA, or whenever the codebase uses the `ReducerArchitecture` package.
---

# Reducer Architecture

Mirror the conventions in the SyncUpsTRA reference app unless the target codebase already has a narrower local pattern. Resolve any `SyncUps/...` example path in this skill against the `master` branch of [SyncUpsTRA on GitHub](https://github.com/ilyathewhite/SyncUpsTRA). Resolve any ReducerArchitecture repo-relative path in this skill, such as `Sources/ReducerArchitecture/...`, against the `master` branch of [ReducerArchitecture on GitHub](https://github.com/ilyathewhite/ReducerArchitecture). Prefer the package's existing store and effect APIs over importing TCA-style conventions or inventing new abstractions.

## Quick Start

- Read `references/feature-patterns.md` before adding or refactoring a feature. It is the detailed guide for feature structure, reducer boundaries, and environment wiring.
- Read `references/child-store-patterns.md` when the feature uses child stores, view-owned auxiliary stores, or parent/child composition.
- Read `references/overlay-patterns.md` when the feature uses sheets or alerts.
- Read `references/async-state-patterns.md` when the feature uses `AsyncTaskValue` or other async UI state patterns.
- Read `references/testing-patterns.md` before adding tests or reviewing coverage.
- Read `references/package-api.md` when you need the exact ReducerArchitecture surface area or template shape.

## Non-Negotiables

- Split features into `<Feature>.swift` for the store namespace and `<Feature>UI.swift` for SwiftUI. Add `<Feature>Env.swift`, `<Feature>State.swift`, and `Views/` only when they make the feature easier to scan.
- Treat each feature as one namespace providing `ContentView`, `StoreState`, `MutatingAction`, `EffectAction`, `StoreEnvironment`, `PublishedValue`, `reduce`, and `runEffect`.
- Create stores with `env: nil`, then install `store.environment` and trigger first store actions in `.connectOnAppear`.
- Keep only view-system state in `ContentView`; put feature state, derived values, and coordination state in `StoreState`.
- Keep synchronous state changes in `reduce` and dependency calls, async work, overlays, timers, and persistence in `runEffect`.
- Use `MutatingAction` only for direct state changes. Use `EffectAction` for dependency work, async work, overlays, child-store coordination, and other non-mutating triggers.
- Keep workflow coordination visible in store actions and state transitions instead of burying it inside environment closures.
- Prefer `store.publish(...)`, `store.cancel()`, `store.binding(...)`, and the built-in async effect helpers over ad hoc callbacks or manually started tasks.
- Treat `bind` as a source-to-target bridge. Prefer child -> parent observation; use parent -> child binding only for simple mirrored input.

## Workflow

1. Build `ContentView` first so the UI reveals the actions the store must support.
2. Add the `StoreState` needed to drive that UI.
3. Implement `reduce` and `runEffect`.
4. Add the minimal `StoreEnvironment` surface still required by those effects.
5. Connect bindings, child stores, and environment hookup in the UI.
6. Compare the result against the closest SyncUpsTRA example and add matching tests.

## Avoid

- Do not call app dependencies from `reduce`.
- Do not wire full environments at arbitrary call sites unless the target codebase already does that consistently.
- Do not make an action mutating when it actually represents effect work.
- Do not hide feature logic inside `StoreEnvironment` when it should stay visible as actions and state transitions.
- Do not move persistence or coordination into random view callbacks when an env helper or reducer effect should own it.
