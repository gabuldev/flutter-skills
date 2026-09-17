---
name: flutter-caching
description: Use when implementing caching or offline behaviour in a Flutter feature - an offline-first repository, image caching, a Dio cache interceptor, a write sync queue, cache invalidation, or choosing between secure storage and shared_preferences. Also triggers on "make it work without internet", "store it locally", or "it is slow, it refetches every time".
---

You are implementing caching for a Flutter feature. The patterns below assume Dio for HTTP, a secure-storage wrapper for sensitive data, `flutter_bloc` for state, and Clean Architecture (domain/data/presentation) - adapt the names to your stack, the shapes carry over.

## Caching Strategy Selection

| Strategy | When to Use | Package/Impl |
|---|---|---|
| `shared_preferences` | Non-sensitive key-value: UI prefs, filters, TTL timestamps, feature flags | `shared_preferences` |
| Secure storage | Auth tokens, credentials, sensitive user data. Wrap it in a `Storage` service with a `StorageKeys` enum rather than passing raw strings | `flutter_secure_storage` |
| In-memory cache | Short-lived GET responses, session-scoped data, computed values | Dio interceptor or Map in repository |
| `cached_network_image` | Product photos, logos, any remote image in list or detail screens | `cached_network_image` |
| Local DB (drift/sqflite) | Offline-first with complex queries (orders, history, large local datasets) | `drift` or `sqflite` |

**Rule of thumb**: SecureStorage for secrets, shared_preferences for preferences, in-memory for API responses, local DB for offline-first features.

---

## 1. Offline-First Repository Pattern

When argument is `offline-first`, implement the yield-local-then-remote pattern. This is what any app that must keep working without connectivity needs.

### Domain Layer - No changes needed

The repository interface stays the same. Caching is a data-layer concern:

```dart
// domain/repositories/<feature>_repository.dart
abstract class <Feature>Repository {
  Stream<List<<Feature>>> watchAll(); // Stream instead of Future for offline-first
  Future<List<<Feature>>> getAll();
  Future<void> sync();
}
```

### Data Layer - Local Datasource

```dart
// data/datasources/<feature>/<feature>_local_datasource.dart
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';

class <Feature>LocalDatasource {
  static const _cacheKey = '<feature>_cache';
  static const _ttlKey = '<feature>_cache_ttl';
  static const _ttlDuration = Duration(minutes: 30);

  Future<List<<Feature>Model>?> getCached() async {
    final prefs = await SharedPreferences.getInstance();
    final json = prefs.getString(_cacheKey);
    if (json == null) return null;

    // Check TTL
    final ttl = prefs.getInt(_ttlKey) ?? 0;
    if (DateTime.now().millisecondsSinceEpoch > ttl) {
      await clearCache();
      return null;
    }

    final list = jsonDecode(json) as List;
    return list
        .map((e) => <Feature>Model.fromJson(e as Map<String, dynamic>))
        .toList();
  }

  Future<void> saveCache(List<<Feature>Model> models) async {
    final prefs = await SharedPreferences.getInstance();
    final json = jsonEncode(models.map((m) => m.toJson()).toList());
    await prefs.setString(_cacheKey, json);
    await prefs.setInt(
      _ttlKey,
      DateTime.now().add(_ttlDuration).millisecondsSinceEpoch,
    );
  }

  Future<void> clearCache() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove(_cacheKey);
    await prefs.remove(_ttlKey);
  }
}
```

### Data Layer - Offline-First Repository Impl

