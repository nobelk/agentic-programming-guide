# AGENTS.md — Dart & Flutter

Instructions for AI agents contributing code to this project. Read before writing code.

## Stack Defaults

When a choice isn't dictated by existing code, use:

- **State management:** Riverpod (`@riverpod` code-gen, `AsyncNotifier` for I/O)
- **Routing:** `go_router`
- **Data classes:** `freezed` + `json_serializable`
- **HTTP:** `dio` with a centralized `ApiClient`
- **DI:** Riverpod providers (no static singletons, no global mutable state)
- **Lints:** `very_good_analysis` or `flutter_lints`, zero warnings
- **Testing:** `mocktail` for mocks; unit > widget > integration

## Project Layout

Feature-first. Each feature has `data/`, `domain/`, `presentation/`.

- `domain/` is pure Dart — **no `flutter/material.dart` imports**.
- `presentation/` depends on `domain/`, never on `data/`.
- `data/` implements interfaces defined in `domain/`.
- Features never import another feature's `data/` or `presentation/`.
- `app/` is the only place that wires concrete impls to interfaces.

## Non-Negotiable Rules

1. **`const` everything possible.** Every widget, value object, and config class that can have a `const` constructor must have one.
2. **No `!` force-unwrap** unless an invariant is documented and provably held. Prefer `??`, `?.`, or pattern matching.
3. **Dispose everything.** `AnimationController`, `TextEditingController`, `StreamSubscription`, `Timer`, `FocusNode` — all disposed in `dispose()`.
4. **`context` after `await` requires `if (!context.mounted) return;`** — always.
5. **No business logic in `build()`.** Move it to a controller or use case.
6. **No I/O in constructors or `initState` without lifecycle handling.** Use a controller or `addPostFrameCallback`.
7. **Lists > 10 items use `.builder` constructors.** Never `ListView(children: [...])` for dynamic data.
8. **Typed failures at layer boundaries.** Repositories map `DioException`/`SocketException` to a sealed `Failure`. Never leak transport errors to UI.
9. **No `print`.** Use a logger. Never log tokens, PII, or full bodies at `info` or above.
10. **`final` for locals by default.** `var` only when reassignment is intentional.

## Dart 3 Idioms

- Use **sealed classes** + **pattern matching** for finite state and result types. The compiler enforces exhaustiveness — rely on it.
- Use **records** for local, lightweight return values; don't expose them in public package APIs.
- Use **`Isolate.run`** for CPU-bound work (parsing big JSON, image processing). Never block the UI thread.
- Use `async*` generators over manual `StreamController` where possible.
- Mark fire-and-forget calls with `unawaited(...)`. Every other future is awaited.

## Widgets

- Extract named widget classes instead of `_buildFoo()` methods. Named widgets can be `const`, appear in DevTools, and are testable.
- Default to `StatelessWidget`. Promote to `StatefulWidget` only for owned mutable state, controllers, or animations.
- Read styling from `Theme.of(context)` or design system tokens — never hardcode colors, text styles, or paddings.
- Use `ValueKey`/`ObjectKey` for reorderable lists. Avoid `GlobalKey` unless there's no alternative.
- Scope rebuilds: `select` (Riverpod), `Selector` (Provider), `ValueListenableBuilder`, or split widgets so state changes rebuild minimal subtrees.

## State Management Rules

- One provider/controller per concern. Compose via `ref.watch`.
- `ref.watch` for rebuild dependencies. `ref.listen` for side effects (snackbars, navigation).
- Use `AsyncValue` / `AsyncNotifier` for anything involving I/O — never hand-roll loading/error booleans.
- Emit immutable states. Use `freezed`.
- Never store `BuildContext` in state objects.
- Never call `setState` inside `build`.

## Error Handling

- **Expected failures** (validation, not found, auth denied) → `Result<T, Failure>` or sealed `Failure` types.
- **Unexpected failures** (programmer error) → exceptions. Never use exceptions for control flow.
- UI handles failures via exhaustive `switch` on sealed `Failure` — no raw error strings in widgets.
- Set `FlutterError.onError` and `PlatformDispatcher.instance.onError` for crash reporting. Wrap `runApp` in `runZonedGuarded`.

## Performance

- Profile before optimizing. Use DevTools Performance overlay and `--profile` builds.
- Offload heavy parsing/compute to `Isolate.run`.
- Set `cacheWidth`/`cacheHeight` on images displayed smaller than source.
- Set `itemExtent` or `prototypeItem` on lists with uniform heights.
- Wrap expensive, rarely-changing list items in `RepaintBoundary`.
- Cancel in-flight requests on navigation away (cancel tokens).

## Testing

- Test domain logic and repository mapping as plain Dart unit tests — no Flutter binding needed.
- Widget tests: find by `Semantics`, text, or `Key`. Override providers/blocs with fakes; no real I/O.
- Golden tests run on one platform in CI to avoid font diffs.
- Target meaningful coverage on domain + data mapping. Don't chase numbers on widget code.

## Commands

```bash
dart format .                    # before every commit
dart analyze                     # must be clean
flutter test                     # unit + widget
dart run build_runner build -d   # after modifying @freezed / @riverpod classes
```

## Before Submitting Code

- [ ] `dart format .` run, `dart analyze` clean, zero lint warnings
- [ ] All `const`-able widgets are `const`
- [ ] No `!` force-unwraps without justification
- [ ] Every stateful widget with resources implements `dispose`
- [ ] No `context` used after `await` without `mounted` check
- [ ] No `print`, no commented-out code
- [ ] Public APIs have `///` doc comments
- [ ] Failures are typed at layer boundaries
- [ ] Lists with dynamic/large data use `.builder`
- [ ] Tests exist for non-trivial logic

## When Uncertain

1. State management? → Riverpod `AsyncNotifier`.
2. Routing? → `go_router`.
3. Data class? → `freezed`.
4. Error handling? → Sealed `Failure` + `Result<T, Failure>`.
5. New abstraction? → Don't add it. Inline until a second caller appears.
6. Performance concern? → Measure first.
7. Tempted to override a lint? → Fix the code instead.

Prefer the option that's easier to delete.
