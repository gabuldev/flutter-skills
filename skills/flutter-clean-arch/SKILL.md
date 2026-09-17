---
name: flutter-clean-arch
description: Use when creating or auditing the layers of a Flutter feature or package - where a use case, repository, datasource, entity or model belongs; whether domain may import Flutter; whether a dependency between layers is inverted. Triggers on "where does this go?" in a feature, or a request for an architecture review.
---

You are working with Clean Architecture in a Flutter project.

## Layer Responsibilities

```
┌─────────────────────────────────────────┐
│  PRESENTATION                           │
│  Modules, Screens, Controllers (Cubits) │
│  Only knows: Domain entities + Usecases │
├─────────────────────────────────────────┤
│  DOMAIN (pure Dart, no Flutter)         │
│  Entities, Repository interfaces,       │
│  Usecases                               │
│  Depends on: nothing                    │
├─────────────────────────────────────────┤
│  DATA                                   │
│  Models, Datasources, Repo impls        │
│  Depends on: Domain interfaces + Dio    │
└─────────────────────────────────────────┘
```

## Full Feature Scaffold

When argument is `scaffold` or no argument, create all layers for `<feature_name>`:

### Domain Layer

**entity (domain/entities/<feature>.dart):**
```dart
import 'package:equatable/equatable.dart';

class <Feature> extends Equatable {
  final String id;
  final String name;
  // Add all business-relevant fields (no JSON, no framework types)

  const <Feature>({
    required this.id,
    required this.name,
  });

  @override
  List<Object?> get props => [id, name];

  <Feature> copyWith({String? id, String? name}) => <Feature>(
    id: id ?? this.id,
    name: name ?? this.name,
  );
}
```

**repository interface (domain/repositories/<feature>_repository.dart):**
```dart
import '../entities/<feature>.dart';

abstract class <Feature>Repository {
  Future<List<<Feature>>> getAll();
  Future<<Feature>> getById(String id);
  Future<void> create(<Feature> entity);
  Future<void> update(<Feature> entity);
  Future<void> delete(String id);
}
```

**usecases (domain/usecases/<feature>/):**

Each usecase = one file, one responsibility:
```dart
// get_<feature>s_usecase.dart
class Get<Feature>sUsecase {
  final <Feature>Repository _repository;
  Get<Feature>sUsecase(this._repository);

  Future<List<<Feature>>> call() => _repository.getAll();
}

// create_<feature>_usecase.dart
class Create<Feature>Usecase {
  final <Feature>Repository _repository;
  Create<Feature>Usecase(this._repository);

  Future<void> call(<Feature> entity) => _repository.create(entity);
}
```

### Data Layer

**model (data/models/<feature>_model.dart):**
```dart
import '../../domain/entities/<feature>.dart';

class <Feature>Model {
  final String id;
  final String name;

  const <Feature>Model({required this.id, required this.name});

  factory <Feature>Model.fromJson(Map<String, dynamic> json) =>
      <Feature>Model(
        id:   json['id']   as String,
        name: json['name'] as String,
      );

  Map<String, dynamic> toJson() => {'id': id, 'name': name};

  <Feature> toEntity() => <Feature>(id: id, name: name);

  static <Feature>Model fromEntity(<Feature> entity) =>
      <Feature>Model(id: entity.id, name: entity.name);
}
```

**datasource (data/datasources/<feature>_datasource.dart):**
```dart
import 'package:dio/dio.dart';
import '../models/<feature>_model.dart';

class <Feature>Datasource {
  final Dio _dio;
  <Feature>Datasource(this._dio);

  Future<List<<Feature>Model>> getAll() async {
    final response = await _dio.get('/<feature>s');
    return (response.data as List)
        .map((e) => <Feature>Model.fromJson(e as Map<String, dynamic>))
        .toList();
  }

  Future<<Feature>Model> getById(String id) async {
    final response = await _dio.get('/<feature>s/$id');
    return <Feature>Model.fromJson(response.data as Map<String, dynamic>);
  }

  Future<void> create(<Feature>Model model) async {
    await _dio.post('/<feature>s', data: model.toJson());
  }

  Future<void> update(<Feature>Model model) async {
    await _dio.put('/<feature>s/${model.id}', data: model.toJson());
  }

  Future<void> delete(String id) async {
    await _dio.delete('/<feature>s/$id');
  }
}
```

**repository impl (data/repositories/<feature>_repository_impl.dart):**
```dart
import '../../domain/entities/<feature>.dart';
import '../../domain/repositories/<feature>_repository.dart';
import '../datasources/<feature>_datasource.dart';
import '../models/<feature>_model.dart';

class <Feature>RepositoryImpl implements <Feature>Repository {
  final <Feature>Datasource _dataSource;
  <Feature>RepositoryImpl(this._dataSource);

  @override
  Future<List<<Feature>>> getAll() async {
    final models = await _dataSource.getAll();
    return models.map((m) => m.toEntity()).toList();
  }

  @override
  Future<<Feature>> getById(String id) async {
    final model = await _dataSource.getById(id);
    return model.toEntity();
  }

  @override
  Future<void> create(<Feature> entity) =>
      _dataSource.create(<Feature>Model.fromEntity(entity));

  @override
  Future<void> update(<Feature> entity) =>
      _dataSource.update(<Feature>Model.fromEntity(entity));

  @override
  Future<void> delete(String id) => _dataSource.delete(id);
}
```

## When argument is `audit`

Read the existing code for `<feature_name>` and check:

1. **Domain isolation**: Do entities import anything from Flutter/Dio/external? (They shouldn't)
2. **Repository pattern**: Does the datasource return models? Does the repo impl map to entities?
3. **Usecase single responsibility**: Does each usecase do exactly one thing?
4. **Dependency direction**: Does data layer depend on domain? (Yes, correct) Does domain depend on data? (No, wrong)
5. **Model ↔ Entity separation**: Are API models separate from domain entities?
6. **Exception handling**: Are data-layer exceptions caught and re-thrown as domain exceptions?

Report findings and fix any violations.

## Architecture Rules

1. **Domain layer has zero external dependencies** — pure Dart, no Flutter, no Dio
2. **Entities use Equatable** — value equality for BLoC state comparisons
3. **Models handle JSON** — entities never have `fromJson`/`toJson`
4. **Repository interface in domain** — implementation in data
5. **Usecases are thin** — orchestrate repo calls, no business logic in datasources
6. **One usecase = one operation** — don't create `MenuUsecases` with 5 methods
7. **Datasources return models** — repositories return entities
8. **Exceptions**: datasource throws HTTP exceptions, repository catches and rethrows domain exceptions

## Instructions

1. Read existing domain/data files before creating new ones
2. Check `core/lib/injections.dart` to see what's already registered
3. Only create layers that don't exist yet
4. After scaffolding, run `/flutter-di` to register new dependencies
5. After DI, run `/flutter-state` to create the Cubit
