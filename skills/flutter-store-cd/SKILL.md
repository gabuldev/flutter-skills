---
name: flutter-store-cd
description: Use when setting up or debugging continuous delivery of a Flutter app to the App Store and Google Play with GitHub Actions - a tag that builds and uploads both apps, Android upload-keystore signing, iOS manual signing with a certificate and provisioning profile on a macOS runner, App Store Connect API keys, Play service accounts, build numbers, or when a store upload or CI build fails (draft app errors, SDK version 409, altool timeouts, codesign hangs, pods refusing a profile). Also covers the first-release traps around the pipeline - first manual Play upload, Play App Signing fingerprints for Firebase and App Check, and submitting the first subscriptions.
---

You are setting up, or fixing, the pipeline that takes a Flutter app from a git
tag to TestFlight and Google Play. The design in one line: **push a `v*` tag,
two independent workflows build, sign and upload each platform; nothing else
triggers a store build.**

Two ready workflows sit next to this file, written in full from a production app.
Copy them to `.github/workflows/`, then set the `env:` block at the top of each:

- [`templates/android-play.yml`](templates/android-play.yml): ubuntu, signs an
  App Bundle, uploads it to Play
- [`templates/ios-testflight.yml`](templates/ios-testflight.yml): macOS, archives
  and signs an IPA, uploads it to TestFlight

The rest of this skill is the reasoning behind them and the traps each line
guards against. Read it before changing a line: most lines exist because
something broke.

## The release flow

```bash
# 1. bump the version name in pubspec.yaml (the build number is ignored, see below)
version: 1.4.0+1

# 2. commit, merge, tag, push
git tag v1.4.0 && git push origin v1.4.0
```

- **One tag, two workflows.** Both trigger on `push: tags: ['v*']`, so both
  stores get the same version. Each also has `workflow_dispatch`, so you can rerun
  one platform alone.
- **Never on every push.** macOS runners bill at 10x. On the Free plan, 2,000
  minutes are about 200 macOS minutes, a few dozen iOS builds a month.
- **Android on ubuntu.** It already has the Android SDK and a JDK, and bills at
  1x. Building Android on macOS pays ten times more for nothing.
- **Gates before compiling.** `flutter analyze` and `flutter test` (and
  `dart test`/`dart analyze` for each local package) run first. They take
  seconds and fail before any expensive compile time is spent.

## Build numbers: let CI own them

```bash
flutter build appbundle --build-number="${{ github.run_number }}"
flutter build ios       --build-number="${{ github.run_number }}"
```

Both stores reject a build number they have already seen. `github.run_number`
only goes up, so nobody has to remember to bump `+N` in `pubspec.yaml`. The
version *name* still comes from `pubspec.yaml`.

Each workflow has its **own** counter, so Android and iOS build numbers differ
for the same tag. That's fine, since the stores never compare them.

**Renaming or recreating a workflow file resets its counter**, and the next
upload collides with an old number. If you must, add an offset:
`--build-number=$(( ${{ github.run_number }} + 100 ))`.

Pass the tag into the app if you show it anywhere:
`--dart-define=APP_VERSION=${{ github.ref_name }}`.

## Secrets vs variables

- **Secrets** hold whatever lets someone ship as you:
  - the keystore and its passwords
  - the `.p12` and its password
  - the provisioning profile
  - the App Store Connect `.p8`
  - the Play service-account JSON
- **Variables** hold public values:
  - Apple team id and profile name
  - client-side SDK keys that are meant to ship inside the app (RevenueCat
    `appl_`/`goog_`, Firebase options). Putting them in secrets only hides them
    from the logs where you need to debug.
- A missing `--dart-define` is an **empty string**, not an error. Make the app
  degrade safely on an empty key, for example paywalls unlocked instead of
  crashing, and remember that an empty variable in CI ships exactly that.
- Write keys to `$RUNNER_TEMP` (or `$HOME/...` with a `trap ... EXIT` that deletes
  them), never inside the working tree. Base64 binary secrets straight to the
  clipboard so they never land in shell history:
  - macOS: `base64 -i file | pbcopy`. The BSD `base64` has no `-w`, and its output
    is already one line.
  - Linux: `base64 -w0 file | xclip -selection clipboard`

## Android

### Gradle reads `key.properties`, falls back to debug

```kotlin
// android/app/build.gradle.kts
val keystorePropertiesFile = rootProject.file("key.properties")
val keystoreProperties = Properties().apply {
    if (keystorePropertiesFile.exists()) {
        FileInputStream(keystorePropertiesFile).use { load(it) }
    }
}

android {
    signingConfigs {
        create("release") {
            keyAlias = keystoreProperties.getProperty("keyAlias")
            keyPassword = keystoreProperties.getProperty("keyPassword")
            storePassword = keystoreProperties.getProperty("storePassword")
            storeFile = keystoreProperties.getProperty("storeFile")?.let { file(it) }
        }
    }
    buildTypes {
        release {
            // No key.properties means debug signing, so `flutter run --release`
            // still works locally. Play rejects a debug-signed bundle, so a
            // missing file in CI fails loudly at upload instead of shipping.
            signingConfig = if (keystorePropertiesFile.exists())
                signingConfigs.getByName("release")
            else signingConfigs.getByName("debug")
        }
    }
}
```

- Gitignore `key.properties`, `*.jks` and `*.keystore`. CI writes
  `key.properties` with `storeFile` pointing at `$RUNNER_TEMP/upload.jks`.
- Build with **`flutter build appbundle`**, not `./gradlew`. `settings.gradle.kts`
  reads `flutter.sdk` from `local.properties`, which only `flutter` generates.
- AGP 8.x needs **JDK 17** (`actions/setup-java`, temurin 17).

### Uploading to Play

