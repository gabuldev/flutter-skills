---
name: flutter-new-module
description: Use when creating a new feature or module in a Flutter app (lib/modules/<feature>/) - generates the full skeleton: module with DI, Cubit controller, sealed status class, screen, widgets and route. Triggers on "new screen", "new feature" or "new module".
---

You are creating a complete feature module for a scalable Flutter project - Cubit for state, `flutter_injections` for DI, one folder per feature under `lib/modules/`.

## Input
- Module name: from the request, or from the skill argument (e.g., "payment", "order", "profile").
  **If no name was given, ask before creating any file** — it drives every directory,
  file and class name below, and renaming afterwards touches all of them.
- If a second argument is given, it's the package path (e.g., "../my_app/lib")
- If no path given, use the current package's `lib/` directory

## Module Structure to Create

Create ALL the following files for module `<module_name>`:

```
modules/<module_name>/
├── <module_name>_module.dart          ← Module entry + DI registration
└── presentation/
    ├── <module_name>_controller.dart  ← Cubit (state machine)
    ├── <module_name>_status.dart      ← State classes (sealed/abstract)
    ├── <module_name>_screen.dart      ← Main UI screen
    └── widgets/                       ← Module-specific widgets (if needed)
```

And in core (if new domain is needed):
```
domain/
├── entities/<module_name>.dart        ← Immutable entity (Equatable)
├── repositories/<module_name>_repository.dart  ← Abstract interface
└── usecases/<module_name>/
    └── get_<module_name>_usecase.dart

data/
├── models/<module_name>_model.dart    ← API model with .toEntity()
├── datasources/<module_name>_datasource.dart  ← Dio HTTP calls
└── repositories/<module_name>_repository_impl.dart
```

## File Templates

### 1. Entity (domain/entities/<module_name>.dart)
```dart
import 'package:equatable/equatable.dart';

class ModuleName extends Equatable {
  final String id;
  // add fields

  const ModuleName({required this.id});

  @override
  List<Object?> get props => [id];
}
```

### 2. Repository interface (domain/repositories/<module_name>_repository.dart)
```dart
import '../entities/<module_name>.dart';

abstract class <ModuleName>Repository {
  Future<List<ModuleName>> getAll();
}
```

### 3. Usecase (domain/usecases/<module_name>/get_<module_name>_usecase.dart)
```dart
import '../../repositories/<module_name>_repository.dart';
import '../../entities/<module_name>.dart';

class Get<ModuleName>Usecase {
  final <ModuleName>Repository _repository;
  Get<ModuleName>Usecase(this._repository);

  Future<List<ModuleName>> call() => _repository.getAll();
}
```

### 4. API Model (data/models/<module_name>_model.dart)
```dart
import '../../domain/entities/<module_name>.dart';

class <ModuleName>Model {
  final String id;
  // add fields from API

  <ModuleName>Model({required this.id});

  factory <ModuleName>Model.fromJson(Map<String, dynamic> json) =>
      <ModuleName>Model(id: json['id']);

  <ModuleName> toEntity() => <ModuleName>(id: id);
}
```

### 5. Datasource (data/datasources/<module_name>_datasource.dart)
```dart
import 'package:dio/dio.dart';
import '../models/<module_name>_model.dart';

class <ModuleName>Datasource {
  final Dio _dio;
  <ModuleName>Datasource(this._dio);

  Future<List<<ModuleName>Model>> getAll() async {
    final response = await _dio.get('/<module_name>s');
    return (response.data as List)
        .map((e) => <ModuleName>Model.fromJson(e))
        .toList();
  }
}
```

### 6. Repository impl (data/repositories/<module_name>_repository_impl.dart)
```dart
import '../../domain/entities/<module_name>.dart';
import '../../domain/repositories/<module_name>_repository.dart';
import '../datasources/<module_name>_datasource.dart';

class <ModuleName>RepositoryImpl implements <ModuleName>Repository {
  final <ModuleName>Datasource _dataSource;
  <ModuleName>RepositoryImpl(this._dataSource);

  @override
  Future<List<ModuleName>> getAll() async {
    final result = await _dataSource.getAll();
    return result.map((e) => e.toEntity()).toList();
  }
}
```

