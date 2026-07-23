# Shelter-Next

> Shelter-Next is an unofficial fork of [Shelter](https://gitea.angry.im/PeterCxy/Shelter) by PeterCxy. It targets newer Android versions, is being ported to Kotlin, and is maintained independently.

Shelter is a Free and Open-Source (FOSS) app that leverages the "Work Profile" feature of Android to provide an isolated space that you can install or clone apps into.

## Features

- Installing apps inside a work profile for isolation
- "Freeze" apps inside the work profile to prevent them from running or being woken up when you are not actively using them
- Installing two copies of the same app on the same device

## Status

In development. No public builds are available yet.

- `compileSdk` / `targetSdk` 35, `minSdk` 24.
- A gradual Kotlin port and a modern-SDK migration are in progress.

## Building

`local.properties` with `sdk.dir=/path/to/Android/Sdk` is required, and the `libs/SetupWizardLibrary` submodule must be initialised before the first build:

```
git submodule update --init --recursive
```

Then:

```
./gradlew :app:assembleDebug     # debug APK
./gradlew :app:assembleRelease   # release APK (unsigned)
```

Requires JDK 21 and the Android SDK (platform 35, build-tools 35.0.0). The build derives `versionCode`/`versionName` from git, so build from a git checkout with full history.

## License

Shelter-Next is distributed under the GNU General Public License v3.0 (see [LICENSE](LICENSE)), unchanged from upstream. See [NOTICE](NOTICE) for attribution. Original work © PeterCxy and the Shelter contributors.
