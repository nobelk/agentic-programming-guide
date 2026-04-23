# Dart & Flutter Agent Guidelines

> A reference for AI coding agents producing **idiomatic, production-grade, performant** Dart/Flutter code. These guidelines assume Dart 3.x and Flutter 3.x+ unless noted.

---

## 1. Core Principles

1. **Correctness first, then clarity, then performance.** Never sacrifice correctness for cleverness. Optimize only with measurement.
2. **Immutability by default.** Prefer `final`, `const`, and immutable data classes. Mutation is opt-in, not opt-out.
3. **Composition over inheritance.** Favor small, composable widgets and functions over deep class hierarchies.
4. **Explicit over implicit.** Prefer named parameters, explicit types on public APIs, and named constructors over positional magic.
5. **Fail loudly in development, gracefully in production.** Use `assert`, typed errors, and sealed result types.
6. **Every abstraction must pay rent.** Do not introduce a repository, BLoC, or layer unless it carries its weight.

---

## 2. Project Structure

### 2.1 Feature-First Layout (preferred for apps > trivial)

```
lib/
  main.dart
  app/                    # App shell, routing, theming, DI composition
    app.dart
    router.dart
    theme.dart
  core/                   # Cross-cutting primitives (no feature deps)
    errors/
    extensions/
    network/
    utils/
  features/
    auth/
      data/               # DTOs, data sources, repository impls
      domain/             # Entities, repository interfaces, use cases
      presentation/       # Widgets, controllers/notifiers, screens
    cart/
    catalog/
  shared/                 # Reusable widgets, design system tokens
    widgets/
    design_system/
```

**Rules:**
- A `feature` **never** imports from another feature's `data/` or `presentation/`. Cross-feature communication flows through `domain/` contracts or shared primitives.
- `core/` depends on nothing app-specific.
- `app/` is the only place allowed to wire concrete implementations to interfaces.

### 2.2 Barrel Files
- Use sparingly. They hurt tree-shaking visibility and cause circular import headaches.
- Prefer direct imports. If used, restrict to a feature's public surface (e.g. `features/auth/auth.dart`).

### 2.3 `pubspec.yaml` Discipline
- Pin direct dependencies to a compatible range (`^x.y.z`), not `any`.
- Separate `dependencies` (runtime) from `dev_dependencies` (tools, mocks, generators).
- Commit `pubspec.lock` for applications; do **not** commit it for reusable packages.

---

## 3. Dart Language Idioms

### 3.1 Null Safety
- Never use `!` (bang operator) to force non-null except when invariants are provably held. Prefer pattern matching or `?.` chains.
- Use `late` only when initialization genuinely cannot happen in the constructor and the field will always be set before read. Never use `late` to silence a compiler error.
- Prefer `String?` over sentinel values like `""` or `"none"`.

```dart
// BAD
final name = user.name!;          // Risk of runtime crash

// GOOD
final name = user.name ?? 'Guest';
// or
if (user.name case final name?) {
  // use name
}
```

### 3.2 Immutability

```dart
// Prefer const-constructible, immutable data classes.
class Money {
  final int amountCents;
  final String currency;
  const Money({required this.amountCents, required this.currency});
}
```

- Every widget, value object, and config class that **can** be `const` **must** have a `const` constructor.
- Use `final` for locals by default. Use `var` only when reassignment is intentional.

### 3.3 Records & Patterns (Dart 3+)
- Use **records** for lightweight return values — avoid creating a class for a one-off 2-field return.
- Use **patterns** for destructuring, switch exhaustiveness, and validation.

```dart
// Record return — good for internal, local use
({int width, int height}) measure() => (width: 100, height: 200);

// Pattern matching on a sealed type — exhaustive & safe
final message = switch (state) {
  Loading()       => 'Loading…',
  Success(:final data) => 'Got ${data.length} items',
  Failure(:final error) => 'Error: $error',
};
```

- Do **not** use records as public API types in widely-consumed packages — prefer named classes for evolvability.

### 3.4 Sealed Classes (Algebraic Data Types)
Model finite state and result types as `sealed` hierarchies so the compiler enforces exhaustiveness.

