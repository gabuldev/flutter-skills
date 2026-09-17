---
name: flutter-state
description: Use when implementing or reviewing Flutter state management with Cubit - a sealed status class with variants, BlocBuilder with pattern matching, a Cubit fed by a Stream for realtime data, or an event bus for cross-module communication. Triggers when a screen has to react to data that changes.
---

You are implementing state management for a Flutter feature using Cubit + RxDart.

## Three Patterns by Use Case

Choose based on the skill argument, if one was given, or on the context of the feature:

- `simple` — basic async operations (load data, submit form)
- `stream` — real-time data via RxDart BehaviorSubject
- `event-bus` — cross-module communication via Broker

---

## Pattern 1: Simple Cubit (default)

Best for: load list, submit form, CRUD

**<feature>_status.dart:**
```dart
abstract class <Feature>Status {
  const <Feature>Status();
}

class <Feature>Initial extends <Feature>Status {
  const <Feature>Initial();
}

class <Feature>Loading extends <Feature>Status {
  const <Feature>Loading();
}

class <Feature>Success extends <Feature>Status {
  final List<<Entity>> items;
  const <Feature>Success(this.items);
}

class <Feature>Failure extends <Feature>Status {
  final String message;
  const <Feature>Failure(this.message);
}
```

**<feature>_controller.dart:**
```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import '<feature>_status.dart';

class <Feature>Controller extends Cubit<<Feature>Status> {
  final Get<Feature>Usecase _get<Feature>;

  <Feature>Controller(this._get<Feature>)
      : super(const <Feature>Initial());

  Future<void> load() async {
    emit(const <Feature>Loading());
    try {
      final items = await _get<Feature>.call();
      emit(<Feature>Success(items));
    } catch (e) {
      emit(<Feature>Failure(e.toString()));
    }
  }

  Future<void> refresh() => load();
}
```

**In screen:**
```dart
BlocBuilder<<Feature>Controller, <Feature>Status>(
  builder: (context, status) => switch (status) {
    <Feature>Loading()  => const Center(child: CircularProgressIndicator()),
    <Feature>Failure(:final message) => Center(child: Text(message)),
    <Feature>Success(:final items)   => _buildList(items),
    _                                => const SizedBox.shrink(),
  },
),
```

---

## Pattern 2: Stream Cubit (real-time)

Best for: live order updates, command status, POS terminal events

**<feature>_controller.dart:**
```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:rxdart/rxdart.dart';
import '<feature>_status.dart';

class <Feature>Controller extends Cubit<<Feature>Status> {
  final <Feature>Repository _repository;
  final _subject = BehaviorSubject<List<<Entity>>>();
  StreamSubscription? _sub;

  <Feature>Controller(this._repository) : super(const <Feature>Initial());

  Stream<List<<Entity>>> get stream => _subject.stream;

  Future<void> watch() async {
    emit(const <Feature>Loading());
    try {
      _sub = _repository
          .watchAll()
          .debounceTime(const Duration(milliseconds: 300))
          .listen(
            (items) {
              _subject.add(items);
              emit(<Feature>Success(items));
            },
            onError: (e) => emit(<Feature>Failure(e.toString())),
          );
    } catch (e) {
      emit(<Feature>Failure(e.toString()));
    }
  }

  @override
  Future<void> close() {
    _sub?.cancel();
    _subject.close();
    return super.close();
  }
}
```

---

## Pattern 3: Event Bus (cross-module)

Best for: payment completion → order refresh, command events, notifications

**Publishing an event (in sender controller):**
```dart
class PaymentController extends Cubit<PaymentStatus> {
  final Broker _broker;

  PaymentController(this._broker, ...) : super(const PaymentInitial());

  Future<void> confirmPayment(String commandId) async {
    // ... process payment
    _broker.dispatch(BrokerEvent(
      kind: 'payment.confirmed',
      data: {'commandId': commandId},
    ));
  }
}
```

**Consuming an event (in receiver controller):**
```dart
class CommandController extends Cubit<CommandStatus> {
  final Broker _broker;
  StreamSubscription? _paymentSub;

  CommandController(this._broker, ...) : super(const CommandInitial()) {
    _listenToPayments();
  }

  void _listenToPayments() {
    _paymentSub = _broker
        .on('payment.confirmed')
        .listen((event) => refresh());
  }

  @override
  Future<void> close() {
    _paymentSub?.cancel();
    return super.close();
  }
}
```

**Consumer widget (subscribes during module load):**
```dart
class PaymentConsumer {
  final Broker broker;
  StreamSubscription? _sub;

  PaymentConsumer({required this.broker});

  void start() {
    _sub = broker.on('order.created').listen((event) {
      // react to event
    });
  }

  void dispose() => _sub?.cancel();
}
```

---

## Status State Design Rules

1. Use `abstract class` (not sealed — Dart sealed requires same file)
2. Each status is a separate class extending the abstract base
3. States are **immutable** — use `const` constructors
4. Success state carries data as `final` fields
5. Failure state carries only `final String message`
6. Never embed logic in status classes — only data

## Controller Rules

1. Extends `Cubit<T>` — not `Bloc<Event, State>` (simpler for most cases)
2. Methods are `Future<void>` for async, `void` for sync
3. Always `emit(Loading())` before async calls
4. Always wrap async in try/catch → emit Failure
5. Cancel all subscriptions in `close()`
6. Never call `emit()` after `close()` — check `!isClosed` if needed

## Instructions

1. Read the existing controller in the module if one exists
2. Choose the right pattern based on whether the feature needs real-time data or cross-module events
3. Create `_status.dart` and `_controller.dart` files
4. Update the module's DI registration (see `/flutter-di` skill)
5. Update the screen to use `BlocBuilder` or `BlocListener` appropriately
6. Use switch expressions for status handling in UI (cleaner than if/else)