```dart
// data/repositories/<feature>_repository_impl.dart
import 'dart:async';

class <Feature>RepositoryImpl implements <Feature>Repository {
  final <Feature>Datasource _remoteDatasource;
  final <Feature>LocalDatasource _localDatasource;

  <Feature>RepositoryImpl(this._remoteDatasource, this._localDatasource);

  @override
  Stream<List<<Feature>>> watchAll() async* {
    // 1. Yield cached data immediately (fast UI)
    final cached = await _localDatasource.getCached();
    if (cached != null) {
      yield cached.map((m) => m.toEntity()).toList();
    }

    // 2. Fetch fresh data from remote
    try {
      final remote = await _remoteDatasource.getAll();
      await _localDatasource.saveCache(remote);
      yield remote.map((m) => m.toEntity()).toList();
    } catch (e) {
      // If no cached data was yielded and remote fails, rethrow
      if (cached == null) rethrow;
      // Otherwise silently fail - user already has cached data
    }
  }

  @override
  Future<List<<Feature>>> getAll() async {
    try {
      final remote = await _remoteDatasource.getAll();
      await _localDatasource.saveCache(remote);
      return remote.map((m) => m.toEntity()).toList();
    } catch (e) {
      final cached = await _localDatasource.getCached();
      if (cached != null) return cached.map((m) => m.toEntity()).toList();
      rethrow;
    }
  }

  @override
  Future<void> sync() async {
    final remote = await _remoteDatasource.getAll();
    await _localDatasource.saveCache(remote);
  }
}
```

### DI Registration in CoreInjections

```dart
// core/lib/injections.dart
static List<Inject<Object>> <feature>() => [
  Inject<<Feature>Datasource>(
    (i) => <Feature>DatasourceImpl(dio: i.find<Dio>()),
  ),
  Inject<<Feature>LocalDatasource>(
    (i) => <Feature>LocalDatasource(),
  ),
  Inject<<Feature>Repository>(
    (i) => <Feature>RepositoryImpl(
      i.find<<Feature>Datasource>(),
      i.find<<Feature>LocalDatasource>(),
    ),
  ),
];
```

### Cubit Usage with Stream

```dart
// presentation/controllers/<feature>_controller.dart
class <Feature>Controller extends Cubit<<Feature>State> {
  final <Feature>Repository _repository;
  StreamSubscription? _subscription;

  <Feature>Controller(this._repository) : super(<Feature>Initial());

  void loadAll() {
    emit(<Feature>Loading());
    _subscription?.cancel();
    _subscription = _repository.watchAll().listen(
      (items) => emit(<Feature>Loaded(items)),
      onError: (e) => emit(<Feature>Error(e.toString())),
    );
  }

  @override
  Future<void> close() {
    _subscription?.cancel();
    return super.close();
  }
}
```

---

## 2. Image Caching

When argument is `image-cache`, use `cached_network_image` for product/menu images.

### Widget Usage

```dart
import 'package:cached_network_image/cached_network_image.dart';

class ProductImage extends StatelessWidget {
  final String? imageUrl;
  final double size;

  const ProductImage({super.key, this.imageUrl, this.size = 80});

  @override
  Widget build(BuildContext context) {
    if (imageUrl == null || imageUrl!.isEmpty) {
      return _placeholder();
    }

    return CachedNetworkImage(
      imageUrl: imageUrl!,
      width: size,
      height: size,
      fit: BoxFit.cover,
      placeholder: (_, __) => _placeholder(),
      errorWidget: (_, __, ___) => _placeholder(),
      memCacheWidth: (size * MediaQuery.devicePixelRatioOf(context)).toInt(),
    );
  }

  Widget _placeholder() => SizedBox(
        width: size,
        height: size,
        child: const Icon(Icons.restaurant_menu, color: Colors.grey),
      );
}
```

### Cache Management

```dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter_cache_manager/flutter_cache_manager.dart';

// Clear all cached images (e.g., on logout)
await DefaultCacheManager().emptyCache();

// Clear specific image
await DefaultCacheManager().removeFile(imageUrl);
```

### Where to use

- Listing and detail screens showing remote images
- Selection grids in a checkout or order flow
- Thumbnails in an admin/management table

---

## 3. In-Memory Dio Cache Interceptor

When argument is `memory-cache`, add response caching for GET requests.

### Interceptor Implementation