```dart
sealed class Result<T, E> {
  const Result();
}
final class Ok<T, E> extends Result<T, E> {
  final T value;
  const Ok(this.value);
}
final class Err<T, E> extends Result<T, E> {
  final E error;
  const Err(this.error);
}
```

- Prefer `sealed` + `final` subclasses over abstract classes when the set of subtypes is closed.
- Use `freezed` when you need copyWith, equality, and union types with minimal boilerplate.

### 3.5 Extension Methods
- Use to add ergonomic helpers to types you don't own.
- Keep them focused; one extension per concern. Name with `<Target>X` or `<Target>Extensions`.
- Do **not** use extensions to hide business logic — put that in a service.

```dart
extension DateTimeX on DateTime {
  bool get isToday {
    final now = DateTime.now();
    return year == now.year && month == now.month && day == now.day;
  }
}
```

### 3.6 Async & Concurrency
- **Always** `await` futures you care about. Mark fire-and-forget calls explicitly with `unawaited(...)` from `dart:async`.
- Prefer `Future.wait` for independent concurrent operations.
- Wrap potentially failing awaits with focused `try/catch`; do **not** wrap entire methods in a single catch-all.
- For CPU-bound work, use `Isolate.run` (Dart 3). Never do heavy parsing on the UI thread.

```dart
// CPU-bound parse offloaded to isolate
final parsed = await Isolate.run(() => jsonDecode(bigPayload));
```

### 3.7 Streams
- Use `Stream` for sequences over time; use `Future` for single values.
- Prefer `async*` generators over manual `StreamController` when you can.
- Always cancel `StreamSubscription`s in `dispose()`. Leaked subscriptions are a top-5 source of Flutter memory bugs.

### 3.8 Collections
- Use collection literals (`[]`, `{}`) and collection-if/collection-for over imperative `add` calls.
- Prefer `.map`, `.where`, `.fold` over manual loops when expressiveness wins.
- Materialize lazy iterables with `.toList(growable: false)` when the result is final and size-known.

```dart
final visibleItems = [
  for (final item in items)
    if (item.isVisible) item.copyWith(highlighted: item.id == selectedId),
];
```

---

## 4. Flutter Architecture

### 4.1 Layered / Clean Architecture

```
Presentation  ──▶  Domain  ◀──  Data
 (Widgets,         (Entities,     (DTOs,
  Controllers)      UseCases,      Repositories impl,
                    Repo ifaces)   Data sources)
```

- **Presentation** depends on **Domain**. Never on **Data** directly.
- **Data** depends on **Domain** (to implement its interfaces).
- **Domain** depends on **nothing** framework-specific. It must be pure Dart (no `flutter/material.dart`).

### 4.2 Repository Pattern
- A repository returns **domain entities**, never DTOs or HTTP responses.
- Repositories return `Future<Result<T, Failure>>` or throw **typed** domain exceptions — never raw `DioException` or `SocketException`.
- Keep repositories I/O-focused; put business rules in use cases or controllers.

### 4.3 Use Cases / Interactors
- Introduce use cases when a single operation composes multiple repositories or encapsulates non-trivial business rules.
- Do **not** create a use case class for a pass-through call. That's ceremony, not architecture.

### 4.4 Dependency Injection
- Standard approach: **Riverpod providers** for app-scoped and feature-scoped dependencies.
- Alternative: `get_it` + `injectable` for service locator patterns in very large codebases.
- Never use global singletons (`static` instances of services). They destroy testability.

---

## 5. State Management

### 5.1 Decision Matrix

| Scope                                  | Recommended                                   |
|----------------------------------------|-----------------------------------------------|
| Ephemeral UI state (toggles, forms)    | `StatefulWidget` + `setState`                 |
| Shared app state, async, DI            | **Riverpod** (preferred default)              |
| Event-driven domains (wizards, flows)  | `flutter_bloc` (Cubit/BLoC)                   |
| Simple inherited values                | `InheritedWidget` / `Provider`                |

