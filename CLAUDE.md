# CLAUDE.md

Guidance for Claude Code when working in this repository. This document also serves as a
blueprint: follow it to reproduce the same architecture in a new project.

## Project Overview

**Communitas** — a community forum app (posts, comments, likes, profiles, search) built with:

- **Flutter** (Dart SDK ^3.10+), **BLoC** for state management
- **Clean Architecture** split across a **monorepo** of local packages
- **Supabase** as the backend (Postgres + RLS + RPCs + Realtime + Storage)
- **fpdart** `Either<Failure, T>` for functional error handling
- **get_it + injectable** for dependency injection
- **go_router** for navigation (auth-driven redirects, StatefulShellRoute bottom nav)

## Monorepo Layout

```
communitas/
├── bloc_app/                # The Flutter APP (presentation layer only)
│   └── lib/
│       ├── core/            # App-wide infra: DI, router, shared blocs, event bus, widgets
│       └── features/        # Feature-first UI: pages, blocs, widgets, handlers
└── package/
    ├── core/                # Shared kernel: errors, usecase base, constants, utils
    ├── domain/              # Business layer: entities, repository contracts, usecases
    └── data_supabase/       # Data layer: Supabase datasources, models, repository impls
```

### Dependency rule (strict, one direction only)

```
bloc_app ──► data_supabase ──► domain ──► core
    │                            ▲          ▲
    └────────────────────────────┴──────────┘
```

- `core` depends on nothing local.
- `domain` depends only on `core`. **No Flutter/Supabase imports in domain** (pure Dart + equatable + fpdart).
- `data_supabase` depends on `domain` + `core` (implements domain contracts against Supabase).
- `bloc_app` depends on all three. UI never imports Supabase directly — always through domain usecases.
- Swapping backends = writing a new `data_*` package; domain and app are untouched.

Local packages are wired via `path:` dependencies in each `pubspec.yaml`:

```yaml
dependencies:
  core:
    path: ../core          # from another package
  domain:
    path: ../../package/domain   # from bloc_app
```

## Layer-by-Layer Conventions

### 1. `package/core` — shared kernel

```
core/lib/
├── constants.dart           # barrel → src/constants/
├── errors.dart              # barrel → src/errors/
├── usecase.dart             # barrel → src/usecase/
├── utils.dart               # barrel → src/utils/
└── src/
    ├── constants/app_constants.dart   # Roles, Tables, Views, Storage, DBFunctions, PostgresErrorCodes
    ├── errors/exceptions.dart         # thrown by DATA layer (AuthenticationException, NetworkException, ...)
    ├── errors/failures.dart           # returned to UI via Either (ServerFailure, NetworkFailure, ...)
    ├── usecase/usecase_interface.dart # UseCase<ReturnType, ParamsType> + NoParams
    └── utils/ui_utils.dart            # showErrorSnackbar etc.
```

Key contracts:

```dart
abstract interface class UseCase<ReturnType, ParamsType> {
  Future<Either<Failure, ReturnType>> call(ParamsType params);
}

abstract class Failure extends Equatable {
  const Failure({required this.message});
  final String message;
}
```

All magic strings (table names, RPC names, storage buckets, role names, Postgres error
codes) live in `app_constants.dart` — never inline them.

### 2. `package/domain` — business rules

```
domain/lib/
├── auth.dart / post.dart / profile.dart / search.dart   # one barrel per feature
└── src/<feature>/
    ├── entities/        # Equatable, immutable, pure Dart (e.g. UserEntity, PostDisplay)
    ├── dto/             # cross-layer value objects (e.g. ImageUploadResult)
    ├── repositories/    # abstract interface class XRepository — the CONTRACT
    └── usecases/        # one class per operation + a usecases.dart barrel
```

Usecase pattern — one file, params class + usecase class together:

```dart
class GetPostsParams extends Equatable {
  const GetPostsParams({required this.offset, this.limit = 10});
  final int offset;
  final int limit;
  @override
  List<Object> get props => [offset, limit];
}

class GetPostsUseCase implements UseCase<List<PostDisplay>, GetPostsParams> {
  GetPostsUseCase({required PostRepository postRepository})
      : _postRepository = postRepository;
  final PostRepository _postRepository;

  @override
  Future<Either<Failure, List<PostDisplay>>> call(GetPostsParams params) =>
      _postRepository.getPosts(offset: params.offset, limit: params.limit);
}
```

Repository contracts return `Future<Either<Failure, T>>` (or `Stream<T>` for reactive data
like `onAuthStateChanged`). Use `NoParams` when a usecase takes no input.

### 3. `package/data_supabase` — data layer

