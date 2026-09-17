---
name: flutter-testing
description: Use when writing or fixing Dart/Flutter tests — unit tests for use cases and repositories, widget tests, mocking with mocktail, bloc_test for Cubits, E2E flows with Maestro, or deciding where a test file belongs. Also triggers on a coverage request, "write the test for this", or a failing test.
---

You are implementing tests for a Flutter feature.

## Test Pyramid

```
┌─────────────────────┐
│   E2E (Maestro)     │  Few — critical user flows only
│   Real device/emu   │
├─────────────────────┤
│   Widget Tests      │  Medium — one per screen/widget
│   WidgetTester      │
├─────────────────────┤
│   Unit Tests        │  Many — usecases, repos, models,
│   Pure Dart          │  controllers, extensions
└─────────────────────┘
```

Choose based on the skill argument, if one was given, or on what is being tested:
- `unit` — test usecases, repositories, models, controllers
- `widget` — test screens and widgets with WidgetTester
- `e2e` — create Maestro flow for critical user journey
- `all` (default) — generate all three layers

---

## Mocking Strategy

**Prefer Fake implementations over mocking libraries.** Fakes give better-defined inputs/outputs and are easier to maintain.

**Fake repository (test/fakes/fake_<feature>_repository.dart):**
```dart
import 'package:app_core/domain/entities/<feature>.dart';
import 'package:app_core/domain/repositories/<feature>_repository.dart';

class Fake<Feature>Repository implements <Feature>Repository {
  List<<Feature>> items = [];
  bool shouldThrow = false;

  @override
  Future<List<<Feature>>> getAll() async {
    if (shouldThrow) throw Exception('Fake error');
    return items;
  }

  @override
  Future<<Feature>> getById(String id) async {
    if (shouldThrow) throw Exception('Fake error');
    return items.firstWhere((e) => e.id == id);
  }

  @override
  Future<void> create(<Feature> entity) async {
    if (shouldThrow) throw Exception('Fake error');
    items.add(entity);
  }

  @override
  Future<void> update(<Feature> entity) async {
    if (shouldThrow) throw Exception('Fake error');
    final index = items.indexWhere((e) => e.id == entity.id);
    items[index] = entity;
  }

  @override
  Future<void> delete(String id) async {
    if (shouldThrow) throw Exception('Fake error');
    items.removeWhere((e) => e.id == id);
  }
}
```

**Use `mocktail` only when Fake is impractical** (e.g., Dio, Storage, third-party interfaces):
```dart
import 'package:mocktail/mocktail.dart';
import 'package:dio/dio.dart';

class MockDio extends Mock implements Dio {}
class MockStorage extends Mock implements Storage {}
```

**Test fixtures (test/fixtures/<feature>_fixtures.dart):**
```dart
import 'package:app_core/domain/entities/<feature>.dart';

class <Feature>Fixtures {
  static <Feature> get single => const <Feature>(
    id: 'test-id-1',
    name: 'Test <Feature>',
  );

  static List<<Feature>> get list => [
    const <Feature>(id: 'test-id-1', name: 'First'),
    const <Feature>(id: 'test-id-2', name: 'Second'),
  ];

  static Map<String, dynamic> get json => {
    'id': 'test-id-1',
    'name': 'Test <Feature>',
  };
}
```

---

## Unit Tests

### Testing Models (JSON serialization)

**test/data/models/<feature>_model_test.dart:**
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:app_core/data/models/<feature>_model.dart';
import '../../fixtures/<feature>_fixtures.dart';

void main() {
  group('<Feature>Model', () {
    test('fromJson creates correct model', () {
      final model = <Feature>Model.fromJson(<Feature>Fixtures.json);

      expect(model.id, 'test-id-1');
      expect(model.name, 'Test <Feature>');
    });

    test('toJson returns correct map', () {
      final model = <Feature>Model(id: 'test-id-1', name: 'Test <Feature>');
      final json = model.toJson();

      expect(json['id'], 'test-id-1');
      expect(json['name'], 'Test <Feature>');
    });

    test('toEntity converts to domain entity', () {
      final model = <Feature>Model(id: 'test-id-1', name: 'Test <Feature>');
      final entity = model.toEntity();

      expect(entity, isA<<Feature>>());
      expect(entity.id, model.id);
    });
  });
}
```

### Testing Usecases

**test/domain/usecases/<feature>/get_<feature>s_test.dart:**
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:app_core/domain/usecases/<feature>/get_<feature>s_usecase.dart';
import '../../../fakes/fake_<feature>_repository.dart';
import '../../../fixtures/<feature>_fixtures.dart';

void main() {
  late Fake<Feature>Repository repository;
  late Get<Feature>sUsecase usecase;

  setUp(() {
    repository = Fake<Feature>Repository();
    usecase = Get<Feature>sUsecase(repository);
  });

  group('Get<Feature>sUsecase', () {
    test('returns list from repository', () async {
      repository.items = <Feature>Fixtures.list;

      final result = await usecase.call();

      expect(result, hasLength(2));
      expect(result.first.id, 'test-id-1');
    });

    test('throws when repository fails', () async {
      repository.shouldThrow = true;

      expect(() => usecase.call(), throwsException);
    });
  });
}
```