### 5.2 Riverpod Guidance
- Prefer `@riverpod` code generation (`riverpod_generator`) over manual `Provider` declarations.
- Use `AsyncNotifier`/`AsyncValue` for any state involving I/O — they model loading, data, and error natively.
- Keep providers **small and focused**. One provider per concern. Compose via `ref.watch`.
- Use `ref.listen` for side effects (snackbars, navigation); use `ref.watch` for UI rebuild dependencies.
- Dispose resources in `ref.onDispose`.

```dart
@riverpod
class CartController extends _$CartController {
  @override
  FutureOr<Cart> build() async {
    ref.onDispose(() { /* cleanup */ });
    return ref.read(cartRepositoryProvider).load();
  }

  Future<void> addItem(Item item) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(
      () => ref.read(cartRepositoryProvider).add(item),
    );
  }
}
```

### 5.3 BLoC Guidance
- Use `Cubit` unless you genuinely need event streams with transformers (debounce, throttle, concurrency strategies).
- Emit **immutable** states. Prefer `freezed` for state classes.
- One BLoC per feature screen, not per widget.

### 5.4 Anti-Patterns
- Do **not** call `setState` inside `build`.
- Do **not** do I/O in widget constructors.
- Do **not** store `BuildContext` in a state object; it becomes invalid across rebuilds.
- Do **not** read from multiple state management solutions inside one widget without a deliberate reason.

---

## 6. Widget Best Practices

### 6.1 Composition
- Break widgets into small named classes (`class _CartSummary extends StatelessWidget`) rather than giant private `_build*` methods. Named widgets allow `const`, get names in the devtools tree, and are independently testable.
- A `build` method longer than ~60 lines is a signal to extract.

### 6.2 `const` Everywhere Possible
- `const` constructors allow Flutter to skip rebuilds entirely and share instances.
- Enable `prefer_const_constructors` and `prefer_const_literals_to_create_immutables` in `analysis_options.yaml`.

### 6.3 Stateless vs Stateful
- Default to `StatelessWidget`. Promote to `StatefulWidget` only when a widget owns mutable state, a controller, or an animation.
- Keep `StatefulWidget` state minimal. Lift state up or into a controller/notifier whenever other widgets need to react.

### 6.4 Keys
- Use `ValueKey` or `ObjectKey` when items in a list can be reordered, added, or removed and you need Flutter to preserve state correctly across rebuilds.
- Use `GlobalKey` only as a last resort (e.g., programmatic form access, hero flights). They are expensive and constrain architecture.

### 6.5 Builder Patterns
- Prefer `Builder` / `LayoutBuilder` / `ValueListenableBuilder` to rebuild only the subtree that needs to change.
- Use `Selector` (Provider) or `select` (Riverpod) to narrow rebuilds to specific fields.

### 6.6 Theming
- Never hardcode colors, text styles, or paddings in widgets. Read from `Theme.of(context)` or a design system token class.
- Expose semantic tokens (`AppColors.surfaceMuted`), not primitive values (`Color(0xFFE5E5E5)`).

---

## 7. Performance

### 7.1 Build Cost
- Move expensive computation (sorting, filtering big lists) out of `build`. Memoize with `useMemoized` (flutter_hooks) or compute once in state.
- Use `const` widgets aggressively. They short-circuit rebuild.
- Split large widgets so changing state rebuilds only the affected subtree.

### 7.2 Lists
- **Always** use `ListView.builder` / `GridView.builder` / `SliverList` for lists with > ~10 items or unknown length.
- Set `itemExtent` or `prototypeItem` when all items share height — Flutter can skip layout passes.
- Wrap expensive, rarely-changing list items in `RepaintBoundary`.

### 7.3 Images
- Use `cached_network_image` or equivalent for remote images.
- Specify `cacheWidth`/`cacheHeight` when displaying large images at small sizes — decoding a 4000px image into a 100px thumbnail wastes memory.
- Prefer `Image.asset` for bundled images; `Image.file` for local; ensure `BoxFit` matches layout intent.

