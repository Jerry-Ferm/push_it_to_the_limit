---
name: flutter-implement-json-serialization
description: Create immutable model classes with JSON serialization using `freezed` and `json_serializable`. Use when defining domain models or API models that require fromJson/toJson support.
metadata:
  last_modified: Fri, 06 Jun 2026 00:00:00 GMT
---
# JSON Serialization with Freezed

This project uses `freezed` + `json_serializable` for all model classes. Do not write manual `fromJson`/`toJson` — use code generation.

## Contents
- [Required Packages](#required-packages)
- [Workflow: Implementing a Serializable Model](#workflow-implementing-a-serializable-model)
- [Workflow: Nested Models](#workflow-nested-models)
- [Examples](#examples)

## Required Packages

Already in `pubspec.yaml`:
```yaml
dependencies:
  freezed_annotation: ^3.0.0
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.14
  freezed: ^3.2.5
  json_serializable: ^6.9.0
```

## Workflow: Implementing a Serializable Model

### Task Progress
- [ ] **Step 1: Create the model file.** Add `part '<filename>.freezed.dart';` and `part '<filename>.g.dart';` directives.
- [ ] **Step 2: Annotate with `@freezed`.** Add `class Foo with _$Foo { ... }`.
- [ ] **Step 3: Define the factory constructor.** Use `const factory` with named required parameters.
- [ ] **Step 4: Add `fromJson`.** Add `factory Foo.fromJson(Map<String, dynamic> json) => _$FooFromJson(json);`.
- [ ] **Step 5: Run code generation.** Execute `dart run build_runner build --delete-conflicting-outputs`.
- [ ] **Step 6: Validate.** Call `Foo.fromJson(map)` and `foo.toJson()` in a unit test.

### When to use which pattern
- **Domain models** (used across features): `@freezed` in `domain/models/<domain>/`.
- **API models** (only in the data layer): also `@freezed`, kept in `data/repositories/<domain>/` or `data/services/`.

## Workflow: Nested Models

If a model contains other model types, annotate with `@JsonSerializable(explicitToJson: true)` — or simply use `@freezed` on both classes (freezed generates `explicitToJson` behavior automatically when using code-gen).

For **list of nested models**, call `fromJson` on each element:
```dart
factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);
```
`json_serializable` handles nested `@freezed` types automatically.

## Examples

### Simple Model

```dart
// lib/domain/models/user/user.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

Usage:
```dart
final user = User.fromJson({'id': '1', 'name': 'Jerry', 'email': 'j@example.com'});
final map = user.toJson();
final copy = user.copyWith(name: 'Updated');
```

### Model with Nested Types

```dart
// lib/domain/models/booking/booking.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:push_it_to_the_limit/domain/models/user/user.dart';

part 'booking.freezed.dart';
part 'booking.g.dart';

@freezed
class Booking with _$Booking {
  const factory Booking({
    required String id,
    required String destination,
    required User user,
    required DateTime date,
  }) = _Booking;

  factory Booking.fromJson(Map<String, dynamic> json) => _$BookingFromJson(json);
}
```

### Sealed Union (multiple cases)

```dart
@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.unauthenticated() = Unauthenticated;
  const factory AuthState.authenticated({required User user}) = Authenticated;
  const factory AuthState.loading() = AuthLoading;
}
```

No `fromJson` needed on sealed unions unless they map to an API discriminated union.

### Custom JSON key names

```dart
@freezed
class ApiUser with _$ApiUser {
  const factory ApiUser({
    required String id,
    @JsonKey(name: 'full_name') required String fullName,
    @JsonKey(name: 'avatar_url') String? avatarUrl,
  }) = _ApiUser;

  factory ApiUser.fromJson(Map<String, dynamic> json) => _$ApiUserFromJson(json);
}
```

### After every model change

```bash
dart run build_runner build --delete-conflicting-outputs
```

Use `watch` mode during active development:
```bash
dart run build_runner watch --delete-conflicting-outputs
```
