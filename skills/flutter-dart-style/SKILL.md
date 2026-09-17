---
name: flutter-dart-style
description: Use when writing or reviewing any Dart or Flutter code for craft - naming, null safety, pattern matching, records, error handling idioms, dartdoc comments, logging, assertion style in tests, widget-level code quality (const, keys, dispose, BuildContext across an await), and analysis_options lint setup. Triggers on "clean this up", "is this idiomatic?", a code review of Dart, or a new file that sets a precedent others will copy.
---

You are writing or reviewing Dart for craft — the layer below architecture.
Where a project convention exists, it wins; this is what to do when there isn't
one, and what the analyzer will not tell you.

Related in this repo: **flutter-clean-arch** for layering, **flutter-testing**
for what to test, **flutter-concurrency** for isolates and streams,
**flutter-accessibility** for semantics, **flutter-theming** for colour and type.

Some topics below have a **canonical skill from the Dart team**
([dart-lang/skills](https://github.com/dart-lang/skills)) that goes deeper than
this file should. Where one exists it is named inline — install it and defer to
it. What stays here is the opinion: which convention this codebase picked, and
the traps the analyzer won't flag.

## The first rule: let the tools own what the tools own

Never spend a review comment on something `dart format` or `dart analyze` will
fix. Formatting, unused imports, missing `const` where the lint catches it —
all machine work.

```bash
dart format .        # not `flutter format`, removed in Flutter 3.38
dart fix --apply     # applies analyzer-suggested fixes
dart analyze
```

> Setting up and driving the analyzer is `dart-run-static-analysis`'s job.

Review *semantics*: a name that misleads, an error swallowed, a shape that
won't survive the next change.

## Naming

| Thing | Case | Note |
|---|---|---|
| Classes, enums, typedefs, extensions | `UpperCamelCase` | |
| Members, variables, parameters, constants | `lowerCamelCase` | constants too — not `SCREAMING_CAPS` |
| Files, directories, import prefixes | `lowercase_with_underscores` | |
| Private | leading `_` | |

Beyond case:

- **Put the most descriptive noun last.** `pageCount`, not `numPages`.
  `inputUserName`, not `userNameInput`. The last word is what a reader scanning
  a column of names actually sees.
- **Don't abbreviate** unless the abbreviation is more common than the word
  (`id`, `http`, `db` are fine; `usr`, `btn`, `cfg` are not). Abbreviation drift
  inside one file — `user` in one method and `usr` in the next — is worth a
  review comment even though it changes no behaviour.
- **Non-boolean property → noun phrase** (`elements`, `remainingBudget`).
  **Function that does something → verb phrase** (`sortItems()`, `sendInvite()`).
  **Boolean → non-imperative predicate** that reads as an assertion:
  `isEmpty`, `hasElements`, `canClose` — never `close` or `getIsEmpty`.
- **Don't prefix getters with `get`.** In Dart the access syntax already says
  it: `user.name`, not `user.getName()`.
- **`toX()` returns a new object; `asX()` returns a view or wrapper** over the
  same data. Picking the wrong one tells the caller the wrong thing about cost
  and about whether mutations alias.
- **Don't repeat the parameter's type in the name.** `addPerson(Person p)`, not
  `addPersonWithPerson`. The signature is right there.

## Null safety

Soundly null-safe code is the default; the interesting part is what to do at
the boundary.

- **`!` is a claim you are making to the compiler.** Every one is a potential
  runtime crash. Use it only where the invariant is genuinely local and
  obvious, and prefer restructuring so the compiler can see it:

  ```dart
  // avoid — the ! is load-bearing and far from the check
  if (user != null) { ... }
  doSomething(user!.name);

  // prefer — promote once, into a non-nullable local
  final u = user;
  if (u != null) doSomething(u.name);
  ```

  A field can't be promoted (another isolate or subclass could change it),
  which is why the local is necessary.

- Reach for the null-aware operators — `?.`, `??`, `??=`, `...?` — before an
  explicit `if (x == null)`.
- **Don't initialise to `null` explicitly.** `int? count;` is already null.
- Prefer `late final` over a nullable field you know gets set once in `initState`.
  It fails loudly at the wrong access instead of spreading `?` through callers.
- At an API boundary you don't control (JSON, platform channels), validate once
  on the way in and hand the rest of the codebase non-nullable types. Don't let
  `Map<String, dynamic>` nullability leak past the model layer.

## Pattern matching over a sealed status

> The general feature is `dart-use-pattern-matching`'s subject. What follows is
> only how it pairs with the sealed status classes this codebase uses.

- **Prefer `switch` expressions** over statements when producing a value. They
  are exhaustive over a sealed hierarchy, so adding a variant becomes a compile
  error instead of a silently missed branch — which is the whole reason to seal
  a status class in the first place (see **flutter-state**).

  ```dart
  final label = switch (status) {
    LoadingStatus()      => 'Loading…',
    SuccessStatus(:final items) => '${items.length} found',
    ErrorStatus(:final message) => message,
  };
  ```

  `:final items` destructures in the pattern — no cast, no second lookup.

- **Records for a genuinely anonymous pair**, when a named class would be
  ceremony and the tuple never leaves the file:

  ```dart
  (int width, int height) measure() => (800, 600);
  final (w, h) = measure();
  ```

  Once a record crosses a layer boundary or gains a third field, make it a
  class. Records have no name, so they can't carry meaning or invariants.

- Avoid `default:` in a switch over a sealed type. It defeats exhaustiveness
  and turns the next added variant into a runtime bug.

## Error handling

- **`catch` without an `on` clause catches everything**, including
  programming errors you wanted to crash on. Catch the type you can act on.
- **Never swallow.** An empty `catch {}`, or one that only returns `null`,
  destroys the stack trace at the exact moment you needed it.
- **`rethrow`, not `throw e`** — `throw e` restarts the stack trace at the
  catch site and you lose the origin.
- Preserve the stack trace when you cross a layer: `catch (e, s)` and pass `s`
  along. A domain exception with no trace is a bug report you can't act on.
- Throw objects implementing `Exception` (recoverable, expected) or `Error`
  (a bug — don't catch these). Never throw a raw `String`.

## Members and constructors

- **Prefer `final` fields.** Immutability removes a whole class of question
  from every read.
- **Don't wrap a field in a trivial getter/setter.** Dart lets you turn a field
  into a getter later without changing call sites — that's the point of
  uniform access. Add the getter when it does something.
- **Use `=>` for members that are a single expression**, and a block body when
  it isn't. Don't chain `=>` across three lines to avoid braces.
- **Initialising formals**: `Foo(this.bar)` over an assignment in the body.
- **`const` constructors wherever the class allows it** — this is what lets
  callers build the widget once and skip rebuilds.
- If you override `==`, **override `hashCode`**, and don't define `==` on a
  mutable class — a mutated key silently disappears from any `Set` or `Map`
  holding it. For value objects, `Equatable` or a record is usually less risk
  than hand-writing both.

## Type annotations

Annotate where it's a contract or where inference can't see it; stay quiet
where the compiler already knows:

- **Do annotate**: return types, parameter types, fields and top-level
  variables without an obvious initialiser.
- **Don't annotate**: locals with an initialiser, and lambda parameters whose
  type comes from context.

```dart
// the types add nothing here
ListView.builder(
  itemBuilder: (context, index) => ItemCard(items[index]),
);

// but here the annotation is the only thing documenting the shape
final Map<String, List<int>> byCategory = {};
```

An empty collection literal with no annotation infers `dynamic` and quietly
disables type checking for everything downstream.

## Widget-level quality

- **A private widget class beats a `Widget _buildHeader()` helper.** It isn't
  style: a helper method has no element of its own, so it rebuilds whenever the
  parent does and can never be `const`. A `class _Header extends StatelessWidget`
  gets its own element and can be skipped entirely.
- **Break up a long `build()`** the same way — into widget classes, not more
  helpers.
- **No expensive work in `build()`.** It runs on every frame that touches this
  subtree. Network calls, sorting a large list, parsing — all belong above it.
  Heavy pure computation goes to `compute()` / `Isolate.run()`; see
  **flutter-concurrency**.
- **Dispose what you create**: controllers, `StreamSubscription`s, `Ticker`s,
  `FocusNode`s. If a field has a `dispose()`, your `State` needs one.
- **`BuildContext` across an `await` is a bug waiting to happen.** Guard it:

  ```dart
  await repository.save(item);
  if (!context.mounted) return;
  Navigator.of(context).pop();
  ```

  `use_build_context_synchronously` catches the common shape, not all of them.
- **Stable `Key`s on dynamic lists.** Without one, Flutter matches children by
  position, so an insert or reorder keeps the wrong state attached to the wrong
  item — the classic "the checkbox moved to another row" bug.
- **`ListView.builder`, never a `ListView(children: [...])`** for a list whose
  length you don't control. The non-builder form materialises every child.

## Documentation

> Formatting rules — summary sentence, third-person verbs, `[brackets]`, where
> the comment sits relative to annotations — are `dart-write-documentation`'s
> subject, and it is the authority. Two judgement calls it can't make for you:

- **Document what a caller can't see**, not what the signature already shows.
  Nullability, what it throws and when, side effects, cost. A doc comment that
  re-lists the parameters has added nothing.

  ```dart
  /// Fetches the profile for [userId].
  ///
  /// Returns `null` when no such user exists. Throws [NetworkException] if the
  /// request fails for connectivity reasons — callers that can retry should
  /// catch it; other failures are programming errors and propagate.
  Future<User?> fetchUser(String userId) async { ... }
  ```

- **Comment why, not what.** `// increment i` is noise. `// the API is 1-based`
  is the reason the next person doesn't "fix" it back into a bug.

## Logging

`print()` is for scratch debugging and the `avoid_print` lint exists for a
reason. Use `dart:developer`, which DevTools understands:

```dart
import 'dart:developer' as developer;

try {
  await repository.sync();
} catch (e, s) {
  developer.log(
    'Sync failed',
    name: 'myapp.sync',   // shows as a filterable channel in DevTools
    level: 1000,          // 1000 = SEVERE, per package:logging levels
    error: e,
    stackTrace: s,
  );
}
```

Never log a credential, token, or personally identifying field. In an app that
handles user data, assume every log line reaches a crash reporter.

## Assertion style in tests

Use `package:checks` for unit and service assertions — better failure messages
and type-safe chaining than bare `expect`. `dart-migrate-to-checks-package`
covers the migration and the full matcher mapping; the two things worth fixing
as convention here:

```dart
check(user)
  ..has((u) => u.name,  'name').equals('Ana')
  ..has((u) => u.email, 'email').contains('@');
```

- **Prefer `.has((u) => u.field, 'field')` over comparing whole objects.** It
  makes the failure name *which* field differed instead of dumping two objects
  side by side and leaving you to diff them.
- **Widget tests keep `expect`** with `findsOneWidget` / `findsNothing` — those
  are `flutter_test` matchers with no `checks` equivalent. Both styles in one
  file is fine; they solve different problems.

See **flutter-testing** for structure, mocking and what deserves a test at all.

## Lint setup

`dart-run-static-analysis` owns running the analyzer and `dart fix`. The
convention worth stating: **promote a lint to `error` when violating it ships a
bug; leave it a warning when it is taste** — and always exclude generated
files, because noise you can't fix trains people to ignore the analyzer.

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  errors:
    use_build_context_synchronously: error   # real crashes, not style
    unawaited_futures: error
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
```

Add rules deliberately. A rule nobody agreed to becomes a rule everybody
suppresses.

## Review checklist

1. Would `dart format` / `dart analyze` have caught it? Then don't raise it.
2. Does any name mislead, or drift from the name used three lines up?
3. Is there a `!` whose invariant isn't visible within a few lines?
4. Is any error swallowed, or rethrown as `throw e` losing the trace?
5. Is a `switch` over a sealed type using `default:`?
6. Does a `State` create something with a `dispose()` and not dispose it?
7. Is `context` used after an `await` without a `mounted` guard?
8. Is a `Widget _buildX()` helper doing what a private widget class should?
9. Does a new public API have a doc comment saying what a caller can't see?
10. Is anything logged that shouldn't leave the device?
