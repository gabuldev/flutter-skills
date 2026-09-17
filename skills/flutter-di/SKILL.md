---
name: flutter-di
description: Use when registering or resolving a dependency with flutter_injections - adding a registration to the core injections, assembling a feature FlutterModule, choosing between singleton/lazySingleton/factory, or calling FlutterInjections.get. Triggers on a runtime "dependency not registered" error.
---

You are configuring dependency injection for a Flutter feature using `flutter_injections`.

## DI Architecture Overview

```
FlutterInjectionsWidget (root, in AppWidget)
  └── CoreInjections.core()        ← global singletons (Dio, repos, session)
       ├── auth()
       ├── services()
       ├── eventBus()
       └── ...

FlutterInjectionsScope (per-module, lazy)
  └── HomeModule.injections        ← scoped to module lifetime
       ├── HomeController
       └── HomeConsumer
```

## Registration Lifetimes

| Type | Usage | Pattern |
|------|-------|---------|
| `singleton` | One instance for app lifetime | Dio, Session, Broker |
| `lazySingleton` | Created on first use, then kept | Controllers, Repositories |
| `factory` | New instance every `get<T>()` call | Usecases, one-time actions |

## Step 1: Register in CoreInjections

File: `core/lib/injections.dart`

Add a new static method for the domain group:

```dart
class CoreInjections {
  static List<Inject<Object>> core() => [
    ...auth(),
    ...services(),
    ...eventBus(),
    ...<module_name>(),  // ← ADD THIS
  ];

  // ← ADD THIS METHOD
  static List<Inject<Object>> <module_name>() => [
    Inject<<ModuleName>Datasource>(
      (i) => <ModuleName>Datasource(i.find<Dio>()),
    ),
    Inject<<ModuleName>Repository>(
      (i) => <ModuleName>RepositoryImpl(i.find<<ModuleName>Datasource>()),
    ),
  ];
}
```

## Step 2: Register controller in the Module

File: `modules/<module_name>/<module_name>_module.dart`

```dart
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
        create: (_) => FlutterInjections.get<<ModuleName>Controller>(),
        child: const <ModuleName>Screen(),
      ),
    );
  }
}
```

## Step 3: Resolve dependencies

**In widgets/screens (via BlocProvider):**
```dart
context.read<<ModuleName>Controller>()
```

**Programmatic resolution (outside widget tree):**
```dart
final controller = FlutterInjections.get<<ModuleName>Controller>();
```

**Inside an Inject factory (chain resolution):**
```dart
Inject<OrderController>.lazySingleton(
  (i) => OrderController(
    getOrders: i.find<GetOrdersUsecase>(),
    broker:    i.find<Broker>(),
  ),
),
```

## Patterns to Follow

### Usecase as factory (stateless, safe to create per-use)
```dart
Inject<Get<ModuleName>Usecase>(
  (i) => Get<ModuleName>Usecase(i.find<<ModuleName>Repository>()),
),
```

### Controller as lazySingleton (stateful, one per scope)
```dart
Inject<<ModuleName>Controller>.lazySingleton(
  (i) => <ModuleName>Controller(i.find<Get<ModuleName>Usecase>()),
),
```

### Repository as singleton (shared data layer)
```dart
Inject<<ModuleName>Repository>(
  (i) => <ModuleName>RepositoryImpl(i.find<<ModuleName>Datasource>()),
),
```

## Event Bus DI

When a module needs to publish/subscribe events via Broker:

```dart
Inject<PaymentConsumer>(
  (i) => PaymentConsumer(broker: i.find<Broker>()),
),
```

## Common Mistakes to Avoid

1. **Do NOT** register controllers in `CoreInjections` — they belong in the module scope
2. **Do NOT** register usecases as singletons — they are stateless and should be factories
3. **Always** use `i.find<T>()` inside Inject factory, never `FlutterInjections.get<T>()` (which bypasses scope)
4. **Order matters** — list dependencies before dependents in the injections list

## Instructions

1. Read `core/lib/injections.dart` to understand existing registrations
2. Find the right scope: global (CoreInjections) vs module-scoped (Module.injections)
3. Apply the correct lifetime: singleton/lazySingleton/factory
4. Register datasource → repository in Core; usecase + controller in Module
5. Check for circular dependencies before registering
