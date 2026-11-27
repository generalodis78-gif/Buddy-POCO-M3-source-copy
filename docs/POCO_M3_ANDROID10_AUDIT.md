# POCO M3 / Android 10 compatibility audit

Date: 2026-11-27
Target: Xiaomi POCO M3 (M2010J19CG), Android 10 / API 29
Working branch: `poco-m3-android10`

## Confirmed original blockers

- Gradle `minSdk`, manual AAPT2 linking, D8, and native NDK build all targeted API 30.
- The floating bubble used Android 11 `WindowInsetsAnimation`, `WindowMetrics`, typed `WindowInsets`, and `getCurrentWindowMetrics()` for layout, screen sizing, and thumbnail positioning.
- The accessibility marks overlay also used `getCurrentWindowMetrics()`.
- The chat screen used the typed IME visibility API without an API 29 fallback.
- The location tool used Android 11 `LocationManager.getCurrentLocation()` and a newer fused-provider constant.
- Android 13+ back-dispatch types were referenced directly from the bubble class.
- Manifest contains newer optional permissions/attributes; these do not alone establish runtime compatibility and still need validation on-device.

## Changes committed to this branch

- `app/build.gradle.kts`: minimum SDK lowered to 29.
- `build.sh`: AAPT2 minimum SDK and D8 minimum API lowered to 29.
- `native/build.sh`: NDK target lowered to `aarch64-linux-android29`.
- `Bubble.java`: replaced WindowMetrics display-bound queries with API-29-compatible `Display.getRealSize()`; uses stable system insets and resource fallbacks for usable display bounds and navigation-button detection.
- `Bubble.java`: only installs the API-30 WindowInsetsAnimation callback on API 30+; API 29 relies on the accessibility service's keyboard-top callback and legacy insets listener.
- `Bubble.java`: API-30 typed IME insets are guarded; API 29 does not mistake the navigation bar for the keyboard.
- `Bubble.java`: Android 13 back dispatch is accessed reflectively and held as `Object`, avoiding API-33 types in field signatures on API 29.
- `ChatActivity.java`: API 29 keyboard visibility uses the visible display frame; API 30+ keeps typed IME insets.
- `Marks.java`: uses display real size instead of API-30 WindowMetrics.
- `PhoneTools.java`: API 29 uses `LocationListener/requestLocationUpdates`; API 30+ retains `getCurrentLocation`; fused provider is considered only on API 31+. The API 29 listener runs on a private HandlerThread and is removed after the lookup.
- Workflow: builds a debug-signed test APK, verifies package ID, minSdk 29, targetSdk 36, arm64-v8a ABI and APK signature, and skips F-Droid signature comparison when no release keystore is configured.

## CI validation

Latest checked workflow run: [Build #17](https://github.com/generalodis78-gif/Buddy-POCO-M3-source-copy/actions/runs/46019162834)

- Helper tests: passed.
- Native libraries: built successfully for API 29.
- GitHub and F-Droid release APK variants: assembled successfully.
- Debug test APK: generated, package ID checked, minSdk 29 checked, targetSdk 36 checked, arm64-v8a ABI checked, and signature verification passed.
- Artifact uploaded as `buddy-apk`.
- F-Droid signature reproducibility check was skipped because no repository release keystore is configured. This is expected for unsigned release variants.

CI compilation and APK metadata verification are not an Android 10 runtime test. They do not prove that the overlay, accessibility service, speech, permissions, embedded Linux/PRoot, or background behavior work on the physical phone.

## Remaining acceptance gates

1. Install the debug-signed test APK on a real POCO M3 running Android 10 / API 29.
2. Exercise setup, accessibility overlay, bubble positioning, keyboard behavior, voice input/output, permissions, notifications, schedules, location, embedded Linux/PRoot, and background persistence.
3. Check memory/storage behavior on the actual device and verify that the expected `arm64-v8a` ABI is supported.
4. Configure a persistent release keystore before producing an updateable release APK. The CI debug key is temporary and should be treated as a first-install test signature only.

## Current status

The API 29 compatibility changes compile, native libraries build, helper tests pass, and CI verifies an installable debug-signed test APK with the expected package/minimum SDK/ABI metadata. The branch is a CI-validated Android 10 test candidate, not yet a device-verified port. No claim is made that every Buddy feature already works on the POCO M3.