### 7. Status (presentation/<module_name>_status.dart)
```dart
abstract class <ModuleName>Status {
  const <ModuleName>Status();
}

class <ModuleName>Initial extends <ModuleName>Status {
  const <ModuleName>Initial();
}

class <ModuleName>Loading extends <ModuleName>Status {
  const <ModuleName>Loading();
}

class <ModuleName>Success extends <ModuleName>Status {
  final List<<Entity>> items;
  const <ModuleName>Success(this.items);
}

class <ModuleName>Failure extends <ModuleName>Status {
  final String message;
  const <ModuleName>Failure(this.message);
}
```

### 8. Controller (presentation/<module_name>_controller.dart)
```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:app_core/domain/usecases/<module_name>/get_<module_name>_usecase.dart';
import '<module_name>_status.dart';

class <ModuleName>Controller extends Cubit<<ModuleName>Status> {
  final Get<ModuleName>Usecase _get<ModuleName>;

  <ModuleName>Controller(this._get<ModuleName>) : super(const <ModuleName>Initial());

  Future<void> load() async {
    emit(const <ModuleName>Loading());
    try {
      final items = await _get<ModuleName>.call();
      emit(<ModuleName>Success(items));
    } catch (e) {
      emit(<ModuleName>Failure(e.toString()));
    }
  }
}
```

### 9. Screen (presentation/<module_name>_screen.dart)
```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '<module_name>_controller.dart';
import '<module_name>_status.dart';

class <ModuleName>Screen extends StatefulWidget {
  const <ModuleName>Screen({super.key});

  @override
  State<<ModuleName>Screen> createState() => _<ModuleName>ScreenState();
}

class _<ModuleName>ScreenState extends State<<ModuleName>Screen> {
  @override
  void initState() {
    super.initState();
    context.read<<ModuleName>Controller>().load();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('<ModuleName>')),
      body: BlocBuilder<<ModuleName>Controller, <ModuleName>Status>(
        builder: (context, status) {
          if (status is <ModuleName>Loading) {
            return const Center(child: CircularProgressIndicator());
          }
          if (status is <ModuleName>Failure) {
            return Center(child: Text(status.message));
          }
          if (status is <ModuleName>Success) {
            return ListView.builder(
              itemCount: status.items.length,
              itemBuilder: (context, index) {
                final item = status.items[index];
                return ListTile(title: Text(item.id));
              },
            );
          }
          return const SizedBox.shrink();
        },
      ),
    );
  }
}
```

### 10. Module entry + DI (<module_name>_module.dart)
```dart
import 'package:flutter/material.dart';
import 'package:flutter_injections/flutter_injections.dart';
import 'package:app_core/domain/usecases/<module_name>/get_<module_name>_usecase.dart';
import 'package:app_core/domain/repositories/<module_name>_repository.dart';
import 'presentation/<module_name>_controller.dart';
import 'presentation/<module_name>_screen.dart';

class <ModuleName>Module extends StatelessWidget {
  const <ModuleName>Module({super.key});

  List<Inject<Object>> get injections => [
    Inject<<ModuleName>Controller>.lazySingleton(
      (i) => <ModuleName>Controller(
        Get<ModuleName>Usecase(i.find<<ModuleName>Repository>()),
      ),
    ),
  ];

  @override
  Widget build(BuildContext context) {
    return FlutterInjectionsScope(
      injections: injections,
      child: BlocProvider(
        create: (context) => FlutterInjections.get<<ModuleName>Controller>(),
        child: const <ModuleName>Screen(),
      ),
    );
  }
}
```

## Instructions

1. Read the existing project structure first to understand conventions
2. Create all files above substituting `<ModuleName>` and `<module_name>` with the actual name
3. Register the new repository and datasource in `CoreInjections` in `core/lib/injections.dart`
4. Add the new route to the app's `app_routes.dart`
5. Use the same import alias convention already in the project
6. Do NOT add documentation comments or unnecessary annotations
7. Keep controllers lean — only state transitions in controller, logic in usecases