### 7.4 Animations
- Use implicit animations (`AnimatedContainer`, `AnimatedOpacity`) for simple tweens.
- Use explicit `AnimationController` + `AnimatedBuilder` for coordinated or interactive animations.
- Always `dispose()` controllers.
- Animate only what changes. Wrap the animating subtree in `AnimatedBuilder` with a targeted `child` parameter so the static portion is built once.

### 7.5 Rebuilds Audit
- Use `debugPrintRebuildDirtyWidgets`, the Flutter DevTools Performance overlay, and `--profile` builds to measure.
- Never optimize on vibes. Profile, change, measure.

### 7.6 Memory
- Cancel `StreamSubscription`s, `Timer`s, `AnimationController`s, and `TextEditingController`s in `dispose`.
- Avoid retaining `BuildContext` in closures that outlive the widget.

---

## 8. Error Handling

### 8.1 Typed Failures
Model domain failures as a sealed hierarchy. Do not leak `Exception` subtypes from other layers.

```dart
sealed class AuthFailure {
  const AuthFailure();
}
final class InvalidCredentials extends AuthFailure { const InvalidCredentials(); }
final class NetworkUnavailable extends AuthFailure { const NetworkUnavailable(); }
final class ServerError extends AuthFailure {
  final int statusCode;
  const ServerError(this.statusCode);
}
```

### 8.2 Result Types vs Exceptions
- **Result types** (`Result<T, F>`, `Either<F, T>`) for **expected** failures (validation, not found, auth denied).
- **Exceptions** for **unexpected** failures (programmer error, invariant violation).
- Do not use exceptions for control flow.

### 8.3 Boundaries
- Catch and translate **at layer boundaries**: a repository catches `DioException`, maps to a domain `Failure`, and returns/throws that.
- UI displays failures via exhaustive `switch` over the sealed failure type — no raw error strings.

### 8.4 Global Safety Nets
- Set `FlutterError.onError` and `PlatformDispatcher.instance.onError` to report uncaught errors to crash reporting (Sentry, Crashlytics).
- Wrap `runApp` in `runZonedGuarded` to catch zone-level async errors.

---

## 9. Data Layer

### 9.1 Serialization
- Use `freezed` + `json_serializable` (or `dart_mappable`) for DTOs. Do **not** write `fromJson`/`toJson` by hand for non-trivial models.
- Keep DTOs in `data/` only. Convert to domain entities at the repository boundary.

### 9.2 HTTP
- Standard choice: `dio` for features (interceptors, cancel tokens, retries) or `http` for minimal needs.
- Centralize base URL, headers, auth, and error mapping in a single `ApiClient` wrapper.
- Always set timeouts. Never leave them to defaults.
- Use cancel tokens / `http.Client.close()` on navigation away from screens that triggered requests.

### 9.3 Local Storage
- `shared_preferences` — only for tiny key/value config (user prefs, flags).
- `flutter_secure_storage` — for tokens and credentials.
- `drift` / `isar` / `sqflite` — for structured, queryable data.
- `hive` — when you need fast key-value with objects, no SQL needed.

### 9.4 Caching
- Read-through cache pattern at the repository: check cache → if stale, fetch network → update cache → return.
- Make cache TTL explicit per resource, not global.

---

## 10. Navigation

- Standard: **`go_router`** for declarative routing, deep links, and nested navigators.
- Define routes in a single location (typed route classes where possible with `go_router_builder`).
- Never pass widgets through route arguments — pass IDs or value objects, let the target screen resolve dependencies.
- Handle auth redirection with `redirect:` in router config, not with imperative checks inside widgets.

---

## 11. Testing

### 11.1 Pyramid
- **Many** unit tests (pure Dart, fast, no Flutter binding).
- **Fewer** widget tests (single widget or screen, pumpWidget + find/verify).
- **Fewest** integration tests (end-to-end flows on device/emulator).

### 11.2 Unit Tests
- Test **domain logic** and **repository mapping logic** without Flutter.
- Use `mocktail` (preferred over `mockito` for null safety and ergonomics).
- Arrange / Act / Assert structure. One logical assertion per test.

