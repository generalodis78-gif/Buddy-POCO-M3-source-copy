# POCO M3 / Android 10 compatibility audit

Date: 2026-11-27
Target device: Xiaomi POCO M3 (M2010J19CG), Android 10 / API 29
Source snapshot in this repository: Buddy 2.4.0 (versionCode 264)
Upstream checked separately: Buddy 2.5.1 (versionCode 328)

## Current conclusion

The checked-in application is explicitly built for Android 11 / API 30 and cannot be considered Android 10 compatible. Do not lower only `minSdk`: there are Android 11 framework APIs used in normal UI and location paths, and the native build also targets API 30.

No Android 10 compatibility changes have been applied yet. This file records confirmed blockers before implementation.

## Confirmed build-level blockers

- `app/build.gradle.kts`: `minSdk = 30`.
- `build.sh`: AAPT2 uses `--min-sdk-version 30`; D8 uses `--min-api 30`.
- `native/build.sh`: NDK target is `aarch64-linux-android30` (`API=30`).
- Native build produces only `arm64-v8a` libraries. Device ABI must be verified before an installable build is accepted.

## Confirmed Android 11+ framework API blockers

### Floating bubble / window geometry — `Bubble.java`

- `WindowInsetsAnimation` and `View.setWindowInsetsAnimationCallback` are Android 11 / API 30 APIs, used by `followKeyboard()`.
- `WindowManager.getCurrentWindowMetrics()`, `WindowMetrics`, `WindowInsets.Type`, and `WindowInsets.getInsets(...)` are API 30 APIs. Calls occur in bubble layout, screen sizing, navigation-bar and keyboard calculations.
- Android 10 needs a separate compatibility path using pre-30 display metrics / visible display frame / resource insets. Gesture inset calculations need a conservative fallback; they must not be simulated with invented values.

### Location — `PhoneTools.java`

- `LocationManager.getCurrentLocation(...)` is API 30.
- The code references `LocationManager.FUSED_PROVIDER`, which is newer than API 29 and must be guarded or avoided on Android 10.
- API 29 needs a request-location-updates / last-known-location fallback with timeout and correct listener removal.

### Newer optional APIs

- `Bubble.java` references Android 13 back-dispatch APIs and Android 14 predictive-back animation APIs. Existing SDK checks are present, but class loading / verification on API 29 must be tested; if needed, isolate newer types behind a separate helper or reflection.
- `MainActivity.java` also uses Android 13 back-dispatch types behind an SDK check.
- Manifest contains newer declarations/attributes such as notification permission, Bluetooth connect, promoted notifications, locale config and back callback behavior. These need a compatibility review and runtime validation; merely leaving unknown declarations in the manifest is not proof of compatibility.

## Other build and packaging observations

- Gradle uses compileSdk 36, AGP 9.4.1 and Gradle wrapper 9.8.0. These are build-host requirements, not by themselves a reason an API 29 device cannot run the resulting APK.
- The app includes an embedded Alpine/PRoot Linux environment and native libraries. Successful Java compilation alone is insufficient: native library loading, PRoot startup, executable permissions, terminal PTY, and the first Alpine download must be tested.
- The workflow builds both GitHub and F-Droid release variants and compiles native code with a fixed NDK. A CI APK build is a build test, not a POCO device runtime test.

## Planned implementation and acceptance gates

1. Rebase/update the working source against the current upstream snapshot only after reviewing the upstream changes; do not silently mix version 2.4.0 and 2.5.x files.
2. Add API 29 compatibility for bubble keyboard/window geometry and location.
3. Set Gradle, AAPT2, D8 and native NDK minimums consistently to API 29, then compile and inspect the APK manifest and native ABI.
4. Add API-level checks or isolated helpers for every API 30+ and later framework call used on normal paths.
5. Run helper tests and both release-variant CI builds.
6. Verify APK package metadata, minSdk, signing state, native ABI and shared-library dependencies.
7. Only label the result POCO M3 compatible after installation and runtime tests on an actual Android 10 / API 29 device. Until then, report it as a CI-built candidate, not a verified working app.

## Status

Audit completed for the build configuration, manifest, native build script, and primary bubble/location code paths. Full source-wide API audit and implementation remain in progress.
