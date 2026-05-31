# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
flutter pub get          # Install dependencies
flutter run              # Run the app (choose device when prompted)
flutter run -d linux     # Run on Linux desktop
flutter run -d web-server --web-port 8080  # Run on web
flutter test             # Run all tests
flutter test test/widget_test.dart  # Run a single test file
flutter analyze          # Static analysis (lint)
flutter build apk        # Build Android APK
flutter build linux      # Build Linux desktop

# Code generation (run after adding/changing @freezed, @riverpod, or @JsonSerializable classes)
dart run build_runner build --delete-conflicting-outputs
dart run build_runner watch --delete-conflicting-outputs  # watch mode during development
```

## Key Packages

| Package | Purpose |
|---|---|
| `go_router` | Declarative routing |
| `flutter_riverpod` + `riverpod_annotation` | State management (code-gen style) |
| `freezed` | Immutable data classes and sealed unions |
| `json_serializable` | JSON serialization (used alongside freezed) |
| `build_runner` | Code generation runner (dev dependency) |

## Architecture

MVVM with three layers, organized layer-first at the top level and feature-first within `ui/`.

```
lib/
  config/              # Dependency wiring (providers, env config)
  data/
    repositories/      # One folder per domain; interface + implementations together
    services/          # API clients, local storage services
  domain/
    models/            # Freezed immutable models, one folder per model
    use_cases/         # Business logic that spans multiple repositories
  routing/             # go_router configuration
  ui/
    core/              # Shared widgets, themes, localization
    <feature>/         # e.g. home/, booking/, auth/
      view_models/     # Riverpod Notifiers — one per screen
      widgets/         # Screens and sub-widgets for this feature
  utils/
  main.dart
```

**Layer rules:**
- `ui` depends on `domain`; never imports from `data`
- `data` implements interfaces defined in `domain`
- `domain` is pure Dart — no Flutter or package dependencies
- Repository interfaces live in `data/repositories/<domain>/` alongside their implementations, not in `domain/`

## State Management Pattern

ViewModels are `Notifier` / `AsyncNotifier` classes exposed via `@riverpod`. Use `@riverpod` (auto-dispose) by default; `@Riverpod(keepAlive: true)` for app-lifetime state (e.g. auth). ViewModels delegate to repositories/use cases and expose state to widgets — no business logic in widgets.

## Data Classes

All models use `freezed`. Domain models and API models both follow this pattern (as in compass_app):

```dart
@freezed
class Booking with _$Booking {
  const factory Booking({required String id, required String destination}) = _Booking;
  factory Booking.fromJson(Map<String, dynamic> json) => _$BookingFromJson(json);
}
```

Run `dart run build_runner build --delete-conflicting-outputs` after any model change.

## Routing

All routes declared in `lib/routing/router.dart` using go_router. Auth guards go in `redirect` callbacks driven by Riverpod auth state.

## Skills

Flutter and Dart skills are registered in `skills-lock.json`. Use `/skills` to browse guided workflows for adding tests, routing setup, JSON serialization, and more.

## Linting

Lint rules from `package:flutter_lints/flutter.yaml` (`analysis_options.yaml`). Run `flutter analyze`. Suppress with `// ignore: rule_name`.
