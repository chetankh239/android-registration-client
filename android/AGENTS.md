# AGENTS.md — android/

Parent guide: [`../AGENTS.md`](../AGENTS.md)

## Purpose

The Flutter Android embedding project (module `app`) plus four native
Android library modules that implement the device-facing logic Flutter
calls into via Pigeon: `clientmanager`, `keymanager`, `packetmanager`,
`transliterationmanager`. See each module's own `AGENTS.md` for its
specific responsibilities; this file covers what's shared across all
five (`app` + the four managers).

## Layout

```text
android/
├── build.gradle              # root Gradle config — single source of truth for versions (see below)
├── settings.gradle            # includes :app, :clientmanager, :keymanager, :packetmanager, :transliterationmanager
├── gradle.properties           # JVM args, AndroidX/Jetifier flags
├── jacoco.gradle                 # shared coverage config, applied by the 4 manager modules (not by app)
├── gradle/wrapper/gradle-wrapper.properties   # Gradle 8.5
├── app/                            # Flutter embedding — see Configuration below
├── clientmanager/                    # AGENTS.md
├── keymanager/                         # AGENTS.md
├── packetmanager/                        # AGENTS.md
└── transliterationmanager/                 # AGENTS.md
```

`gradlew`/`gradlew.bat` and `.gradle/` are gitignored (`android/.gitignore`)
— a fresh clone has **no wrapper scripts checked in**. They come from
Android Studio/`flutter create` regenerating them, or must already exist
locally; don't assume `./gradlew` is present without checking.

## Module dependency graph

From each module's `implementation project(...)` lines:

```text
app                     → clientmanager, keymanager, packetmanager, transliterationmanager
clientmanager           → packetmanager, keymanager
packetmanager           → keymanager
keymanager              → (no module deps; testImplementation project(':clientmanager') — test-only)
transliterationmanager  → (no module deps)
```

`keymanager`'s test-only dependency on `clientmanager` is the reverse of
the production direction (`clientmanager` depends on `keymanager` in
main code) — don't be confused by this when tracing what depends on
what; it only applies to `keymanager`'s own test sources.

## `android/build.gradle` (root Gradle config)

