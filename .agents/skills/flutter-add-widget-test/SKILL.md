---
name: flutter-add-widget-test
description: Implement a component-level test using `WidgetTester` to verify UI rendering and user interactions (tapping, scrolling, entering text). Use when validating that a specific widget displays correct data and responds to events as expected.
metadata:
  last_modified: Fri, 06 Jun 2026 00:00:00 GMT
---
# Writing Flutter Widget Tests

## Contents
- [Setup & Configuration](#setup--configuration)
- [Core Components](#core-components)
- [Riverpod Integration](#riverpod-integration)
- [Workflow: Implementing a Widget Test](#workflow-implementing-a-widget-test)
- [Interaction & State Management](#interaction--state-management)
- [Examples](#examples)

## Setup & Configuration

1. `flutter_test` is already in `dev_dependencies` — no changes needed.
2. Place test files in `test/` mirroring the `lib/` structure (e.g., `test/ui/home/widgets/home_screen_test.dart`).
3. Suffix all test files with `_test.dart`.

## Core Components

- **`WidgetTester`**: Primary interface for building and interacting with widgets. Provided by `testWidgets()`.
- **`Finder`**: Locates widgets (`find.text('Submit')`, `find.byType(TextField)`, `find.byKey(Key('k'))`).
- **`Matcher`**: Verifies state (`findsOneWidget`, `findsNothing`, `findsNWidgets(2)`).

## Riverpod Integration

All widgets that use `ref.watch()` or `ConsumerWidget` must be wrapped in `ProviderScope`.

**Override providers** to inject test doubles (use `mocktail` — not `mockito`). Use `overrideWith` as the universal form — `overrideWithValue` only works on sync functional providers and will not compile on class-based notifiers or async providers:
```dart
await tester.pumpWidget(
  ProviderScope(
    overrides: [
      userRepositoryProvider.overrideWith((ref) => mockRepository),
    ],
    child: const MaterialApp(home: ProfileScreen(userId: '1')),
  ),
);
```

Use `ProviderContainer` for testing providers in isolation (without widgets):
```dart
final container = ProviderContainer(
  overrides: [userRepositoryProvider.overrideWith((ref) => mockRepository)],
);
addTearDown(container.dispose);
final result = await container.read(profileViewModelProvider('1').future);
```

## Workflow: Implementing a Widget Test

### Task Progress
- [ ] **Step 1: Define the test.** Use `testWidgets('description', (tester) async { ... })`.
- [ ] **Step 2: Build the widget.** Call `await tester.pumpWidget(...)`. Wrap in `ProviderScope` if the widget uses Riverpod. Wrap in `MaterialApp` if it needs theme/routing context.
- [ ] **Step 3: Override providers.** If the widget reads providers, override them with mocks or stubs.
- [ ] **Step 4: Pump.** Call `await tester.pump()` or `await tester.pumpAndSettle()` for async state.
- [ ] **Step 5: Locate and verify.** Use `find` + `expect` to assert initial state.
- [ ] **Step 6: Simulate interactions.** Tap, scroll, enter text.
- [ ] **Step 7: Pump again and verify.** Assert updated state.
- [ ] **Step 8: Run.** `flutter test test/path/to/widget_test.dart`.
- [ ] **Step 9: Feedback loop.** Review failures → fix assertions or widget logic → re-run.

## Interaction & State Management

- **Static rendering:** `pumpWidget()` once, then `expect()`.
- **State changes (taps, text input):** `tap()` or `enterText()`, then `pump()`.
- **Animations/async:** `tester.pumpAndSettle()`.
- **Async provider state:** Use `pump()` after interactions to advance futures. `pumpAndSettle()` for animations.
- **Long lists:** `scrollUntilVisible(itemFinder, 500.0, scrollable: listFinder)`.

## Examples

### Testing a ConsumerWidget with Provider Override

```dart
// test/ui/gallery/widgets/gallery_screen_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:push_it_to_the_limit/data/repositories/photo/photo_repository.dart';
import 'package:push_it_to_the_limit/ui/gallery/widgets/gallery_screen.dart';

class MockPhotoRepository extends Mock implements PhotoRepository {}

void main() {
  late MockPhotoRepository mockRepo;

  setUp(() {
    mockRepo = MockPhotoRepository();
  });

  testWidgets('shows photos when loaded', (tester) async {
    when(() => mockRepo.getPhotos()).thenAnswer(
      (_) async => [Photo(id: '1', title: 'Sunset')],
    );

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          photoRepositoryProvider.overrideWith((ref) => mockRepo),
        ],
        child: const MaterialApp(home: GalleryScreen()),
      ),
    );

    // Pump to resolve the async provider
    await tester.pump();
    await tester.pump();

    expect(find.text('Sunset'), findsOneWidget);
  });

  testWidgets('shows error when fetch fails', (tester) async {
    when(() => mockRepo.getPhotos()).thenThrow(Exception('Network error'));

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          photoRepositoryProvider.overrideWith((ref) => mockRepo),
        ],
        child: const MaterialApp(home: GalleryScreen()),
      ),
    );

    await tester.pumpAndSettle();

    expect(find.textContaining('Error'), findsOneWidget);
  });
}
```

### Testing Pure UI (No Providers)

```dart
testWidgets('shows loading indicator', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(
      home: Scaffold(body: CircularProgressIndicator()),
    ),
  );

  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});
```

### Testing Provider Logic with ProviderContainer

```dart
test('GalleryViewModel loads photos', () async {
  final mockRepo = MockPhotoRepository();
  when(() => mockRepo.getPhotos()).thenAnswer(
    (_) async => [Photo(id: '1', title: 'Sunset')],
  );

  final container = ProviderContainer(
    overrides: [photoRepositoryProvider.overrideWith((ref) => mockRepo)],
  );
  addTearDown(container.dispose);

  final photos = await container.read(galleryViewModelProvider.future);
  expect(photos, hasLength(1));
  expect(photos.first.title, 'Sunset');
});
```
