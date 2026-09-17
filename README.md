# flutter-skills

Claude Code skills for Flutter development — extracted from production apps and
generalised so you can drop them into your own project.

A [skill](https://docs.claude.com/en/docs/claude-code/skills) is a folder with a
`SKILL.md` that Claude loads **on demand**, when the task at hand matches its
description. These fifteen cover the parts of a Flutter codebase where the
answer is a convention rather than a fact the model can look up: which layer a
file belongs in, how a Cubit should emit, where a browser-only import will blow
up your test suite.

> **Using the official skills too?** The Flutter and Dart teams publish their
> own — see [Related official skills](#related-official-skills). They cover
> different ground and are designed to sit alongside these.

## Install

Copy the skills you want into your project's `.claude/skills/` (per project) or
`~/.claude/skills/` (every project):

```bash
git clone https://github.com/gabuldev/flutter-skills.git
cp -r flutter-skills/skills/* your-project/.claude/skills/
```

Take only what fits — each skill is self-contained, and a skill for a pattern
you don't use is just noise in the model's decision to load one.

Verify they registered with `/skills` inside Claude Code. Claude picks a skill
up on its own from the description; you can also invoke one directly by name.

## The skills

### Code craft

| Skill | What it covers |
|---|---|
| [`flutter-dart-style`](skills/flutter-dart-style/SKILL.md) | Naming, null safety at boundaries, error-handling idioms, `const`/keys/`dispose`, `BuildContext` across an `await`, logging, and what to leave to the analyzer |

### Architecture & structure

| Skill | What it covers |
|---|---|
| [`flutter-clean-arch`](skills/flutter-clean-arch/SKILL.md) | Domain/data/presentation layers — where each file goes, dependency direction, and an `audit` mode that finds inverted ones |
| [`flutter-new-module`](skills/flutter-new-module/SKILL.md) | Scaffolds a whole feature: module + DI, Cubit controller, sealed status, screen, route |
| [`flutter-di`](skills/flutter-di/SKILL.md) | Dependency injection with `flutter_injections` — global vs module scope, singleton vs factory |
| [`flutter-monorepo`](skills/flutter-monorepo/SKILL.md) | Melos workspaces — adding packages, bootstrap, per-app environment |

### State & async

| Skill | What it covers |
|---|---|
| [`flutter-state`](skills/flutter-state/SKILL.md) | Cubit + sealed status classes, `BlocBuilder` with pattern matching, Stream-fed Cubits, cross-module event bus |
| [`flutter-concurrency`](skills/flutter-concurrency/SKILL.md) | async/await vs Isolates, cancelling in-flight work, and the 16 ms rule for when to offload |
| [`flutter-caching`](skills/flutter-caching/SKILL.md) | Offline-first repositories, Dio cache interceptor, write sync queues, secure storage vs preferences |

### UI

| Skill | What it covers |
|---|---|
| [`flutter-theming`](skills/flutter-theming/SKILL.md) | Material 3 `ColorScheme`/`ThemeData`, component themes, replacing hardcoded colours, consistency audit |
| [`flutter-design-system`](skills/flutter-design-system/SKILL.md) | A shared tokens package — colour, type scale, spacing, radius — and components built from it |
| [`flutter-forms`](skills/flutter-forms/SKILL.md) | Validation, Cubit wiring, multi-step forms, and pt-BR masks (CPF, CNPJ, phone, CEP, currency) |
| [`flutter-accessibility`](skills/flutter-accessibility/SKILL.md) | `Semantics`, screen-reader labels, contrast, tap targets, font scaling, and enabling semantics on web |

### Build, test & platform

| Skill | What it covers |
|---|---|
| [`flutter-testing`](skills/flutter-testing/SKILL.md) | Unit, widget and `bloc_test` patterns with mocktail, plus Maestro E2E flows |
| [`flutter-run`](skills/flutter-run/SKILL.md) | Flavors, `--dart-define`/`--dart-define-from-file`, device selection, and a task runner that keeps them in sync |
| [`flutter-web`](skills/flutter-web/SKILL.md) | The web-only traps: rAF throttling in background tabs, `@TestOn('browser')`, `js_interop` behind an interface, blank-page rebuilds |

## What these assume

The skills were written against a particular stack, and the code samples show
it. The **shapes** transfer; the package names may not.

- **State** — `flutter_bloc` (Cubit, not Bloc) + `rxdart`
- **DI** — [`flutter_injections`](https://pub.dev/packages/flutter_injections)
- **HTTP** — `dio`
- **Monorepo** — `melos`, with a shared `core` package and a `design_system` package
- **Models** — hand-written `fromJson`/`toEntity`, no `freezed`/`json_serializable`
- **Tests** — `mocktail`, `bloc_test`, `maestro` for E2E

If you use `riverpod`, `get_it` or `freezed`, the architecture and testing
skills still apply — edit the samples in the ones that don't.

`flutter-forms` is deliberately **pt-BR specific** in its masks and validators
(CPF, CNPJ, CEP). That part is the point of it; the form/Cubit wiring around it
is generic.

## Related official skills

The Flutter and Dart teams publish their own skill collections, both
BSD-3-Clause. **Install them too** — they cover different ground, and this repo
defers to them where they are authoritative.

| Repo | What it is |
|---|---|
| [flutter/agent-plugins](https://github.com/flutter/agent-plugins) | 10 skills from the Flutter team: widget/integration tests, widget previews, layered architecture, responsive layout, fixing layout overflows, JSON serialization, `go_router`, localization, `package:http` |
| [dart-lang/skills](https://github.com/dart-lang/skills) | 14 skills from the Dart team: unit tests, coverage, mockito mocks, `package:checks` migration, static analysis, pattern matching, dartdoc, FFI/ffigen, CLI apps, dependency conflicts |

```bash
npx skills add dart-lang/skills --skill '*' --agent universal --yes
```

**How they differ from this repo.** The official skills are task recipes on the
standard stack — "add a widget test", "set up `go_router`", "run static
analysis". They assume no house style and answer the same for everyone.

These skills are the opposite: an opinionated set of **house conventions** for a
specific stack (Cubit, `flutter_injections`, Dio, Melos), plus the traps that
cost someone here an afternoon. They answer "how do *we* do it", not "how is it
done".

Where they overlap, prefer the official one for mechanics and this one for the
convention. `flutter-dart-style` names the relevant `dart-*` skill inline
wherever the Dart team owns a topic more deeply.

## Contributing

Issues and PRs welcome. A good skill here is **specific** — it encodes a
decision that a competent Flutter dev would otherwise have to make from scratch
every time, and ideally a mistake that already cost someone an afternoon. A
skill that restates the Flutter docs earns nothing.

Two things to keep in mind when adding one:

- The `description` is the whole loading mechanism. Write it as the situations
  it should fire in, not as a topic label.
- No project-specific names, URLs or credentials. These run in other people's
  repos.

## License

MIT — see [LICENSE](LICENSE).