```
data_supabase/lib/
├── auth.dart / post.dart / ...        # barrels mirroring domain features
└── src/<feature>/
    ├── datasources/
    │   ├── x_remote_data_source.dart          # abstract interface (returns MODELS, throws EXCEPTIONS)
    │   └── supabase_x_remote_data_source.dart # impl against SupabaseClient
    ├── models/          # extend domain entities, add fromJson via json_serializable
    └── repositories/    # XRepositoryImpl implements domain contract
```

**Error-handling flow (the heart of the architecture):**

1. **Datasource** talks to Supabase, catches SDK errors, **throws typed Exceptions**
   (`AuthenticationException`, `NetworkException`, `UnknownException`).
2. **RepositoryImpl** wraps datasource calls in try/catch and **maps each Exception to a
   Failure**, returning `Left(Failure)` / `Right(value)`. No exception escapes to the UI.
3. **Bloc** folds the Either into state; UI never sees a raw exception.

```dart
try {
  await _authRemoteDataSource.signup(...);
  return const Right(null);
} on AuthenticationException catch (e) {
  return Left(AuthenticationFailure(message: e.message));
} on NetworkException {
  return const Left(NetworkFailure());
} on UnknownException catch (e) {
  return Left(UnknownFailure(message: e.message));
}
```

**Model pattern** — models extend entities and add serialization:

```dart
@JsonSerializable(createToJson: false)
class UserModel extends UserEntity {
  const UserModel({required super.id, required super.username,
      @JsonKey(name: 'avatar_url') super.avatarUrl, required super.role});
  factory UserModel.fromJson(Map<String, dynamic> json) => _$UserModelFromJson(json);
  factory UserModel.fromSupabaseUser(User user) { ... }
}
```

- `@JsonKey` for snake_case DB columns goes **on the constructor super-parameter**.
- Never declare abstract getters in a concrete model (compile error).
- Requires `part 'x_model.g.dart';` + running build_runner (see Codegen).

**Supabase access patterns:**

- Reads go through **database views** (`post_display_view`) for joined display data.
- Multi-step writes go through **RPCs** (`create_post_and_return_post_display_view`) so
  insert + return happen atomically server-side.
- Names come from `DBFunctions` / `Views` / `Tables` constants in core.
- Enforcement (who can insert/delete) is **RLS policies in Supabase** — client checks are
  UX only, never security. UI role-gating must mirror the RLS policies.
- Realtime: subscribe with `onPostgresChanges`; the table must be added to the
  `supabase_realtime` publication AND readable under RLS or subscription errors.

### 4. `bloc_app` — presentation

```
bloc_app/lib/
├── main.dart                    # dotenv → Supabase.initialize → configureDependencies → runApp
├── core/
│   ├── di/
│   │   ├── di.dart              # getIt + @InjectableInit configureDependencies()
│   │   ├── di.config.dart       # GENERATED — never edit
│   │   └── register_module.dart # @module: SupabaseClient, GoRouter, data + domain registrations
│   ├── config/router/
│   │   ├── app_router.dart      # createRouter(authBloc): redirect logic + all routes
│   │   ├── route_constants.dart # RoutePaths + RouteNames (no magic strings)
│   │   └── go_router_refresh_stream.dart  # bloc stream → Listenable for redirects
│   ├── blocs/realtime/          # app-wide realtime connection bloc
│   ├── bus/                     # GlobalEventBus (@singleton broadcast StreamController)
│   ├── utils/                   # sealed_class_state.dart etc.
│   └── widgets/                 # ErrorPage, ScaffoldWithNavBar (bottom nav shell)
└── features/<feature>/presentation/
    ├── blocs/<bloc_name>/       # one folder per bloc: bloc + event + state (part files)
    ├── pages/                   # route-level widgets
    ├── widgets/                 # feature-local widgets
    └── handlers/                # reusable bloc logic (pagination, like-toggling)
```

**BLoC conventions:**

- One bloc per screen concern (`PostListBloc`, `PostDetailBloc`, `PostFormBloc`, ...).
- `part 'x_event.dart'; part 'x_state.dart';` — events/states are part files.
- States are single Equatable classes with a `status` enum + `copyWith` (not sealed unions),
  including transient fields (`transientFailure`, `scrollToTopEventId`) consumed via
  "Consumed" events after the listener fires.
- Blocs are annotated `@injectable` (page-scoped, new per page) or `@singleton`
  (`AuthenticationBloc`, app-lifetime, `dispose: (i) => i.close()` handled by DI).
- Blocs depend on **usecases only** — never repositories or datasources directly.

**GetIt ↔ widget-tree bridge (critical):** GetIt registration does NOT put a bloc in the
widget tree. Pages bridge explicitly:

```dart
return BlocProvider(
  create: (_) => getIt<PostListBloc>()..add(PostListFetched()),
  child: const PostView(),
);
```

