---
name: flutter-apply-architecture-best-practices
description: Architects a Flutter application using the recommended layered approach (UI, Logic, Data). Use when structuring a new project or refactoring for scalability.
metadata:
  last_modified: Fri, 06 Jun 2026 00:00:00 GMT
---
# Architecting Flutter Applications

## Contents
- [Architectural Layers](#architectural-layers)
- [Project Structure](#project-structure)
- [Workflow: Implementing a New Feature](#workflow-implementing-a-new-feature)
- [Examples](#examples)

## Architectural Layers

Enforce strict Separation of Concerns by dividing the application into distinct layers. Never mix UI rendering with business logic or data fetching.

### UI Layer (Presentation)
Implement the MVVM pattern using Riverpod code-gen (`@riverpod`).

- **Views:** Extend `ConsumerWidget` (stateless) or `ConsumerStatefulWidget` (stateful). Use `ref.watch(provider)` to subscribe to state — never call `ref.read()` inside `build()`. Keep widgets dumb: no business logic, no direct repository calls.
- **ViewModels:** Extend `Notifier<T>` (sync) or `AsyncNotifier<T>` (async). Annotate with `@riverpod`. The `build()` method defines the initial state and wires dependencies via `ref.watch()`. Expose command methods that mutate state via `state =`. Use `@Riverpod(keepAlive: true)` only for app-lifetime state (e.g., auth).

### Data Layer
Implement the Repository pattern to isolate data access.

- **Services:** Stateless classes wrapping external APIs (Dio clients, local databases). Return raw API models or `Result` wrappers. Exposed as `@riverpod` providers.
- **Repositories:** Consume one or more Services. Transform API models into Domain Models. Handle caching and retry logic. Exposed as `@riverpod` providers.

### Logic Layer (Domain — Optional)
- **Use Cases:** Extract only if complex business logic clutters the ViewModel or must be reused across multiple ViewModels.

### Dependency Injection
No manual DI setup is needed. Each class is exposed as a `@riverpod` provider. ViewModels inject repositories via `ref.watch(repositoryProvider)` inside `build()`.

## Project Structure

```text
lib/
├── config/              # App-level providers, env config
├── data/
│   ├── repositories/    # Interface + implementation per domain
│   └── services/        # Dio clients, local storage
├── domain/
│   ├── models/          # Freezed immutable models
│   └── use_cases/       # Optional cross-repository logic
├── routing/             # go_router config as @riverpod provider
├── ui/
│   ├── core/            # Shared widgets, themes
│   └── <feature>/
│       ├── view_models/ # @riverpod Notifiers
│       └── widgets/     # Screens and sub-widgets
├── utils/
└── main.dart            # runApp(ProviderScope(child: MyApp()))
```

## Workflow: Implementing a New Feature

### Task Progress
- [ ] **Step 1: Define Domain Models.** Create `@freezed` data classes in `domain/models/`.
- [ ] **Step 2: Implement Services.** Create `@riverpod` service classes for external API calls.
- [ ] **Step 3: Implement Repositories.** Create `@riverpod` repository that consumes Services and returns Domain Models.
- [ ] **Step 4: Conditional Logic (Domain Layer).**
  - *If complex cross-repository logic:* Create a Use Case class.
  - *If simple CRUD:* Skip to Step 5.
- [ ] **Step 5: Implement the ViewModel.** Create `@riverpod class FooViewModel extends _$FooViewModel`. Inject repositories via `ref.watch()` in `build()`. Expose command methods.
- [ ] **Step 6: Implement the View.** Extend `ConsumerWidget`. Use `ref.watch(fooViewModelProvider)` and `.when()` for async state.
- [ ] **Step 7: Run code generation.** Execute `dart run build_runner build --delete-conflicting-outputs`.
- [ ] **Step 8: Run tests.** Execute unit tests for the ViewModel and Repository.

## Examples

### Data Layer: Service and Repository

```dart
// 1. Service (Dio client, raw API interaction)
@riverpod
ApiClient apiClient(Ref ref) => ApiClient(dio: ref.watch(dioProvider));

class ApiClient {
  ApiClient({required Dio dio}) : _dio = dio;
  final Dio _dio;

  Future<UserApiModel> fetchUser(String id) async {
    final response = await _dio.get('/users/$id');
    return UserApiModel.fromJson(response.data as Map<String, dynamic>);
  }
}

// 2. Repository (single source of truth, returns Domain Model)
@riverpod
UserRepository userRepository(Ref ref) =>
    UserRepository(apiClient: ref.watch(apiClientProvider));

class UserRepository {
  UserRepository({required ApiClient apiClient}) : _apiClient = apiClient;
  final ApiClient _apiClient;

  Future<User> getUser(String id) async {
    final apiModel = await _apiClient.fetchUser(id);
    return User(id: apiModel.id, name: apiModel.fullName);
  }
}
```

### UI Layer: ViewModel and View

```dart
// 3. ViewModel (AsyncNotifier — preferred for async state)
@riverpod
class ProfileViewModel extends _$ProfileViewModel {
  @override
  Future<User> build(String userId) async {
    final repo = ref.watch(userRepositoryProvider);
    return repo.getUser(userId);
  }

  void refresh() => ref.invalidateSelf();
}

// 4. View (ConsumerWidget)
class ProfileScreen extends ConsumerWidget {
  const ProfileScreen({super.key, required this.userId});
  final String userId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final profileAsync = ref.watch(profileViewModelProvider(userId));

    return profileAsync.when(
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (error, stack) => Center(child: Text('Error: $error')),
      data: (user) => Column(
        children: [
          Text(user.name),
          ElevatedButton(
            onPressed: () => ref.read(profileViewModelProvider(userId).notifier).refresh(),
            child: const Text('Refresh'),
          ),
        ],
      ),
    );
  }
}
```

### App Entry Point

```dart
void main() {
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(routerConfig: router);
  }
}
```
