---
name: dart-generate-test-mocks
description: Define mock objects for external dependencies using `package:mocktail`. Use when unit testing classes that depend on complex external services like APIs, databases, or repositories.
metadata:
  last_modified: Fri, 06 Jun 2026 00:00:00 GMT
---
# Mocking with mocktail

This project uses `mocktail` (not `mockito`). No code generation is needed — `mocktail` uses `extends Mock` with runtime behavior.

## Contents
- [Structuring Code for Testability](#structuring-code-for-testability)
- [Managing Dependencies](#managing-dependencies)
- [Creating Mocks](#creating-mocks)
- [Implementing Unit Tests](#implementing-unit-tests)
- [Workflow: Creating and Running Mocked Tests](#workflow-creating-and-running-mocked-tests)
- [Examples](#examples)

## Structuring Code for Testability

Design classes to support dependency injection. Inject external services through constructors so they can be swapped with mocks in tests.

- Define an abstract interface (or concrete class) for the dependency.
- Inject via constructor, not via global access or `ref.read()` at call sites.
- Expose `@riverpod` providers that accept the dependency — override the provider in tests.

## Managing Dependencies

Already in `pubspec.yaml`:
```yaml
dev_dependencies:
  mocktail: ^1.0.5
```

No `build_runner` step needed for mocks.

## Creating Mocks

Extend `Mock` and implement the target interface:
```dart
class MockUserRepository extends Mock implements UserRepository {}
```

For classes that require constructor arguments, use `registerFallbackValue` in `setUpAll` for any custom types used as arguments:
```dart
setUpAll(() {
  registerFallbackValue(const User(id: '', name: ''));
});
```

## Implementing Unit Tests

- **Stubbing:** Use `when(() => mock.method()).thenReturn(value)` or `thenAnswer((_) async => value)` for async.
  - Always use arrow syntax in `when()` — not `when(mock.method())`.
- **Verification:** Use `verify(() => mock.method()).called(1)` to assert invocations.
- **Argument matchers:** Use `any()`, `any(named: 'paramName')`, or specific values.
- **Throwing:** Use `thenThrow(exception)` to simulate errors.

## Workflow: Creating and Running Mocked Tests

### Task Progress
- [ ] 1. Identify the external dependency to mock (e.g., `UserRepository`).
- [ ] 2. Ensure the dependency is injected via constructor or `@riverpod` provider.
- [ ] 3. Create `class MockFoo extends Mock implements Foo {}` in the test file.
- [ ] 4. Call `registerFallbackValue` for any custom types in `setUpAll`.
- [ ] 5. Stub behaviors with `when(() => mock.method()).thenAnswer(...)`.
- [ ] 6. Execute the system under test.
- [ ] 7. Assert outcomes with `expect()` and verify interactions with `verify()`.
- [ ] 8. Run `flutter test` or `dart test`.

### Feedback Loop
1. Run `flutter test`.
2. If `MissingStubError`: add a `when()` stub for the missing method.
3. If `type is not a subtype`: add `registerFallbackValue` for the missing type.
4. Repeat until green.

## Examples

### Mocking a Repository

```dart
// test/ui/gallery/view_models/gallery_view_model_test.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:push_it_to_the_limit/data/repositories/photo/photo_repository.dart';
import 'package:push_it_to_the_limit/ui/gallery/view_models/gallery_view_model.dart';

class MockPhotoRepository extends Mock implements PhotoRepository {}

void main() {
  late MockPhotoRepository mockRepo;
  late ProviderContainer container;

  setUp(() {
    mockRepo = MockPhotoRepository();
    container = ProviderContainer(
      overrides: [
        photoRepositoryProvider.overrideWith((ref) => mockRepo),
      ],
    );
  });

  tearDown(() => container.dispose());

  test('loads photos successfully', () async {
    final photos = [Photo(id: '1', title: 'Sunset')];
    when(() => mockRepo.getPhotos()).thenAnswer((_) async => photos);

    final result = await container.read(galleryViewModelProvider.future);

    expect(result, photos);
    verify(() => mockRepo.getPhotos()).called(1);
  });

  test('propagates error from repository', () async {
    when(() => mockRepo.getPhotos()).thenThrow(Exception('Network error'));

    final result = container.read(galleryViewModelProvider);

    await expectLater(result.future, throwsA(isA<Exception>()));
  });
}
```

### Mocking with Custom Type Arguments

```dart
class MockBookingRepository extends Mock implements BookingRepository {}

void main() {
  setUpAll(() {
    // Required when BookingRequest is passed as an argument to a mocked method
    registerFallbackValue(
      BookingRequest(destination: '', userId: ''),
    );
  });

  test('creates booking', () async {
    when(() => mockRepo.createBooking(any())).thenAnswer((_) async {});

    await container.read(bookingViewModelProvider.notifier).submit(request);

    verify(() => mockRepo.createBooking(any())).called(1);
  });
}
```