### 11.3 Widget Tests
- Test behavior, not implementation. Find by `Semantics` label, by text, or by `Key` — not by widget type when avoidable.
- Pump providers/blocs with overrides in the test harness so tests don't hit real I/O.

```dart
testWidgets('shows empty state when cart is empty', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        cartControllerProvider.overrideWith(() => FakeCartController(empty: true)),
      ],
      child: const MaterialApp(home: CartScreen()),
    ),
  );
  expect(find.text('Your cart is empty'), findsOneWidget);
});
```

### 11.4 Golden Tests
- Use for design system components and critical screens.
- Run golden tests on a single platform (typically macOS or Linux in CI) to avoid font rendering diffs.

### 11.5 Coverage
- Target meaningful coverage of domain + data mapping (typically 80%+). Widget coverage depends on app; don't chase numbers over value.

---

## 12. Code Style & Tooling

### 12.1 Lints
- Enable **`very_good_analysis`** or **`flutter_lints`** at minimum.
- Additionally enable in `analysis_options.yaml`:
  - `prefer_const_constructors`
  - `prefer_const_literals_to_create_immutables`
  - `require_trailing_commas`
  - `avoid_print` (use a logger)
  - `unawaited_futures`

### 12.2 Naming
- `UpperCamelCase` for types, enums, extensions.
- `lowerCamelCase` for variables, functions, parameters, named constants.
- `lowercase_with_underscores` for file names, directories, and import prefixes.
- Prefix private members with `_`. Use library-private rather than exposing internals.
- Do **not** use Hungarian notation or `I`-prefixes for interfaces.

### 12.3 Documentation
- Document every **public** API with `///` doc comments. Include intent, not just signature restatement.
- Use `{@template}` / `{@macro}` for reusable doc blocks in packages.
- Include examples in doc comments for non-obvious APIs.

### 12.4 Formatting
- Always run `dart format .` before commit. Configure git hook or CI check.
- 80-column soft limit; the formatter handles most of it.

### 12.5 Logging
- Use `package:logging` or a structured logger wrapper. Never `print` in production code.
- Log levels must be respected: `fine`/`finer`/`finest` for debug, `info` for milestones, `warning`/`severe` for real problems.
- Never log PII, tokens, or full request/response bodies at `info` or above.

---

## 13. Design Patterns (When & How)

| Pattern | Use When | Flutter/Dart Form |
|---|---|---|
| **Repository** | Abstracting data sources from domain | Abstract class in `domain/`, impl in `data/` |
| **Factory / Factory constructor** | Complex construction, caching, subtype selection | `factory MyClass.fromX(...)` |
| **Builder** | Complex immutable objects with many optionals | `copyWith` via `freezed` |
| **Observer** | Reactive state propagation | `Stream`, `ValueNotifier`, Riverpod, BLoC |
| **Strategy** | Swappable algorithms (sort, format, auth) | Function types or small interface |
| **Adapter** | Wrapping third-party APIs behind domain contracts | Repository impls |
| **Decorator** | Cross-cutting concerns (caching, logging) | Interceptors in `dio`; wrapping providers |
| **Command** | Undo/redo, queued actions | Sealed class of commands + dispatcher |
| **State** | Screen with distinct modes | Sealed state classes + pattern match in `build` |
| **Singleton** | Truly global resource (rare) | Riverpod `Provider` — never a static field |

**Anti-pattern alert:** Do not reach for GoF patterns by reflex. In Dart, first-class functions, records, sealed classes, and pattern matching collapse many classical patterns into a few lines. A `Strategy` is usually just a `Function` typedef.

---

## 14. Common Pitfalls to Avoid

1. **Using `BuildContext` after `await`** without checking `mounted`. Always:
   ```dart
   await doSomething();
   if (!context.mounted) return;
   Navigator.of(context).pop();
   ```