This file centralizes almost every version number for every module in
one `ext {}` block — module `build.gradle` files reference
`rootProject.ext.*` rather than hardcoding versions. Includes AGP `8.3.2`,
Kotlin `1.9.24`, `compileSdk`/`targetSdk` 34, `minSdk` 28,
`serverBaseURL`/`serverHealthCheckPath`/`serverActuatorInfoPath` (see
root `AGENTS.md`'s Configuration section), a hardcoded placeholder
`debugPassword`, and ~50 dependency version properties (Dagger,
OkHttp/Retrofit, Jackson, BouncyCastle, jose4j, SQLCipher, PDFBox, ICU4J,
`biometrics-util`, `kernel-biometrics-api`, etc.).

Other things it does:

- Auto-detects `android.namespace` for any `com.android.library`
  subproject missing one, by parsing `package="..."` out of that
  module's `AndroidManifest.xml` (falls back to `project.group`) — an
  AGP-8 compatibility shim, mainly for third-party Flutter plugin
  subprojects.
- Disables unit tests for a hardcoded list of third-party Flutter plugin
  subprojects whose Mockito-based tests break under JDK 17/21
  (`flutter_plugin_android_lifecycle`, `geolocator_android`,
  `image_picker_android`, `url_launcher_android`,
  `shared_preferences_android`, `webview_flutter_android`).
- Suppresses lint (`NewerVersionAvailable`, `MissingPermission`) for
  every subproject **except** `app`, `clientmanager`, `keymanager`,
  `packetmanager`, `transliterationmanager` — lint is enforced on this
  repo's own modules, not on vendored plugin code.
- `rootProject.buildDir = '../build'`, with each subproject's build
  output remapped under it (`../build/<module>/...`).

**⚠️ Security note, unrelated to any specific change**: this file's
SonarQube block (`property "sonar.login", "..."`) has what looks like a
**real SonarCloud token** hardcoded and committed, alongside a
contributor's personal Windows path
(`C:\Users\sachin.sp\AndroidStudioProjects\...`) in the Jacoco report
paths. Since this is a public repo, treat that token as compromised —
it should be rotated by whoever owns it and replaced with a CI
secret/environment variable, not left in a tracked file. Do not add
another real token anywhere in this repo following that pattern.

## `android/settings.gradle`

Includes exactly 5 modules: `:app`, `:clientmanager`, `:packetmanager`,
`:keymanager`, `:transliterationmanager`. It also asserts `local.properties`
exists and reads `flutter.sdk` from it *before* applying Flutter's
`app_plugin_loader.gradle` — so a `local.properties` with a valid
`flutter.sdk` path is required just to **sync** the project, not only to
build it (see root `AGENTS.md`'s Configuration section).

Minor inconsistency worth knowing about: `clientmanager`, `packetmanager`,
and `keymanager` each get an explicit `project(':x').projectDir = ...`
remap line; `transliterationmanager`'s `include` has no matching
`projectDir` line. This is harmless (its folder name already matches the
project name) but inconsistent with the other three — don't assume a
missing `projectDir` line elsewhere is a bug without checking whether the
folder name already matches.

## `android/app/` (Flutter embedding)

- `namespace`/`applicationId`: `io.mosip.registration_client`.
  `minSdkVersion 28`, JDK 21
  (`sourceCompatibility`/`targetCompatibility VERSION_21`,
  `coreLibraryDesugaringEnabled true`).
- `buildConfigField`s `BASE_URL`/`HEALTH_CHECK_PATH`/`ACTUATOR_INFO_PATH`
  are sourced from the root `ext` block — see root `AGENTS.md`'s
  Configuration section for the placeholder-override mechanism.
- Signing: `signingConfigs.release` reads `keyAlias`/`keyPassword`/
  `storeFile`/`storePassword` from `local.properties` — `key.properties`
  (if present) is merged into that same `localProperties` object first,
  it isn't read as a separate signing config block. If `storeFile` isn't
  set, the release build type falls back to **debug** signing (comment
  in the file: "Signing with the debug keys for now, so
  `flutter run --release` works") — see root `AGENTS.md`'s Configuration
  section for why this fallback shouldn't be relied on for a real release.
- `afterEvaluate`: `mergeDebugAssets`/`mergeReleaseAssets` are made to
  `dependsOn(':clientmanager:dexifyBiosdkAars')` — the app's asset merge
  is coupled to `clientmanager`'s custom biometric-SDK dexing task (see
  `clientmanager/AGENTS.md`). If that task breaks, `app`'s build breaks
  too, even though the failure surfaces in `app`'s build log.
- Depends on all 4 manager modules, plus its own large dependency stack
  (Dagger, Room, WorkManager, OkHttp/Retrofit, PDFBox,
  `io.mosip.biometric.util:biometrics-util`, BouncyCastle, etc.).
- Defines its own inline `jacocoTestReport` task — it does **not** apply
  the shared `../jacoco.gradle` the 4 manager modules use.
- The Flutter↔native Pigeon bridge implementations live under
  `android/app/src/main/java/io/mosip/registration_client/api_services/`
  — each class there implements a generated `<Feature>Pigeon.<Feature>Api`
  interface and delegates to an `spi` service from one of the 4 manager
  modules (see each manager module's `AGENTS.md` for its `spi` interface).
  **None of the 4 manager modules reference Pigeon directly** — don't
  look for Pigeon-generated code inside them.
- `AndroidManifest.xml`: app label "Registration Client",
  `android:allowBackup="false"`, `android:usesCleartextTraffic="false"`.
  Permissions include scoped storage (`READ/WRITE_EXTERNAL_STORAGE` with
  `maxSdk`, plus `MANAGE_EXTERNAL_STORAGE` for newer Android),
  `INTERNET`, `ACCESS_NETWORK_STATE`, `WAKE_LOCK`,
  `RECEIVE_BOOT_COMPLETED`, `SCHEDULE_EXACT_ALARM`/`USE_EXACT_ALARM`,
  and fine/coarse location. Declares `.UploadBackgroundService` and
  `.MainActivity` (`flutterEmbedding=2`, confirming Flutter Android
  Embedding v2).

## Build & Test Commands

See the root `AGENTS.md` for the full Flutter-level commands
(`flutter pub get`, `sh pigeon.sh`, `flutter run`, `flutter build apk`).
From inside `android/`:

```bash
cd android
./gradlew assembleDebug
```

Per-module unit tests (only the 4 manager modules apply the shared
`jacoco.gradle` coverage config; `app` has its own):

```bash
./gradlew :clientmanager:testDebugUnitTest
./gradlew :keymanager:testDebugUnitTest
./gradlew :packetmanager:testDebugUnitTest
./gradlew :transliterationmanager:testDebugUnitTest
```

## Agent rules

### Do

1. Add new version numbers to `android/build.gradle`'s root `ext {}`
   block and reference them via `rootProject.ext.*` from the relevant
   module — don't hardcode a version directly in a module's own
   `build.gradle` unless the existing code already does so for that
   dependency.
2. Confirm which of the 5 modules a change belongs to before editing —
   check the dependency graph above so you don't introduce a reverse
   dependency (e.g. `keymanager` depending on `clientmanager` in main
   code, not just tests).
3. Remember the Pigeon `HostApi` implementations live in `android/app`'s
   `api_services/`, not in the manager modules — if you're wiring up a
   new Pigeon method, that's where the implementation goes.

### Do not

1. Do not treat the SonarQube token in `android/build.gradle` as safe to
   reuse or extend — it's a known, pre-existing exposure (see above),
   not a pattern to copy for new CI config.
2. Do not assume `./gradlew` exists in a fresh checkout — it's gitignored.
3. Do not add Pigeon references inside `clientmanager`, `keymanager`,
   `packetmanager`, or `transliterationmanager` — none of them use
   Pigeon today, and the bridge layer belongs in `android/app`.