```dart
// core/lib/shared/services/dio/cache_interceptor.dart
import 'package:dio/dio.dart';

class CacheEntry {
  final Response response;
  final DateTime expiry;

  CacheEntry(this.response, this.expiry);

  bool get isExpired => DateTime.now().isAfter(expiry);
}

class CacheInterceptor extends Interceptor {
  final Map<String, CacheEntry> _cache = {};
  final Duration defaultTtl;

  CacheInterceptor({this.defaultTtl = const Duration(minutes: 5)});

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // Only cache GET requests
    if (options.method != 'GET') {
      handler.next(options);
      return;
    }

    // Skip cache if explicitly requested
    if (options.extra['noCache'] == true) {
      handler.next(options);
      return;
    }

    final key = _cacheKey(options);
    final entry = _cache[key];

    if (entry != null && !entry.isExpired) {
      // Return cached response
      handler.resolve(entry.response.copyWith(
        requestOptions: options,
        extra: {'fromCache': true},
      ));
      return;
    }

    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    if (response.requestOptions.method == 'GET') {
      final key = _cacheKey(response.requestOptions);
      final ttl = response.requestOptions.extra['cacheTtl'] as Duration? ??
          defaultTtl;
      _cache[key] = CacheEntry(response, DateTime.now().add(ttl));
    }
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // On error, try to serve stale cache for GET requests
    if (err.requestOptions.method == 'GET') {
      final key = _cacheKey(err.requestOptions);
      final entry = _cache[key];
      if (entry != null) {
        handler.resolve(entry.response.copyWith(
          requestOptions: err.requestOptions,
          extra: {'fromCache': true, 'stale': true},
        ));
        return;
      }
    }
    handler.next(err);
  }

  void invalidate(String path) {
    _cache.removeWhere((key, _) => key.startsWith(path));
  }

  void clearAll() => _cache.clear();

  String _cacheKey(RequestOptions options) =>
      '${options.path}?${options.queryParameters}';
}
```

### Integration with existing Dio setup

```dart
// In custom_dio_base.dart or custom_dio_native.dart, add to interceptors:
final cacheInterceptor = CacheInterceptor(
  defaultTtl: const Duration(minutes: 5),
);
dio.interceptors.add(cacheInterceptor);
```

### Datasource usage with cache control

```dart
// Skip cache for a specific request
final response = await _dio.get(
  '/products',
  options: Options(extra: {'noCache': true}),
);

// Custom TTL for a specific request
final response = await _dio.get(
  '/store/config',
  options: Options(extra: {'cacheTtl': const Duration(hours: 1)}),
);
```

---

## 4. Sync Strategy for Writes (Local-First)

For offline scenarios, write locally first and sync in the background.

### Pending Write Model

```dart
// data/models/pending_write.dart
class PendingWrite {
  final String id;
  final String endpoint;
  final String method; // POST, PUT, DELETE
  final Map<String, dynamic> payload;
  final DateTime createdAt;
  bool synchronized;

  PendingWrite({
    required this.id,
    required this.endpoint,
    required this.method,
    required this.payload,
    DateTime? createdAt,
    this.synchronized = false,
  }) : createdAt = createdAt ?? DateTime.now();

  Map<String, dynamic> toJson() => {
        'id': id,
        'endpoint': endpoint,
        'method': method,
        'payload': payload,
        'createdAt': createdAt.toIso8601String(),
        'synchronized': synchronized,
      };

  factory PendingWrite.fromJson(Map<String, dynamic> json) => PendingWrite(
        id: json['id'] as String,
        endpoint: json['endpoint'] as String,
        method: json['method'] as String,
        payload: json['payload'] as Map<String, dynamic>,
        createdAt: DateTime.parse(json['createdAt'] as String),
        synchronized: json['synchronized'] as bool? ?? false,
      );
}
```

### Sync Service

