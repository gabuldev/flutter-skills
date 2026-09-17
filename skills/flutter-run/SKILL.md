---
name: flutter-run
description: Use when running, building, installing or debugging a Flutter app — picking a device or emulator, working with build flavors, passing --dart-define values, loading secrets into a build, or when the app won't start, opens blank, or can't reach the dev API. Covers the task-runner wrapper (Makefile/melos script) that keeps those flags consistent.
---

You are running or building a Flutter app. The core idea of this skill: **a bare
`flutter run` is almost never the right command in a real project.** Flavors,
`--dart-define`s and the Flutter version all have to line up, and remembering
them by hand is where the lost afternoons come from.

## Put a task runner in front of `flutter`

Make the repo's `Makefile` (or melos script, or `just` file) the source of truth
for how each app is launched, and run *that*. It is the only way the flags stay
consistent between your machine, a teammate's, and CI.

A shape that works for a multi-app monorepo:

```bash
make setup                                  # pin Flutter (FVM) + bootstrap deps
make secrets ENV=dev PROJECT=<app>          # fetch secrets → .secrets.<app>.<env>.json (gitignored)

make dev PROJECT=<app>                      # web (Chrome), ENV=dev
make dev PROJECT=<app> PLATFORM=android     # Android, flavor applied automatically
make build PROJECT=<app> ENV=prod           # release build
```

Pin the SDK with **FVM** (`fvm flutter ...`) and commit `.fvmrc`. "Works on my
machine" in Flutter is usually two different SDK minors.

## Configuration: `--dart-define`, and the file form

App config should reach the binary through `--dart-define`, read back with
`String.fromEnvironment`:

```dart
class AppEnv {
  static const baseURL = String.fromEnvironment('BASE_URL');
  static const appName = String.fromEnvironment('APP_NAME');
  static const apiKey  = String.fromEnvironment('X_API_KEY');
}
```

`String.fromEnvironment` is **const** and resolved at compile time. Two
consequences that bite:

- A missing define is not an error — it is the empty string. An app that "can't
  reach the backend" is very often an empty `BASE_URL`, not a network problem.
- Changing a define requires a rebuild, not a hot reload.

Pass many values at once with `--dart-define-from-file` instead of a long chain
of flags:

```bash
flutter run --dart-define-from-file=.secrets.myapp.dev.json
```

Keep `.secrets.*.json` **gitignored** and generate it from your secret manager
(Doppler, 1Password, Vault, `gcloud secrets`) in a `make secrets` target. Have
the task runner fall back to a couple of safe public defines when the file is
absent, and **print a loud warning** — a silent fallback produces an app that
launches and then misbehaves in ways that look like application bugs.

## Android flavors

If the app declares product flavors, a `flutter run` without `--flavor` **fails
outright** — `flutter run` cannot pick one for you. Have the task runner supply
a sensible default and let it be overridden:

```bash
make dev PROJECT=<app> PLATFORM=android                    # default flavor
make dev PROJECT=<app> PLATFORM=android FLAVOR=<other>     # pin one
```

Where a flavor encodes more than one axis (device model × payment acquirer,
free × paid, region), name it `<axisA><AxisB>` and **derive the matching
`--dart-define` from the flavor name** in the task runner rather than asking the
caller to keep two flags in sync. Apps with no flavors must not be passed
`--flavor` at all.

## Device selection

- Web → `-d chrome` (`-d web` is not a device id; that error is common).
- Android/iOS with exactly one device attached → let Flutter auto-select.
- More than one → pin it: `flutter devices` to list, then `-d <id>`.

Confirm the device is actually visible before debugging anything else —
USB debugging off, or an unauthorized adb prompt on the device, looks identical
to a broken toolchain.

## Quality commands

```bash
flutter analyze
dart format .          # NOT `flutter format` — removed in Flutter 3.38
flutter test
flutter clean          # first thing to try on an inexplicable build failure
```

In a melos monorepo, run these across packages with `melos exec -- <cmd>`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| App runs but no data, login fails | `BASE_URL` define empty or pointing at a dead/old backend | Run through the task runner so defines are injected; verify the URL the app compiled with |
| Feature that needs a secret misbehaves, ids come back empty | Only the fallback defines were passed, the rest are `""` | Generate the secrets file so `--dart-define-from-file` injects everything |
| Android build: "flavor must be specified" | Bare `flutter run` on a multi-flavor app | Pass `--flavor`, or use the task-runner target that does |
| `flutter format` → "Could not find a command named format" | Removed in Flutter 3.38 | Use `dart format .` |
| Device not found / `-d web` fails | `web` is not a device id | Use `-d chrome`; `flutter devices` to list real ids |
| Blank white page on web after changing a path dependency | Stale incremental web build | `rm -rf build/web` and rebuild — see **flutter-web** |
| Works locally, breaks in CI | SDK version drift | Pin with FVM and commit `.fvmrc` |

## Rules

1. Prefer the repo's task-runner target over a hand-typed `flutter run`.
2. Never commit the secrets file; generate it, and gitignore the pattern.
3. A fallback for missing config must warn loudly, never fail silently.
4. Pin the Flutter SDK version in the repo.
5. Changing a `--dart-define` needs a rebuild — hot reload will not pick it up.