App-wide blocs are provided once in `main.dart` with `BlocProvider.value(value: getIt<AuthenticationBloc>())`.

**Routing:** `createRouter(authBloc)` is registered as `@singleton GoRouter` in the DI
module; `main.dart` uses `routerConfig: getIt<GoRouter>()`. Redirects derive from
`AuthenticationBloc.state.status` (unknown → splash, unauthenticated → login,
authenticated → posts), refreshed via `GoRouterRefreshStream(authBloc.stream)`. Bottom
navigation uses `StatefulShellRoute.indexedStack` with a branch per tab.

## Dependency Injection

- `getIt = GetIt.instance` in `di.dart`; `configureDependencies() => getIt.init();`
- `register_module.dart` is the composition root (`@module abstract class RegisterModule`):
  - `@singleton SupabaseClient` → `Supabase.instance.client`
  - `@singleton GoRouter router(AuthenticationBloc authBloc) => createRouter(authBloc);`
  - Data layer: `@LazySingleton(as: AuthRemoteDataSource) SupabaseAuthRemoteDataSource get ...;`
    (abstract getter — injectable generates construction, resolving constructor deps from GetIt)
  - Usecases: `@injectable` getters (new instance per resolve)
- Registration lifetimes: infra = `@singleton`/`@lazySingleton`; usecases & page blocs = `@injectable` (factory).

## Code Generation (NOT automatic — biggest recurring gotcha)

`.g.dart` and `di.config.dart` are only produced by explicitly running build_runner **in the
package that owns the annotated file**:

```bash
# json_serializable models
cd package/data_supabase && dart run build_runner build --delete-conflicting-outputs

# injectable DI config
cd bloc_app && dart run build_runner build --delete-conflicting-outputs
```

Re-run whenever you: add/change a `@JsonSerializable` model, add/change anything in
`register_module.dart`, or add/remove `@injectable`/`@singleton` annotations. A stale
`di.config.dart` causes runtime `GetIt: Object/factory with type X is not registered`.

## Common Commands

```bash
# Resolve deps — run IN EACH package after changing its pubspec (path deps don't cascade)
cd package/core && flutter pub get
cd package/domain && flutter pub get
cd package/data_supabase && flutter pub get
cd bloc_app && flutter pub get

# Run the app (always from bloc_app, never repo root)
cd bloc_app && flutter run

# Analyze / test per package
flutter analyze
flutter test

# Release Android App Bundle
cd bloc_app && flutter build appbundle --release
# → bloc_app/build/app/outputs/bundle/release/app-release.aab
```

**Android release signing:** `bloc_app/android/app/build.gradle.kts` loads
`bloc_app/android/key.properties` (git-ignored, alongside `*.jks`) for the release
`signingConfig`. Without that file, release signing values are null. Secrets in `.env`
(bundled as an asset) must only ever be the Supabase URL + publishable/anon key.

## Recurring Gotchas (learned the hard way)

1. **Analyzer can't resolve `package:core/...`** → run `flutter pub get` in the package
   whose pubspec you just edited; each package has its own `.dart_tool/package_config.json`.
2. **Missing `.g.dart` / DI type not registered** → run build_runner in the owning package.
3. **`ProviderNotFoundException`** → the bloc is in GetIt but not in the widget tree; wrap
   the page with `BlocProvider(create: (_) => getIt<XBloc>()..add(InitialEvent()))`.
4. **`42501 insufficient_privilege` from Supabase** → RLS policy problem, not app code.
   Check policies AND whether the RPC is `SECURITY DEFINER` (bypasses RLS, checks live in
   function body) vs `SECURITY INVOKER` (policies apply).
5. **Realtime `Unable to subscribe`** → table missing from `supabase_realtime` publication
   or no SELECT policy for the subscriber.
6. **Role/permission UI gates** must mirror server rules: author-or-admin uses `||`, e.g.
   `currentUser?.id == post.authorId || currentUserRole == Roles.admin`.
7. Structural widget changes need **hot restart**, not hot reload.

## Starting a New Project with This Structure

1. Create `package/core` → errors (exceptions + failures), `UseCase` interface, constants.
2. Create `package/domain` → per feature: entities, repository contracts, usecases; barrel per feature.
3. Create `package/data_<backend>` → datasources (throw exceptions), models (extend entities,
   json_serializable), repository impls (map exceptions → failures).
4. Create the app → DI (`di.dart` + `RegisterModule`), router (`createRouter(authBloc)`),
   feature folders with page-scoped blocs bridged from GetIt.
5. Wire `main.dart`: env → backend init → `configureDependencies()` → root providers → `MaterialApp.router`.
6. Run `flutter pub get` in every package, then build_runner where there are annotations.
