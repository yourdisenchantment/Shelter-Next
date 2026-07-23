# AGENTS.md - Shelter

## Project

- Android app (Java + AIDL) that manages a Work Profile for app isolation/cloning/freeze.
- Package: `net.typeblog.shelter`. minSdk 24, compileSdk/targetSdk 35. AGP 8.6.0, Gradle 8.7 wrapper, Java 8 source.
- Distribution: Google Play repackage only (see `repackage/repackage.sh`). F-Droid is NOT a target -- do not edit `metadata/`, do not push to F-Droid.
- Canonical remote is `origin = git@github.com:yourdisenchantment/Shelter-Next.git` (`dev` = development, `main` = release-only). Upstream PeterCxy/Shelter on gitea is NOT configured as a remote and is read-only reference at most.

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
- Translations live in `app/src/main/res/values-*/`. Translations are NOT maintained in this fork -- do not touch `values-*/strings.xml` files except for `values/strings.xml` (English source) when adding user-facing strings. Do not run Weblate sync.

## Git workflow

- Branches (all pushed to `origin`, GitHub):
  - `dev` -- the working line. New commits go **directly** here (signed). This is the clean starting baseline for all Shelter-Next work.
  - `main` -- frozen release branch, carrying the inherited upstream history. Left untouched until the first release, then updated **only via a squash-merge PR** from `dev`, so each release lands as one clean commit.
  - `feature/<name>` -- optional, for larger or riskier work; branched off `dev`, merged back into `dev`.
- Protection:
  - `dev` -- force-push and deletion are blocked (safety net); direct pushes are allowed. CI ("Android CI") runs on every push but does not gate pushes.
  - `main` -- strict: PR required, the `build` check must pass, force-push and deletion blocked, enforced for admins.
- Release flow: open a PR `dev` -> `main`, wait for green CI, **squash-merge** (collapses to one clean commit on `main`), then tag that commit (e.g. `v1.x.y`) and push the tag.
- Commits and tags are **SSH-signed** (repo-level config: `gpg.format=ssh`, `user.signingkey=~/.ssh/id_ed25519.pub`, `commit.gpgsign=true`, `tag.gpgsign=true`). The public key is registered on GitHub as a signing key. Do not disable signing.
- The agent **never** runs `git add`, `git commit`, `git push`, `git merge`, `git rebase`, `git tag`, or PR merges (writing). It only inspects and proposes commands.

## Commit messages

- Commits are made **manually** with `git commit`. No `commitizen`, no `pre-commit` -- previous attempts to use them were abandoned (commitizen 4.16.5 in this Python env forces interactive prompts, `pre-commit` has no config in this repo and the global tool just adds noise).
- The agent **drafts** commit message text but **does not commit**. Drafts go to `tmp/commits/NN-slug.txt` in the project (NN = `01`, `02`, ... in commit order; a single commit is just `01-slug.txt`).
- One file per commit. NN is the order in which the agent proposes commits for the current uncommitted set.
- Format: Conventional Commits prefix (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`, ...). First line -- English imperative, <= 65 chars, no trailing period. Body -- bullets per logical change, blank line before body.
- The user copies the subject and body from the file into `git commit -m "subject" -m "body" -m "next bullet" ...` (or pastes the whole message into `git commit` without `-m`).
- `tmp/` is gitignored; do not commit its contents.

## Google Play repackage

- `repackage/repackage.sh` rebuilds a Play-safe APK by stripping `MANAGE_EXTERNAL_STORAGE` and disabling the File Shuttle setting. It depends on `apktool` and `$ANDROID_HOME/build-tools/30.0.2/{zipalign,apksigner}` -- the build-tools version is hard-coded and old; pass `$ANDROID_HOME` explicitly. The script prompts for keystore path/alias, so it is interactive -- do not run it unattended.

## Style / gotchas

- Java 8 source/target; no Kotlin, no Compose, no DataBinding.
- No `BuildConfig` generation beyond the AGP default (no custom `buildConfigField`).
- `android:allowBackup="false"` and `android:installLocation="internalOnly"` are intentional.
- `gradle.properties` enables Jetifier (`android.enableJetifier=true`); don't disable it -- some transitive deps still need it.
- The `app/jcenter` repository pulls exactly one artifact (`mobi.upod:time-duration-picker:1.1.3`); do not remove it without also removing the picker usage.
- No CI config in the repo; no pre-commit, no ktlint/checkstyle, no detekt. Lint is the only static check.
