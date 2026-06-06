---
name: flutter-use-http-package
description: Use `dio` and Riverpod `AsyncNotifier` to execute REST API requests and expose results to the UI. Use when fetching from or sending data to a REST API.
metadata:
  last_modified: Fri, 06 Jun 2026 00:00:00 GMT
---
# Implementing Flutter Networking with Dio and Riverpod

This project uses `dio` (not `http`) for networking and Riverpod providers (not `FutureBuilder`/`StatefulWidget`) to expose async data to the UI.

## Contents
- [Configuration & Setup](#configuration--setup)
- [Request Execution & Response Handling](#request-execution--response-handling)
- [Workflow: Implementing a Network Feature](#workflow-implementing-a-network-feature)
- [Examples](#examples)

## Configuration & Setup

Packages already in `pubspec.yaml`:
```yaml
dependencies:
  dio: ^5.9.2
  pretty_dio_logger: ^1.4.0
  flutter_riverpod: ^3.3.1
  riverpod_annotation: ^4.0.2
```

Create a shared `Dio` provider in `lib/config/`:
```dart
@Riverpod(keepAlive: true)
Dio dio(Ref ref) {
  final dio = Dio(BaseOptions(baseUrl: 'https://api.example.com'));
  dio.interceptors.add(PrettyDioLogger());
  return dio;
}
```

Platform permissions:
- **Android:** Add `<uses-permission android:name="android.permission.INTERNET" />` to `AndroidManifest.xml`.
- **macOS:** Add `com.apple.security.network.client` to both entitlements files.

## Request Execution & Response Handling

- **URIs:** Pass as `String` to `dio.get('/path')` — `BaseOptions.baseUrl` handles the host.
- **Headers:** Set globally in `BaseOptions` or per-request via `Options(headers: {...})`.
- **POST body:** Pass a `Map<String, dynamic>` or a model's `.toJson()` directly to `data:`.
- **Error handling:** Catch `DioException` — it carries `statusCode`, `message`, and `response`. Rethrow as domain exceptions from service/repository classes; never leak `DioException` into the domain.
- **Deserialization:** Parse `response.data` (already decoded by Dio) with `Model.fromJson(response.data as Map<String, dynamic>)`.

## Workflow: Implementing a Network Feature

### Task Progress
- [ ] **Step 1: Define the domain model** using `@freezed` with `fromJson`.
- [ ] **Step 2: Define domain exceptions.** Create a domain exception class for errors that cross the data→domain boundary (e.g., `class NetworkException implements Exception { NetworkException(this.message); final String message; }`).
- [ ] **Step 3: Implement the service.** Create a `@riverpod` class wrapping `Dio`. Return domain models; catch `DioException` and rethrow as domain exceptions.
- [ ] **Step 4: Implement the repository.** Expose a `@riverpod` provider that consumes the service and returns domain models.
- [ ] **Step 5: Implement the ViewModel.** Use `AsyncNotifier<T>` or a simple `@riverpod` future provider.
- [ ] **Step 6: Implement the View.** Use `ConsumerWidget` + `ref.watch(provider).when(...)`.
- [ ] **Step 7: Run code generation.** `dart run build_runner build --delete-conflicting-outputs`.
- [ ] **Step 8: Feedback loop.** Run app → trigger request → inspect `PrettyDioLogger` output → fix errors.

## Examples

### Shared Dio Provider

```dart
// lib/config/dio_provider.dart
import 'package:dio/dio.dart';
import 'package:pretty_dio_logger/pretty_dio_logger.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'dio_provider.g.dart';

@Riverpod(keepAlive: true)
Dio dio(Ref ref) {
  final dio = Dio(BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 30),
  ));
  dio.interceptors.add(PrettyDioLogger(requestBody: true));
  return dio;
}
```

### Domain Exception

```dart
// lib/domain/exceptions/network_exception.dart
class NetworkException implements Exception {
  NetworkException(this.message);
  final String message;
  @override
  String toString() => 'NetworkException: $message';
}
```

### Service Layer

```dart
// lib/data/services/photo_service.dart
import 'package:dio/dio.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'photo_service.g.dart';

@riverpod
PhotoService photoService(Ref ref) =>
    PhotoService(dio: ref.watch(dioProvider));

class PhotoService {
  PhotoService({required Dio dio}) : _dio = dio;
  final Dio _dio;

  Future<List<PhotoApiModel>> fetchPhotos() async {
    try {
      final response = await _dio.get('/photos');
      final list = (response.data as List<dynamic>).cast<Map<String, dynamic>>();
      return list.map(PhotoApiModel.fromJson).toList();
    } on DioException catch (e) {
      throw NetworkException('Failed to fetch photos: ${e.message}');
    }
  }
}
```

### Repository Layer

```dart
// lib/data/repositories/photo/photo_repository.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'photo_repository.g.dart';

@riverpod
PhotoRepository photoRepository(Ref ref) =>
    PhotoRepository(service: ref.watch(photoServiceProvider));

class PhotoRepository {
  PhotoRepository({required PhotoService service}) : _service = service;
  final PhotoService _service;

  Future<List<Photo>> getPhotos() async {
    final apiModels = await _service.fetchPhotos();
    return apiModels.map((m) => Photo(id: m.id, title: m.title)).toList();
  }
}
```

### ViewModel (AsyncNotifier)

```dart
// lib/ui/gallery/view_models/gallery_view_model.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'gallery_view_model.g.dart';

@riverpod
class GalleryViewModel extends _$GalleryViewModel {
  @override
  Future<List<Photo>> build() async {
    final repo = ref.watch(photoRepositoryProvider);
    return repo.getPhotos();
  }

  void refresh() => ref.invalidateSelf();
}
```

### View (ConsumerWidget)

```dart
// lib/ui/gallery/widgets/gallery_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class GalleryScreen extends ConsumerWidget {
  const GalleryScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final photosAsync = ref.watch(galleryViewModelProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Gallery')),
      body: photosAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(child: Text('Error: $error')),
        data: (photos) => ListView.builder(
          itemCount: photos.length,
          itemBuilder: (context, index) => ListTile(
            title: Text(photos[index].title),
          ),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => ref.read(galleryViewModelProvider.notifier).refresh(),
        child: const Icon(Icons.refresh),
      ),
    );
  }
}
```

### POST / Mutation

```dart
Future<void> createBooking(BookingRequest request) async {
  try {
    await _dio.post(
      '/bookings',
      data: request.toJson(),
      options: Options(headers: {'Content-Type': 'application/json'}),
    );
  } on DioException catch (e) {
    throw NetworkException('Failed to create booking: ${e.message}');
  }
}
```
