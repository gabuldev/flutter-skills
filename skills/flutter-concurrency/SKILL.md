---
name: flutter-concurrency
description: Use when dealing with asynchrony in Flutter - async/await inside a Cubit, Streams and RxDart, Isolates for heavy work, cancelling an in-flight operation, or choosing between FutureBuilder and BlocBuilder. Triggers on UI freezes, jank, race conditions, or "setState after dispose".
---

You are implementing concurrency for a Flutter feature. Choose the pattern based on the skill argument, if one was given, or on the context of the operation.

## Decision Framework

Pick the right tool for the workload:

| Workload | Pattern | Examples |
|---|---|---|
| I/O (network, storage) | `async/await` | Dio API calls, SharedPreferences, Hive reads |
| CPU < 16 ms | `async/await` | Small list transforms, formatting |
| One-time heavy CPU | `Isolate.run()` | PDF receipt generation, large JSON parsing |
| Continuous background | `Isolate.spawn()` with ports | Payment status polling, data sync |

**Rule of thumb**: if it does NOT block the main isolate for more than one frame (~16 ms), use `async/await`. Otherwise, offload to an isolate.

---

## Pattern 1: Async in Cubits (default)

Best for: API calls, storage reads, form submissions.

Controllers follow the Loading → Success/Failure emission pattern:

```dart
class OrderController extends Cubit<OrderStatus> {
  final OrderRepository _repository;

  OrderController(this._repository) : super(const OrderInitial());

  Future<void> loadOrders() async {
    emit(const OrderLoading());
    try {
      final orders = await _repository.getOrders();
      emit(OrderSuccess(orders));
    } on DioException catch (e) {
      emit(OrderFailure(e.message ?? 'Erro ao carregar pedidos'));
    } catch (e) {
      emit(OrderFailure(e.toString()));
    }
  }
}
```

**Rules:**
- Always emit `Loading` before the async call
- Catch `DioException` separately for HTTP errors
- Never emit after the Cubit is closed (see Pitfalls section)

---

## Pattern 2: Stream Management with RxDart

Best for: real-time data, debounced search, reactive pipelines.

```dart
class SearchController extends Cubit<SearchStatus> {
  final SearchRepository _repository;
  final _querySubject = BehaviorSubject<String>.seeded('');
  StreamSubscription<String>? _querySub;

  SearchController(this._repository) : super(const SearchInitial()) {
    _querySub = _querySubject
        .debounceTime(const Duration(milliseconds: 400))
        .where((q) => q.length >= 2)
        .distinct()
        .listen(_performSearch);
  }

  void onQueryChanged(String query) => _querySubject.add(query);

  Future<void> _performSearch(String query) async {
    emit(const SearchLoading());
    try {
      final results = await _repository.search(query);
      emit(SearchSuccess(results));
    } catch (e) {
      emit(SearchFailure(e.toString()));
    }
  }

  @override
  Future<void> close() {
    _querySub?.cancel();
    _querySubject.close();
    return super.close();
  }
}
```

**Rules:**
- Always dispose `BehaviorSubject` and cancel `StreamSubscription` in `close()`
- Use `debounceTime` for user input to avoid excessive API calls
- Use `distinct()` to skip duplicate emissions
- Use `BehaviorSubject` (not `PublishSubject`) when you need the latest value on subscription

---

## Pattern 3: Isolate.run() for One-Time Heavy Work

Best for: PDF receipt generation, parsing large JSON payloads, image processing.

### PDF Receipt Generation