### Testing Controllers (Cubits)

**test/presentation/<feature>_controller_test.dart:**
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:bloc_test/bloc_test.dart';
import 'package:<app>/modules/<feature>/presentation/<feature>_controller.dart';
import 'package:<app>/modules/<feature>/presentation/<feature>_status.dart';
import '../../fakes/fake_<feature>_repository.dart';
import '../../fixtures/<feature>_fixtures.dart';

void main() {
  late Fake<Feature>Repository repository;
  late Get<Feature>sUsecase usecase;

  setUp(() {
    repository = Fake<Feature>Repository();
    usecase = Get<Feature>sUsecase(repository);
  });

  group('<Feature>Controller', () {
    blocTest<'<Feature>Controller, <Feature>Status'>(
      'emits [Loading, Success] when load succeeds',
      setUp: () => repository.items = <Feature>Fixtures.list,
      build: () => <Feature>Controller(usecase),
      act: (controller) => controller.load(),
      expect: () => [
        isA<<Feature>Loading>(),
        isA<<Feature>Success>()
            .having((s) => s.items, 'items', hasLength(2)),
      ],
    );

    blocTest<'<Feature>Controller, <Feature>Status'>(
      'emits [Loading, Failure] when load fails',
      setUp: () => repository.shouldThrow = true,
      build: () => <Feature>Controller(usecase),
      act: (controller) => controller.load(),
      expect: () => [
        isA<<Feature>Loading>(),
        isA<<Feature>Failure>(),
      ],
    );
  });
}
```

### Testing Repositories

**test/data/repositories/<feature>_repository_impl_test.dart:**
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:app_core/data/datasources/<feature>/<feature>_datasource.dart';
import 'package:app_core/data/models/<feature>_model.dart';
import 'package:app_core/data/repositories/<feature>_repository_impl.dart';

class Mock<Feature>Datasource extends Mock implements <Feature>Datasource {}

void main() {
  late Mock<Feature>Datasource datasource;
  late <Feature>RepositoryImpl repository;

  setUp(() {
    datasource = Mock<Feature>Datasource();
    repository = <Feature>RepositoryImpl(datasource);
  });

  group('<Feature>RepositoryImpl', () {
    test('getAll returns entities mapped from models', () async {
      when(() => datasource.getAll()).thenAnswer((_) async => [
        <Feature>Model(id: '1', name: 'Test'),
      ]);

      final result = await repository.getAll();

      expect(result.first.id, '1');
      verify(() => datasource.getAll()).called(1);
    });
  });
}
```

---

## Widget Tests

### Testing Screens

**test/modules/<feature>/presentation/<feature>_screen_test.dart:**
```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:bloc_test/bloc_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:<app>/modules/<feature>/presentation/<feature>_controller.dart';
import 'package:<app>/modules/<feature>/presentation/<feature>_status.dart';
import 'package:<app>/modules/<feature>/presentation/<feature>_screen.dart';
import '../../../fixtures/<feature>_fixtures.dart';

class Mock<Feature>Controller extends MockCubit<<Feature>Status>
    implements <Feature>Controller {}

void main() {
  late Mock<Feature>Controller controller;

  setUp(() {
    controller = Mock<Feature>Controller();
  });

  Widget buildSubject() {
    return MaterialApp(
      home: BlocProvider<<Feature>Controller>.value(
        value: controller,
        child: const <Feature>Screen(),
      ),
    );
  }

  group('<Feature>Screen', () {
    testWidgets('shows loading indicator when loading', (tester) async {
      when(() => controller.state).thenReturn(const <Feature>Loading());

      await tester.pumpWidget(buildSubject());

      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('shows list when loaded', (tester) async {
      when(() => controller.state)
          .thenReturn(<Feature>Success(<Feature>Fixtures.list));

      await tester.pumpWidget(buildSubject());

      expect(find.text('First'), findsOneWidget);
      expect(find.text('Second'), findsOneWidget);
    });

    testWidgets('shows error message on failure', (tester) async {
      when(() => controller.state)
          .thenReturn(const <Feature>Failure('Something went wrong'));

      await tester.pumpWidget(buildSubject());

      expect(find.text('Something went wrong'), findsOneWidget);
    });
  });
}
```

### Testing Individual Widgets

```dart
testWidgets('<Feature>Card displays entity data', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: <Feature>Card(entity: <Feature>Fixtures.single),
      ),
    ),
  );

  expect(find.text('Test <Feature>'), findsOneWidget);
});

testWidgets('<Feature>Card tap triggers callback', (tester) async {
  var tapped = false;

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: <Feature>Card(
          entity: <Feature>Fixtures.single,
          onTap: () => tapped = true,
        ),
      ),
    ),
  );

  await tester.tap(find.byType(<Feature>Card));
  expect(tapped, isTrue);
});
```

