---
name: flutter-web
description: Use when working on a Flutter web build - code touching dart:js_interop, package:web or dart:ui_web; tests that will not compile on the VM runner; a blank white page after a build; or QA of a running web app in a browser. Triggers on "works on mobile but not on web", a suite that breaks after a web import is added, or a screen that looks frozen in the browser.
---

You are working on a Flutter **web** build. Everything here is a lesson that
cost real hours — each one looks like an application bug and is not.

## Browser QA only counts in a visible tab

Flutter web drives its **entire** frame pipeline from `requestAnimationFrame`,
and Chrome throttles rAF to near zero in a tab that is not the active one. In a
background tab the app still runs — events are processed, network calls fire —
but it barely paints. So route transitions appear frozen for tens of seconds,
and click coordinates read off a screenshot land nowhere, because the painted
frame no longer matches the real hit-test layout.

That looks exactly like two bugs that do not exist: a slow splash screen and
broken Tab traversal.

Before blaming the app:

```js
document.visibilityState   // must be "visible"
```

If it reports `hidden`, bring the browser window forward and make the tab
active. Two things stay reliable either way, and are the better evidence:

- `performance.getEntriesByType('resource')` for real timings — in one case it
  showed auth resolving at 1.4s while the screen looked stuck for 20s.
- `document.activeElement`, and the hidden `<input>` Flutter creates for the
  focused text field, which proves where focus actually is.

Each CDP screenshot forces one frame, so consecutive screenshots step a stalled
transition forward — useful, and itself a hint that you are in a throttled tab.

## Browser-layer code needs the Chrome test runner

Anything touching `dart:js_interop` or `package:web` — `getUserMedia`,
`MediaRecorder`, an `<audio>` element — **cannot even compile** under the
default VM test runner.

`@TestOn('browser')` is what makes that survivable: `flutter test` *skips* an
annotated file instead of compiling it. The browser layer is untested by
default, not untestable:

```bash
flutter test                      # VM: controllers, repositories, pure widgets
flutter test --platform chrome    # the browser layer
```

**Both commands run over `test/`.** Do not move browser tests to a directory of
their own — the web runner builds its entrypoint with `test/` as the root, so a
path outside it becomes a `../` that does not resolve and nothing compiles at
all. The annotation is the separator, not the directory.

The trap that makes this dangerous: `flutter test` alone reports **green** over
a browser test that never compiled. Run both before pushing, and run both in
CI as two separate steps.

`--platform chrome` is also flaky or unavailable on some machines (a file with
`@TestOn('browser')` hanging at *loading* until the test timeout is the usual
symptom). Where that happens, CI is the only place the browser layer runs — so
prefer **pulling the logic out of the browser-bound file into a plain Dart
one** and testing that on the VM runner.

### Make it testable: contract first, js_interop behind it

Declare the **minimum contract as an abstract class**, keep the `js_interop`
implementation behind it, and inject it so a widget test can pass a fake:

```dart
abstract class AudioRecorder {
  Future<void> start();
  Future<Uint8List?> stop();
  void dispose();
}

// browser_audio_recorder.dart — the only file importing package:web
class BrowserAudioRecorder implements AudioRecorder { ... }
```

### Every exit path must release a browser resource

When a widget holds a browser resource, release it on **every** way out —
not only the happy one:

- `dispose()` unconditionally, rather than only when a `_recording` flag flipped.
- A release when the screen turns out to be unmounted after an `await`.
- A guard so a second tap during the first `await` is a no-op.

A real leak was exactly the gap between those: permission was still pending, so
the flag had not flipped, so `dispose()` released nothing — and an orphaned
ticker wrote to a closed `StreamController` every 200 ms for the life of the tab.

## Blank white page after changing a path dependency

Path dependencies (a `design_system/` or `calendar/` package in the same repo)
are not always picked up by the incremental web build. It emits a bundle that
compiles, loads every asset, throws no JS error — and renders a blank white page
with an empty `<flutter-view>`. Hours disappear looking for a bug in code that
is fine.

```bash
rm -rf build/web && flutter build web --release --dart-define=BASE_URL=...
```

Do that **before** concluding anything from a blank screen.

## Watch the import chain, not just the import

A web-only import poisons every file that reaches it transitively, and the
compiler names the file that failed — not the file at fault.

The classic shape: a `app_routes.dart` holding both the route-name constants
*and* the route table. The table imports every screen, one of which imports
`dart:ui_web`. A sign-out use case importing that file for **one string
constant** then makes every VM test that touches it fail to compile, taking the
whole suite with it.

The fix is to split the constants into a file that imports nothing:

```dart
// app_route_names.dart — imports nothing
class AppRouteNames {
  static const login = '/login';
  static const home  = '/home';
}

// app_routes.dart — re-exports the names, and owns the table
class AppRoutes extends AppRouteNames { ... }
```

Screens and the root widget may import the table. A use case, a repository or
an interceptor may not — they take the names file.

So when the VM suite fails to **load** a file rather than fail an assertion,
read which file it named, then walk the import chain back to the web-only leaf.

## Checklist before shipping a web change

```bash
flutter analyze
flutter test                              # VM
flutter test --platform chrome            # browser layer
rm -rf build/web
flutter build web --release --dart-define=BASE_URL=<api-url>
```

## Common mistakes

- Debugging a "frozen" screen in a background tab. Check `document.visibilityState` first.
- Trusting a green `flutter test` over code that only compiles on the browser runner.
- Moving browser tests out of `test/` — `--platform chrome` cannot resolve a path outside it.
- Calling `js_interop` straight from a widget instead of behind an injectable interface.
- Releasing a browser resource only on the happy path. Unmount during an `await` is a real exit.
- Concluding "blank page = my bug" without a clean `build/web` rebuild first.
- Importing the route *table* from a use case or interceptor just to read one route name.
- Forgetting `--dart-define=BASE_URL`, so the compile-time constant is `""` and every call fails. See **flutter-run**.
- Leaving semantics off — Flutter web disables them by default. See **flutter-accessibility**.