```dart
class ReceiptController extends Cubit<ReceiptStatus> {
  ReceiptController() : super(const ReceiptInitial());

  Future<void> generateReceipt(Order order) async {
    emit(const ReceiptLoading());
    try {
      // Offload PDF generation to a background isolate
      final pdfBytes = await Isolate.run(() => _buildReceiptPdf(order));
      emit(ReceiptSuccess(pdfBytes));
    } catch (e) {
      emit(ReceiptFailure('Erro ao gerar comprovante: $e'));
    }
  }
}

// Top-level or static function (required for Isolate.run)
Uint8List _buildReceiptPdf(Order order) {
  final pdf = pw.Document();
  pdf.addPage(
    pw.Page(
      pageFormat: PdfPageFormat(80 * PdfPageFormat.mm, double.infinity),
      build: (context) => pw.Column(
        crossAxisAlignment: pw.CrossAxisAlignment.start,
        children: [
          pw.Text('Pedido #${order.id}', style: pw.TextStyle(fontSize: 14, fontWeight: pw.FontWeight.bold)),
          pw.SizedBox(height: 4),
          ...order.items.map((item) => pw.Text('${item.quantity}x ${item.name} - R\$ ${item.total.toStringAsFixed(2)}')),
          pw.Divider(),
          pw.Text('Total: R\$ ${order.total.toStringAsFixed(2)}', style: pw.TextStyle(fontWeight: pw.FontWeight.bold)),
        ],
      ),
    ),
  );
  return pdf.save();
}
```

**Rules:**
- The function passed to `Isolate.run()` must be **top-level** or **static** (no closures over instance state)
- All arguments must be **sendable** across isolate boundaries (primitives, Lists, Maps, typed data classes — no Cubits, no Dio instances)
- `Isolate.run()` handles spawning and killing the isolate automatically

### Large Data Processing

```dart
Future<void> processReport(List<Map<String, dynamic>> rawData) async {
  emit(const ReportLoading());
  try {
    final processed = await Isolate.run(() => _aggregateReportData(rawData));
    emit(ReportSuccess(processed));
  } catch (e) {
    emit(ReportFailure('Erro ao processar relatório: $e'));
  }
}

// Top-level function
ReportData _aggregateReportData(List<Map<String, dynamic>> rawData) {
  // Heavy aggregation, grouping, sorting — runs off the main isolate
  final grouped = <String, double>{};
  for (final entry in rawData) {
    final category = entry['category'] as String;
    final value = (entry['value'] as num).toDouble();
    grouped[category] = (grouped[category] ?? 0) + value;
  }
  return ReportData(grouped);
}
```

---

## Pattern 4: Isolate.spawn() for Continuous Background Work

Best for: payment status polling, background data sync.

```dart
class PaymentPollingService {
  Isolate? _isolate;
  ReceivePort? _receivePort;
  StreamSubscription? _portSub;

  Future<void> startPolling(String transactionId) async {
    _receivePort = ReceivePort();

    _isolate = await Isolate.spawn(
      _pollPaymentStatus,
      _PollingArgs(transactionId, _receivePort!.sendPort),
    );

    _portSub = _receivePort!.listen((message) {
      if (message is PaymentStatusUpdate) {
        // Forward to EventBus/Broker or Cubit
        Broker.instance.fire(PaymentStatusEvent(message));
      }
    });
  }

  void stopPolling() {
    _portSub?.cancel();
    _receivePort?.close();
    _isolate?.kill(priority: Isolate.immediate);
    _isolate = null;
  }
}

// Top-level function
void _pollPaymentStatus(_PollingArgs args) async {
  while (true) {
    await Future.delayed(const Duration(seconds: 3));
    // Simulate checking payment status (no Dio here — use dart:io HttpClient or pass data via ports)
    final status = await _checkStatus(args.transactionId);
    args.sendPort.send(PaymentStatusUpdate(status));
    if (status == 'approved' || status == 'declined') break;
  }
}

class _PollingArgs {
  final String transactionId;
  final SendPort sendPort;
  _PollingArgs(this.transactionId, this.sendPort);
}
```

**Rules:**
- Always provide a way to **stop** the isolate (`kill()` + close the port)
- The spawned function cannot use objects from the main isolate (no Dio, no Cubit, no repos)
- Communicate via `SendPort` / `ReceivePort` only with sendable types
- Clean up in the service's `dispose()` or controller's `close()`

---

## FutureBuilder vs BlocBuilder

With the **Cubit pattern**, prefer `BlocBuilder` over `FutureBuilder`.