```dart
// core/lib/shared/services/sync/sync_service.dart
import 'package:dio/dio.dart';
import 'dart:convert';
import 'package:shared_preferences/shared_preferences.dart';

class SyncService {
  final Dio _dio;
  static const _pendingKey = 'pending_writes';

  SyncService(this._dio);

  Future<void> enqueue(PendingWrite write) async {
    final prefs = await SharedPreferences.getInstance();
    final pending = await _getPending(prefs);
    pending.add(write);
    await _savePending(prefs, pending);
  }

  Future<void> syncAll() async {
    final prefs = await SharedPreferences.getInstance();
    final pending = await _getPending(prefs);
    final failed = <PendingWrite>[];

    for (final write in pending.where((w) => !w.synchronized)) {
      try {
        switch (write.method) {
          case 'POST':
            await _dio.post(write.endpoint, data: write.payload);
          case 'PUT':
            await _dio.put(write.endpoint, data: write.payload);
          case 'DELETE':
            await _dio.delete(write.endpoint);
        }
        write.synchronized = true;
      } catch (_) {
        failed.add(write);
      }
    }

    // Keep only failed writes
    await _savePending(prefs, failed);
  }

  Future<int> get pendingCount async {
    final prefs = await SharedPreferences.getInstance();
    final pending = await _getPending(prefs);
    return pending.where((w) => !w.synchronized).length;
  }

  Future<List<PendingWrite>> _getPending(SharedPreferences prefs) async {
    final json = prefs.getString(_pendingKey);
    if (json == null) return [];
    final list = jsonDecode(json) as List;
    return list
        .map((e) => PendingWrite.fromJson(e as Map<String, dynamic>))
        .toList();
  }

  Future<void> _savePending(
    SharedPreferences prefs,
    List<PendingWrite> writes,
  ) async {
    final json = jsonEncode(writes.map((w) => w.toJson()).toList());
    await prefs.setString(_pendingKey, json);
  }
}
```

---

## 5. Cache Invalidation

### TTL-Based (already built into patterns above)

Set TTL per feature via `_ttlDuration` in local datasource or `cacheTtl` extra in Dio requests.

### Event-Based via Broker

Use the existing `Broker` to invalidate caches when data changes:

```dart
// In repository impl, listen for invalidation events
class <Feature>RepositoryImpl implements <Feature>Repository {
  final <Feature>LocalDatasource _localDatasource;
  final Broker _broker;

  <Feature>RepositoryImpl(
    this._remoteDatasource,
    this._localDatasource,
    this._broker,
  ) {
    // Invalidate cache when related data changes
    _broker.on('<feature>:updated').listen((_) => _localDatasource.clearCache());
    _broker.on('<feature>:created').listen((_) => _localDatasource.clearCache());
    _broker.on('<feature>:deleted').listen((_) => _localDatasource.clearCache());
  }

  // After a write, dispatch event to invalidate other caches
  @override
  Future<void> create(<Feature> entity) async {
    await _remoteDatasource.create(<Feature>Model.fromEntity(entity));
    _broker.dispatch(BrokerEvent(kind: '<feature>:created', data: entity));
  }
}
```

### Manual Invalidation

```dart
// From a controller/cubit, force refresh:
class <Feature>Controller extends Cubit<<Feature>State> {
  Future<void> forceRefresh() async {
    await _localDatasource.clearCache();
    loadAll(); // Re-fetch from remote
  }
}
```

---

## 6. SecureStorage vs SharedPreferences

The project already has `Storage` (SecureStorage) registered in `CoreInjections.services()` with `StorageKeys` enum.

| Use `Storage` (SecureStorage) for | Use `shared_preferences` for |
|---|---|
| Auth tokens (`StorageKeys.auth`) | Cache timestamps / TTL values |
| User credentials (`StorageKeys.credentials`) | Cached API responses (non-sensitive) |
| Payment terminal codes, API keys | UI preferences (grid view, filters) |
| Sensitive user data | Feature flags, last sync time |

**Do NOT** store cached API data in SecureStorage. It is slower and meant for secrets. Use `shared_preferences` or in-memory maps for response caching.

To add new SecureStorage keys, extend the `StorageKeys` enum in `core/lib/shared/services/storage/storage.dart`.

---

## Instructions

1. Determine which caching strategy fits the feature and the surface it runs on
2. For `offline-first`: create local datasource, modify repository impl to yield-then-fetch, register both datasources in CoreInjections
3. For `image-cache`: add `cached_network_image` to the app's pubspec.yaml, create reusable widget in the app's shared/widgets
4. For `memory-cache`: add `CacheInterceptor` to `core/lib/shared/services/dio/`, wire into Dio instance
5. Always invalidate cache on writes - use Broker events for cross-feature invalidation
6. For offline writes, use the `SyncService` with `PendingWrite` pattern
7. Add `shared_preferences` to `core/pubspec.yaml` if not already present
8. Run `/flutter-di` after adding new datasources or services to CoreInjections