| Trap | What happens / what to do |
|---|---|
| First upload of a new app | The API refuses it. Upload the first `.aab` by hand in Play Console (download it from the workflow artifact). |
| App never published yet | `Only releases with status draft may be created on draft app`. Use `status: draft` until the first publish. |
| Still `draft` after publishing | Every later release sits as a draft nobody installs. It looks like success. **Switch to `completed`** right after the first publish. |
| `track:` singular | Deprecated in `r0adkll/upload-google-play`. Use `tracks: internal` (comma-separated). |
| Personal account (created after 13 Nov 2023) | Production unlocks only after 12 testers spend 14 days in **closed** testing (`alpha`). Internal testing doesn't count. Organization accounts are exempt: check the account type on the Console home before planning around it. |
| Service account | Create it in Google Cloud, then invite its email in Play Console → Users and permissions with release rights for the app. |

### Play App Signing changes your SHA fingerprints

With Play App Signing (the default), **the build users install is signed by
Google's key, not your upload key**. Anything that checks the signature against
a registered fingerprint works in your sideloaded build and fails only from the
store:

- **Firebase App Check (Play Integrity)**, plus Google Sign-In and other
  Google APIs keyed by SHA
- Symptom: every App Check-protected call (Firebase AI Logic, Firestore with
  enforcement) is rejected in production only.

Fix:
1. Copy the **app signing key** SHA-256 (and the upload key's) from Play
   Console → Test and release → App integrity → App signing.
2. Register them on the Firebase Android app (`firebase apps:android:sha:create
   <appId> <sha>`).
3. Register the app in App Check → Apps → Play Integrity.

The fix takes effect without a new release, once the app restarts.

## iOS

### Signing: manual, in the Release config only

Set this on the **Runner target's Release** build configuration in
`ios/Runner.xcodeproj/project.pbxproj`, in Xcode or by hand:

```
CODE_SIGN_STYLE = Manual;
DEVELOPMENT_TEAM = <TEAM_ID>;
CODE_SIGN_IDENTITY = "Apple Distribution";
"CODE_SIGN_IDENTITY[sdk=iphoneos*]" = "Apple Distribution";
PROVISIONING_PROFILE_SPECIFIER = "<profile name>";
```

- **Not on the `xcodebuild` command line.** Passed there, it applies to every
  target, including the CocoaPods ones, which refuse a provisioning profile and
  break the archive.
- Debug and Profile stay `Automatic`, so local development is unaffected.
- `flutter build ios --release --no-codesign` compiles the Dart side, then
  `xcodebuild archive` and `xcodebuild -exportArchive`, with an
  `ExportOptions.plist` generated in the workflow, sign the app.

### Runner traps

| Trap | Guard |
|---|---|
| `codesign` hangs until timeout | It is waiting on a keychain prompt. Use a temp keychain plus `security set-key-partition-list -S apple-tool:,apple:,codesign:`. |
| `Validation failed (409) SDK version issue` | App Store Connect requires a minimum iOS SDK. Pin `xcode-version` (`maxim-lobanov/setup-xcode`) to one that meets it, with a Flutter version that supports that Xcode. |
| Firebase pods fail to compile | The Firebase iOS SDK needs a recent Swift. Same fix: newer Xcode. |
| `altool` fails with NSURLErrorDomain -1001 | A transient CDN timeout before the upload. Retry three times, 60 s apart, so a finished 15-minute build isn't lost. |
| Bundle id mismatch | iOS bundle ids are case-sensitive and can differ from the Android `applicationId` (`com.x.myApp` vs `com.x.myapp`). Keep a separate env var. |

### Getting the credentials (once)

| Secret | Where |
|---|---|
| Distribution `.p12` | Keychain Access → request a CSR → developer.apple.com → Certificates → Apple Distribution → install → export as `.p12` with a password |
| Provisioning profile | developer.apple.com → Profiles → App Store Connect distribution, for the bundle id and that certificate. Its *name* goes in `IOS_PROFILE_NAME`. |
| App Store Connect API key | App Store Connect → Users and Access → Integrations → App Store Connect API → **App Manager** role. The `.p8` downloads only once. |

### Entitlements that only matter in release

If the app uses **App Attest** (Firebase App Check on iOS), add to
`Runner.entitlements`:

```xml
<key>com.apple.developer.devicecheck.appattest-environment</key>
<string>production</string>
```

Without it, TestFlight and App Store builds get "App Check token is invalid"
while everything else looks right.

## First submission: traps the pipeline can't catch

The upload is the easy half. These blocked the first review of an app with
subscriptions:

- **Agreements first.** Paid apps and in-app purchases need the *Paid Apps*
  agreement active (bank account and tax forms done) in App Store Connect →
  Business. Until then StoreKit returns no products and the paywall loads
  empty, even in TestFlight.
- **Each subscription needs a review screenshot** of the purchase screen. A
  widget test that pumps the real paywall with a fake store and writes a PNG at
  the required size (for example 1206×2622) is quicker than a simulator.
- **The first subscriptions must go in the same submission as an app
  version**, together with their subscription group. A version submitted
  without them ships an app whose purchases fail in production. Add the
  subscriptions, the group and the version to one draft submission, then
  submit it.
- **Apps that need hardware** (Bluetooth adapters, wearables) get a "2.1
  Information Needed" asking for a video filmed by a second camera, showing a
  physical device and the hardware pairing and working together. A built-in
  simulator mode helps the reviewer but doesn't replace the video. Budget for
  it before the first submission.
- **New developer accounts** are asked for a screen recording plus written
  answers (purpose, setup, external services, regions, in-app purchases). Keep
  them under 4,000 characters, which is the limit for both the reply and the
  App Review Notes field. After replying, **resubmit**: a reply alone can leave
  the submission parked in "Unresolved issues".
