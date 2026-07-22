# AGENTS.md - Shelter

## Project

- Android app (Java + AIDL) that manages a Work Profile for app isolation/cloning/freeze.
- Package: `net.typeblog.shelter`. minSdk 24, compileSdk/targetSdk 35. AGP 8.6.0, Gradle 8.7 wrapper, Java 8 source.
- Distribution: F-Droid + a Google Play repackage (see `repackage/repackage.sh`).
- The GitHub repo is a mirror; the canonical remote is `https://gitea.angry.im/PeterCxy/Shelter.git`. Issues/PRs are not the supported workflow -- use the mailing list (see `README.md`).

## Build & run

- Use the wrapper, do not call system gradle: `./gradlew <task>`.
- `local.properties` (sdk.dir) is gitignored; create one with `sdk.dir=/path/to/Android/Sdk` before first build.
- Submodule `libs/SetupWizardLibrary` is required. After a fresh clone run `git submodule update --init --recursive` (or pass `--recursive` to clone). The build will fail with cryptic errors if it is empty.
- Module map (`settings.gradle`):
  - `:app` -- the application.
  - `:setup-wizard-lib` -- points at `libs/SetupWizardLibrary/library` and uses `standalone.gradle` (fetches support libs from Maven, not from the AOSP tree). The app depends on it via `gingerbreadCompat{Debug,Release}RuntimeElements` (see `app/build.gradle:85-86`).
- `versionCode` / `versionName` are derived from `git` at build time (`app/build.gradle:12-38`). A build outside a git checkout yields `versionCode = -1` and `versionName = null`, which breaks update checks. Always build from a git working tree.
- Debug builds override `versionCode` to a per-second timestamp; do not compare debug vs release codes.
- `compileSdk 35` requires `buildToolsVersion = '35.0.0'` already set -- do not bump one without the other.

## Useful commands

- Build debug APK: `./gradlew :app:assembleDebug`
- Build release APK (unsigned): `./gradlew :app:assembleRelease` (release uses R8 + resource shrinking; `proguard-rules.pro` is essentially empty)
- Install: `./gradlew :app:installDebug` (requires a connected device or emulator with adb)
- Lint: `./gradlew :app:lint` (lint disables `MissingTranslation`, `ExtraTranslation`, `GoogleAppIndexingWarning`, `InvalidFragmentVersionForActivityResult` in `app/build.gradle:67-69`)
- Unit tests: `./gradlew :app:test{Debug,Release}UnitTest` -- only the auto-generated `ExampleUnitTest` exists under `app/src/test`. There is no real unit test suite.
- Instrumented tests: `./gradlew :app:connectedAndroidTest` -- only the auto-generated `ExampleInstrumentedTest` exists under `app/src/androidTest`. No CI runs them.
- Clean: `./gradlew clean` (deletes `rootProject.buildDir`)

## Architecture map

- Entry points (`app/src/main/AndroidManifest.xml`):
  - `ui.MainActivity` -- launcher; bind/binds services across main + work profile.
  - `ui.SetupWizardActivity` -- first-run flow; uses `:setup-wizard-lib`.
  - `ui.DummyActivity` -- transparent activity that exposes ~14 internal actions (`net.typeblog.shelter.action.*`) so intents can cross the profile boundary; many cross-profile operations route through it.
  - `ui.FinalizeActivity` -- listens for `ACTION_PROVISIONING_SUCCESSFUL` (replaces the legacy device-admin finalize path on Oreo+).
  - `receivers.ShelterDeviceAdminReceiver` -- `BIND_DEVICE_ADMIN`-gated device policy controller.
- Services (`app/src/main/java/net/typeblog/shelter/services/`):
  - `ShelterService` -- core; runs in both main and work profile, bound by `ShelterApplication` (see `ShelterApplication.java:24-51`).
  - `FileShuttleService` -- proxies file FDs across profiles.
  - `FreezeService` -- screen-lock freezing (AlarmManager-based, not a foreground service started by the app).
  - `PaymentStubService` -- NFC payment stub, `enabled="false"`, toggled at runtime.
  - `KillerService` -- ensures all `ShelterService` instances die when the app is swiped from recents.
- AIDL interfaces under `app/src/main/aidl/net/typeblog/shelter/{services,util}/*.aidl` define the IPC contracts for the services and for `UriForwardProxy`.
- Providers: `FileProviderProxy` (authority `net.typeblog.shelter.files`) and `CrossProfileDocumentsProvider` (`...documents`, `MANAGE_DOCUMENTS`-gated, `enabled="false"` by default).
- Utility classes: `LocalStorageManager`, `SettingsManager`, `AuthenticationUtility`, `ApplicationInfoWrapper`, `UriForwardProxy`, `Utility`, `InstallationProgressListener`.
- Translations live in `app/src/main/res/values-*/`. Translations are managed upstream in Weblate; do not edit `values-*/strings.xml` by hand for languages you do not maintain -- see `CHANGELOG.md:30-31` for the workflow. F-Droid metadata is in `metadata/en-US/`.

## Google Play repackage

- `repackage/repackage.sh` rebuilds a Play-safe APK by stripping `MANAGE_EXTERNAL_STORAGE` and disabling the File Shuttle setting. It depends on `apktool` and `$ANDROID_HOME/build-tools/30.0.2/{zipalign,apksigner}` -- the build-tools version is hard-coded and old; pass `$ANDROID_HOME` explicitly. The script prompts for keystore path/alias, so it is interactive -- do not run it unattended.

## Style / gotchas

- Java 8 source/target; no Kotlin, no Compose, no DataBinding.
- No `BuildConfig` generation beyond the AGP default (no custom `buildConfigField`).
- `android:allowBackup="false"` and `android:installLocation="internalOnly"` are intentional.
- `gradle.properties` enables Jetifier (`android.enableJetifier=true`); don't disable it -- some transitive deps still need it.
- The `app/jcenter` repository pulls exactly one artifact (`mobi.upod:time-duration-picker:1.1.3`); do not remove it without also removing the picker usage.
- No CI config in the repo; no pre-commit, no ktlint/checkstyle, no detekt. Lint is the only static check.