| | `FutureBuilder` | `BlocBuilder` |
|---|---|---|
| Use when | Quick prototype, one-off Future with no state | The default for anything with state |
| Re-triggers on rebuild | Yes (unless Future is cached) | No (reacts to state only) |
| Testable | Hard | Easy (mock Cubit, emit states) |
| Error handling | In builder `snapshot.hasError` | In state class (`FailureStatus`) |

```dart
// PREFERRED
BlocBuilder<OrderController, OrderStatus>(
  builder: (context, state) {
    if (state is OrderLoading) return const CircularProgressIndicator();
    if (state is OrderSuccess) return OrderListView(orders: state.orders);
    if (state is OrderFailure) return ErrorWidget(message: state.message);
    return const SizedBox.shrink();
  },
)

// AVOID unless truly one-off and outside of a Cubit-managed feature
FutureBuilder<List<Order>>(
  future: _future, // must be cached in initState, never inline
  builder: (context, snapshot) { ... },
)
```

---

## Cancellation

### HTTP Requests with Dio CancelToken

```dart
class ProductController extends Cubit<ProductStatus> {
  final ProductRepository _repository;
  CancelToken? _cancelToken;

  ProductController(this._repository) : super(const ProductInitial());

  Future<void> loadProducts() async {
    _cancelToken?.cancel('New request started');
    _cancelToken = CancelToken();

    emit(const ProductLoading());
    try {
      final products = await _repository.getProducts(cancelToken: _cancelToken!);
      emit(ProductSuccess(products));
    } on DioException catch (e) {
      if (e.type == DioExceptionType.cancel) return; // silently ignore
      emit(ProductFailure(e.message ?? 'Erro'));
    }
  }

  @override
  Future<void> close() {
    _cancelToken?.cancel('Controller closed');
    return super.close();
  }
}
```

### Stream Subscriptions

```dart
@override
Future<void> close() {
  _subscription?.cancel();  // StreamSubscription
  _subject.close();         // BehaviorSubject
  _cancelToken?.cancel();   // Dio CancelToken
  return super.close();
}
```

**Rule:** Every subscription, subject, and cancel token created in a controller **must** be cleaned up in `close()`.

---

## Common Pitfalls

### 1. Emit after close

```dart
// BAD — can throw StateError if Cubit was closed during the await
Future<void> load() async {
  emit(const Loading());
  final data = await _repo.fetch();
  emit(Success(data)); // 💥 if navigated away
}

// GOOD — guard before emitting
Future<void> load() async {
  emit(const Loading());
  final data = await _repo.fetch();
  if (isClosed) return;
  emit(Success(data));
}
```

### 2. Uncancelled subscriptions

```dart
// BAD — leaks the subscription
class MyController extends Cubit<MyStatus> {
  MyController(Stream<int> stream) : super(const MyInitial()) {
    stream.listen((val) => emit(MyUpdated(val))); // never cancelled
  }
}

// GOOD
class MyController extends Cubit<MyStatus> {
  late final StreamSubscription<int> _sub;
  MyController(Stream<int> stream) : super(const MyInitial()) {
    _sub = stream.listen((val) => emit(MyUpdated(val)));
  }
  @override
  Future<void> close() { _sub.cancel(); return super.close(); }
}
```

### 3. Blocking the main isolate

```dart
// BAD — 200ms of JSON parsing freezes the UI
final data = jsonDecode(hugeJsonString) as List;

// GOOD — offload to isolate
final data = await Isolate.run(() => jsonDecode(hugeJsonString) as List);
```

### 4. Passing non-sendable objects to isolates

```dart
// BAD — Dio is not sendable
await Isolate.run(() => dio.get('/endpoint')); // 💥 runtime error

// GOOD — fetch on main, process on isolate
final response = await dio.get('/endpoint');
final processed = await Isolate.run(() => _heavyParse(response.data));
```

### 5. FutureBuilder with inline Future

```dart
// BAD — re-creates the Future on every build
Widget build(BuildContext context) {
  return FutureBuilder(
    future: repository.load(), // called every rebuild!
    builder: (ctx, snap) => ...,
  );
}

// GOOD — cache the Future
late final _future = repository.load();
Widget build(BuildContext context) {
  return FutureBuilder(future: _future, builder: (ctx, snap) => ...);
}
```
