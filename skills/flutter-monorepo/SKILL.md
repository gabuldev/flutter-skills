---
name: flutter-monorepo
description: Use when setting up or extending a Melos-based Flutter monorepo - adding a new package, editing melos.yaml, fixing bootstrap, wiring a dependency between packages, or defining per-app environment variables. Triggers when creating a new Dart or Flutter package.
---

You are working with a Melos-based Flutter monorepo.

## Monorepo Layout

```
<project_root>/
├── melos.yaml              <- Melos workspace config
├── pubspec.yaml            <- Workspace root (required for Melos 4+)
├── Makefile                <- Developer commands (setup, dev, build, test)
├── core/                   <- Shared domain + data layer
│   ├── pubspec.yaml
│   └── lib/
│       ├── domain/
│       ├── data/
│       ├── shared/
│       └── injections.dart
├── design_system/          <- UI tokens + shared components
│   ├── pubspec.yaml
│   └── lib/
├── <app1>/                 <- Flutter app (e.g., mobile, web, admin)
│   ├── pubspec.yaml
│   └── lib/
│       ├── modules/
│       ├── app_routes.dart
│       └── app_widget.dart
└── <app2>/
```

## When argument is "setup"

### 1. melos.yaml
```yaml
name: <project_name>

packages:
  - core
  - design_system
  - "<app1>"
  - "<app2>"

scripts:
  bootstrap:
    run: melos bootstrap
    description: Bootstrap all packages

  test:
    run: melos exec -- flutter test
    description: Run tests in all packages

  build:runner:
    run: melos exec --depends-on="build_runner" -- dart run build_runner build --delete-conflicting-outputs
    description: Run build_runner in all packages
```

### 2. Root pubspec.yaml (required for Melos 4+)
```yaml
name: <project_name>_workspace
publish_to: 'none'

environment:
  sdk: '>=3.0.0 <4.0.0'

dev_dependencies:
  melos: ^4.1.0
```

### 3. Core package (core/pubspec.yaml)
```yaml
name: <project_name>_core
publish_to: 'none'

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  dio: ^5.5.0
  flutter_injections: ^1.1.1
  flutter_secure_storage: ^9.2.2
  equatable: ^2.0.5
  rxdart: ^0.28.0
  uuid: ^4.5.1
  intl: any

dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.4
  build_runner: ^2.4.13
```

### 4. App package pubspec.yaml
```yaml
name: <project_name>_<app>
publish_to: 'none'

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  <project_name>_core:
    path: ../core
  <project_name>_design_system:
    path: ../design_system
  bloc: ^8.1.4
  flutter_bloc: ^8.1.6
  flutter_injections: ^1.1.1
  equatable: ^2.0.5

dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.4
```

### 5. Makefile
```makefile
SHELL := /bin/bash
FVM := $(shell which fvm 2>/dev/null || echo "$(HOME)/.pub-cache/bin/fvm")
MELOS := $(shell which melos 2>/dev/null || echo "$(HOME)/.pub-cache/bin/melos")
FLUTTER := $(FVM) flutter

.PHONY: setup dev build test clean

setup: setup-fvm setup-deps

setup-fvm:
	@echo "Setting up FVM..."
	dart pub global activate fvm
	$(FVM) install

setup-deps:
	@echo "Installing dependencies..."
	dart pub get
	$(MELOS) bootstrap

dev-<app>:
	@echo "Starting <app>..."
	cd <app> && $(FLUTTER) run --dart-define-from-file=.env

build-<app>:
	@echo "Building <app>..."
	cd <app> && $(FLUTTER) build web --dart-define-from-file=.env

test:
	$(MELOS) test

clean:
	$(MELOS) exec -- flutter clean
```

### 6. CoreInjections scaffold
```dart
import 'package:dio/dio.dart';
import 'package:flutter_injections/flutter_injections.dart';

class CoreInjections {
  static List<Inject<Object>> core() => [
    ...services(),
  ];

  static List<Inject<Object>> services() => [
    Inject<Dio>.singleton((i) => Dio()),
  ];
}
```

### 7. AppWidget scaffold
```dart
import 'package:flutter/material.dart';
import 'package:flutter_injections/flutter_injections.dart';
import 'package:<project>_core/injections.dart';
import 'app_routes.dart';

class AppWidget extends StatelessWidget {
  const AppWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return FlutterInjectionsWidget(
      injections: CoreInjections.core(),
      child: MaterialApp(
        debugShowCheckedModeBanner: false,
        onGenerateRoute: AppRoutes.onGenerateRoute,
        initialRoute: '/',
      ),
    );
  }
}
```

## When argument is "add-package <name>"

1. Create `<name>/pubspec.yaml` with the app package template above
2. Create `<name>/lib/main.dart`, `app_widget.dart`, `app_routes.dart`
3. Add `- "<name>"` to `melos.yaml` packages list
4. Run `melos bootstrap` to link packages

## Environment Variables Pattern

Each app gets a `.env` file (gitignored) and a `.env.example`:

**.env.example:**
```
BASE_URL=https://api.example.com
X_API_KEY=your_key_here
```

**AppEnv pattern:**
```dart
class AppEnv {
  static String baseURL = const String.fromEnvironment('BASE_URL');
  static String xApiKey = const String.fromEnvironment('X_API_KEY');
}
```

**Run with env:**
```bash
flutter run --dart-define-from-file=.env
```

## Instructions

1. Read existing `melos.yaml` and root `pubspec.yaml` before making changes
2. For new packages, always use `path:` dependencies for sibling packages
3. Keep `core` as pure shared logic — no app-specific code
4. Design system belongs in `design_system/` package, not `core/`
5. Each app registers its own modules + CoreInjections in AppWidget
6. After setup, run `make setup` to verify everything bootstraps correctly
