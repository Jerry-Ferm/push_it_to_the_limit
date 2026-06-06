---
name: flutter-setup-declarative-routing
description: Configure `MaterialApp.router` using `go_router` as a Riverpod provider for URL-based navigation and auth guards. Use when adding new routes or implementing auth-gated navigation.
metadata:
  last_modified: Fri, 06 Jun 2026 00:00:00 GMT
---
# Implementing Routing with go_router and Riverpod

## Contents
- [Core Concepts](#core-concepts)
- [Workflow: Adding Routes](#workflow-adding-routes)
- [Workflow: Auth Guards](#workflow-auth-guards)
- [Workflow: Nested Navigation (Shell Routes)](#workflow-nested-navigation-shell-routes)
- [Examples](#examples)

## Core Concepts

- **`go_router`** handles all declarative routing. Already in `pubspec.yaml`.
- **The router is a Riverpod provider** (`@Riverpod(keepAlive: true)`) so it can `ref.watch()` auth state and trigger redirects reactively.
- All routes are declared in `lib/routing/router.dart`. Auth guards go in the `redirect` callback.
- `ConsumerWidget` in `main.dart` calls `ref.watch(routerProvider)` and passes the result to `MaterialApp.router`.

**Key go_router types:**
- `GoRoute`: Maps a URL path to a screen widget.
- `ShellRoute` / `StatefulShellRoute`: Wraps child routes in a persistent shell (e.g., `BottomNavigationBar`). `StatefulShellRoute` preserves branch state.
- `redirect`: A callback on `GoRouter` or individual `GoRoute` — return a path string to redirect, `null` to allow.

## Workflow: Adding Routes

### Task Progress
- [ ] Add the `GoRoute` entry to the routes list in `lib/routing/router.dart`.
- [ ] Create the screen widget in `lib/ui/<feature>/widgets/`.
- [ ] Run `dart run build_runner build --delete-conflicting-outputs` (because the router is generated via `@riverpod`).
- [ ] Test navigation using `context.go('/path')` or `context.push('/path')`.

## Workflow: Auth Guards

Keep one `GoRouter` instance alive and drive re-evaluation via `refreshListenable`. Watch the notifier (not the state value) so the router body does not re-run and reconstruct `GoRouter` on every auth change — which would wipe the navigation stack.

```dart
@Riverpod(keepAlive: true)
GoRouter router(Ref ref) {
  final notifier = ref.watch(authStateProvider.notifier);

  return GoRouter(
    initialLocation: '/',
    refreshListenable: notifier, // notifier must extend ChangeNotifier
    redirect: (context, state) {
      final isLoggedIn = notifier.isAuthenticated;
      final isOnLoginPage = state.matchedLocation == '/login';

      if (!isLoggedIn && !isOnLoginPage) return '/login';
      if (isLoggedIn && isOnLoginPage) return '/';
      return null;
    },
    routes: [...],
  );
}
```

## Workflow: Nested Navigation (Shell Routes)

### Task Progress
- [ ] Define `StatefulShellRoute.indexedStack` in the `GoRouter` routes list.
- [ ] Create `StatefulShellBranch` for each tab.
- [ ] Implement the shell widget using `StatefulNavigationShell`.
- [ ] Run code generation.

## Examples

### Router Provider (`lib/routing/router.dart`)

```dart
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'router.g.dart';

@Riverpod(keepAlive: true)
GoRouter router(Ref ref) {
  return GoRouter(
    initialLocation: '/',
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomeScreen(),
        routes: [
          GoRoute(
            path: 'details/:id',
            builder: (context, state) =>
                DetailsScreen(id: state.pathParameters['id']!),
          ),
        ],
      ),
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginScreen(),
      ),
    ],
    errorBuilder: (context, state) => ErrorScreen(error: state.error),
  );
}
```

### App Entry Point (`lib/main.dart`)

```dart
import 'package:flutter_web_plugins/url_strategy.dart';

void main() {
  usePathUrlStrategy(); // prevents hash (#) URLs on web
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(
      routerConfig: router,
      title: 'Push It To The Limit',
      theme: ThemeData(
        colorScheme: .fromSeed(seedColor: Colors.deepPurple),
      ),
    );
  }
}
```

### Shell Route with Bottom Nav

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) =>
      AppShell(navigationShell: navigationShell),
  branches: [
    StatefulShellBranch(
      routes: [
        GoRoute(
          path: '/home',
          builder: (context, state) => const HomeScreen(),
        ),
      ],
    ),
    StatefulShellBranch(
      routes: [
        GoRoute(
          path: '/settings',
          builder: (context, state) => const SettingsScreen(),
        ),
      ],
    ),
  ],
),
```

```dart
class AppShell extends StatelessWidget {
  const AppShell({super.key, required this.navigationShell});
  final StatefulNavigationShell navigationShell;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: (index) => navigationShell.goBranch(
          index,
          initialLocation: index == navigationShell.currentIndex,
        ),
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
        ],
      ),
    );
  }
}
```

### Programmatic Navigation

```dart
// Replace current route stack
context.go('/details/123');

// Push onto stack
context.push('/details/123');

// Named route with parameters
context.goNamed('details', pathParameters: {'id': '123'});

// Go back
context.pop();
```