2. **Building huge widgets** that rebuild entirely on any state change. Split and scope.
3. **Calling async work in `initState`** without handling disposal. Use `WidgetsBinding.instance.addPostFrameCallback` or move work into a controller.
4. **Using `FutureBuilder` for stateful data** that should be cached or managed. `FutureBuilder` is for one-shot futures local to a widget; anything shared belongs in a controller.
5. **Ignoring `dispose`.** Controllers, subscriptions, focus nodes — all must be disposed.
6. **Global mutable state** via top-level variables. Use DI.
7. **Mixing UI code and business logic** in the same class. Keep widgets dumb; keep logic testable.
8. **Passing callbacks through 5 widget layers.** That's a signal to use a controller or provider.
9. **Over-fetching**: rebuilding lists that refetch on every scroll. Cache, paginate.
10. **Ignoring platform differences.** Use `Theme.of(context).platform` or `defaultTargetPlatform` for platform-appropriate UX (Cupertino vs Material). Prefer adaptive widgets when targeting both.

---

## 15. Agent Self-Checklist Before Finalizing Code

Before returning any non-trivial Flutter/Dart output, verify:

- [ ] All widgets that can be `const` are `const`.
- [ ] No `!` force-unwrap except with documented invariants.
- [ ] No `print` calls. No commented-out code.
- [ ] Every `StatefulWidget` that owns controllers/subscriptions disposes them.
- [ ] Every `await` either uses `context.mounted` guard before `context` access or doesn't use `context` after.
- [ ] Public APIs have `///` doc comments.
- [ ] Failures are typed; no raw `Exception` thrown across layer boundaries.
- [ ] No business logic in `build` methods.
- [ ] Lists with > 10 items use `*.builder` constructors.
- [ ] Code compiles with `very_good_analysis` or `flutter_lints` enabled, zero warnings.
- [ ] If state management is used, the choice is justified by the state's scope.
- [ ] Tests exist for non-trivial logic and render-critical widgets.

---

## 16. Snippet Library (Reference Templates)

### 16.1 Immutable data class (freezed)
```dart
@freezed
class User with _$User {
  const factory User({
    required UserId id,
    required String displayName,
    String? avatarUrl,
  }) = _User;
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

### 16.2 Riverpod async controller
```dart
@riverpod
class ProductList extends _$ProductList {
  @override
  Future<List<Product>> build(Category category) async {
    final repo = ref.watch(productRepositoryProvider);
    return repo.fetch(category);
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => ref.read(productRepositoryProvider).fetch(category));
  }
}
```

### 16.3 Sealed state + exhaustive switch
```dart
sealed class AuthState { const AuthState(); }
final class Unauthenticated extends AuthState { const Unauthenticated(); }
final class Authenticating extends AuthState { const Authenticating(); }
final class Authenticated extends AuthState {
  final User user;
  const Authenticated(this.user);
}

Widget build(BuildContext context) {
  final state = ref.watch(authControllerProvider);
  return switch (state) {
    Unauthenticated() => const LoginScreen(),
    Authenticating()  => const SplashLoader(),
    Authenticated(:final user) => HomeScreen(user: user),
  };
}
```

### 16.4 Proper StatefulWidget lifecycle
```dart
class _SearchBarState extends State<SearchBar> {
  late final TextEditingController _controller;
  StreamSubscription<String>? _sub;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController();
    _sub = widget.queryStream.listen(_onQuery);
  }

  @override
  void dispose() {
    _sub?.cancel();
    _controller.dispose();
    super.dispose();
  }

  void _onQuery(String q) { /* ... */ }

  @override
  Widget build(BuildContext context) => TextField(controller: _controller);
}
```

---

## 17. Escalation Rules for Agents

When an agent is uncertain, prefer these safe defaults:

1. **Uncertain about state management?** Use Riverpod with `AsyncNotifier`.
2. **Uncertain about navigation?** Use `go_router`.
3. **Uncertain about data classes?** Use `freezed`.
4. **Uncertain about error handling?** Sealed `Failure` + `Result<T, Failure>` at repository boundaries.
5. **Uncertain about a new abstraction?** Don't add it. Inline it until the second caller appears.
6. **Uncertain about performance?** Measure with DevTools before optimizing.
7. **Uncertain about a lint override?** Don't override. Fix the code.

---

*End of guidelines. When in doubt, favor the option that's easier to delete.*