---

## E2E Tests (Maestro)

Maestro testa fluxos completos no device real. Usar apenas para jornadas críticas.

### Setup

Instalar Maestro: `curl -Ls "https://get.maestro.mobile.dev" | bash`

### Estrutura de diretórios

```
<app>/
├── .maestro/
│   ├── config.yaml
│   └── flows/
│       ├── login_flow.yaml
│       ├── create_order_flow.yaml
│       └── payment_flow.yaml
```

### Config (.maestro/config.yaml)

```yaml
appId: com.example.<app>
name: <App> E2E
```

### Flow: Login

**.maestro/flows/login_flow.yaml:**
```yaml
appId: com.example.<app>
name: Login Flow
---
- launchApp
- tapOn: "E-mail"
- inputText: "test@example.com"
- tapOn: "Senha"
- inputText: "Test@123"
- tapOn: "Entrar"
- assertVisible: "Home"
```

### Flow: create a record

**.maestro/flows/create_item_flow.yaml:**
```yaml
appId: com.example.myapp
name: Create Item Flow
---
- launchApp
- runFlow: login_flow.yaml

# Open the creation screen
- tapOn: "New item"
- assertVisible: "Item"

# Add a child record
- tapOn: "Add"
- tapOn:
    id: "product_item_0"
- tapOn: "Confirm"

# Verify it was created
- assertVisible: "Item"
- assertVisible: "1 item"
```

### Flow: checkout / payment

**.maestro/flows/payment_flow.yaml:**
```yaml
appId: com.example.myapp
name: Payment Flow
---
- launchApp
# Assume logged in — run after login_flow
- runFlow: login_flow.yaml

# Navigate to an existing record
- tapOn: "Items"
- tapOn:
    id: "item_row_0"

# Start payment
- tapOn: "Payment"
- assertVisible: "Payment method"
- tapOn: "Cash"
- tapOn: "Confirm"

# Verify success
- assertVisible: "Payment complete"
```

Prefer `id:` selectors over visible text in E2E flows — text changes with copy
edits and with the device locale, and a flow that matches on it breaks for
reasons that have nothing to do with the feature.

### Running E2E flows

```bash
# Fluxo individual
maestro test .maestro/flows/login_flow.yaml

# Todos os fluxos
maestro test .maestro/flows/

# Com device específico
maestro test --device emulator-5554 .maestro/flows/
```

---

## Test File Organization

```
<package>/test/
├── fakes/                          # Fake implementations
│   ├── fake_<feature>_repository.dart
│   └── fake_session.dart
├── fixtures/                       # Test data factories
│   └── <feature>_fixtures.dart
├── data/
│   ├── models/                     # Model serialization tests
│   │   └── <feature>_model_test.dart
│   └── repositories/              # Repository impl tests
│       └── <feature>_repository_impl_test.dart
├── domain/
│   └── usecases/                  # Usecase tests
│       └── <feature>/
│           └── get_<feature>s_test.dart
└── modules/                       # Widget + controller tests (apps only)
    └── <feature>/
        └── presentation/
            ├── <feature>_controller_test.dart
            └── <feature>_screen_test.dart
```

---

## Required Dependencies

Add to `pubspec.yaml` `dev_dependencies` of the package being tested:

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.4
  bloc_test: ^9.1.7
```

Only add `mockito` + `build_runner` if you need code generation for complex interfaces. Prefer `mocktail` (no codegen).

---

## What to Test per Layer

| Layer | What to test | What NOT to test |
|-------|-------------|-----------------|
| **Model** | fromJson, toJson, toEntity, fromEntity | Constructor (trivial) |
| **Datasource** | Skip — tested via repository integration | HTTP calls (mock Dio) |
| **Repository** | Model→Entity mapping, error handling | Datasource internals |
| **Usecase** | Orchestration logic, param validation | Repository internals |
| **Controller** | State transitions (Loading→Success/Failure) | UI rendering |
| **Screen** | Renders correct state, user interactions | Business logic |
| **Widget** | Display, callbacks, edge cases (empty/long text) | Internal state |

---

## Test Naming Convention

```
test('<Subject> <action> <expected result>', () { ... });
```

Examples:
- `'ProductModel fromJson creates model with correct fields'`
- `'GetOrdersUsecase returns empty list when no orders'`
- `'LoginController emits Failure when credentials invalid'`
- `'ProductCard displays product name and price'`

---

## Instructions

1. Read the existing feature code before writing tests
2. Create `fakes/` and `fixtures/` first — reuse across all test files
3. Add `mocktail` and `bloc_test` to `dev_dependencies` if missing
4. Run tests: `make test PROJECT=<package>` or `cd <package> && fvm flutter test test/path/to_specific_test.dart`
5. For E2E, install Maestro and create `.maestro/` in the app directory
6. Follow the test pyramid — many unit, fewer widget, few E2E
7. Every new feature created with `/flutter-clean-arch` should get tests via this skill
