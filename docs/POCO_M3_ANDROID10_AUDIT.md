# POCO M3 / Android 10 compatibility audit

Date: 2026-11-27
Target: Xiaomi POCO M3 (M2010J19CG), Android 10 / API 29
Working branch: `poco-m3-android10`

## Confirmed original blockers

- Gradle `minSdk`, manual AAPT2 linking, D8, and native NDK build all targeted API 30.
- The floating bubble used Android 11 `WindowInsetsAnimation`, `WindowMetrics`, typed `WindowInsets`, and `getCurrentWindowMetrics()` for layout, screen sizing, and thumbnail positioning.
- The location tool used Android 11 `LocationManager.getCurrentLocation()` and a newer fused-provider constant.
- Android 13+ back-dispatch types were referenced directly from the bubble class.
- Manifest contains newer optional permissions/attributes; these do not alone establish runtime compatibility and still need validation on-device.

## Changes committed to this branch

- `app/build.gradle.kts`: minimum SDK lowered to 29.
- `build.sh`: AAPT2 minimum SDK and D8 minimum API lowered to 29.
- `native/build.sh`: NDK target lowered to `aarch64-linux-android29`.
- `Bubble.java`: replaced WindowMetrics display-bound queries with API-29-compatible `Display.getRealSize()`; uses stable system insets and resource fallbacks for usable display bounds and navigation-button detection.
- `Bubble.java`: only installs the API-30 WindowInsetsAnimation callback on API 30+; API 29 relies on the existing accessibility keyboard-top callback and legacy insets listener.
- `Bubble.java`: API-30 typed IME insets are guarded; API 29 does not mistake the navigation bar for the keyboard.
- `Bubble.java`: Android 13 back dispatch is accessed reflectively and held as `Object`, avoiding API-33 types in field signatures on API 29.
- `PhoneTools.java`: API 29 uses `LocationListener/requestLocationUpdates`; API 30+ retains `getCurrentLocation`; fused provider is considered only on API 31+.
- Workflow: F-Droid signature reproducibility check is skipped when no release keystore is configured, because unsigned artifacts cannot be compared as signed APKs.

## CI findings

- Helper tests passed.
- Native libraries built successfully with the API 29 NDK target.
- Both GitHub and F-Droid release APK variants completed Gradle assembly successfully in an earlier run.
- That run failed only at the signature-comparison step because this repository has no configured release keystore; the workflow condition was corrected. A newer workflow run is checking the updated branch.
- CI compilation is not an Android 10 runtime test. It does not prove that the overlay, accessibility service, speech, permissions, embedded Linux/PRoot, or background behavior work on the physical phone.

## Remaining work / acceptance gates

1. Confirm the latest CI run builds both variants and completes the workflow.
2. Inspect the final APK metadata: package ID, minSdk 29, target SDK, signature state, and `arm64-v8a` native libraries.
3. Continue the source-wide audit for API 30+ calls and ensure every newer API is isolated behind a valid SDK check or compatibility implementation.
4. Verify 64-bit ABI and available storage/RAM on the actual POCO M3.
5. Install and exercise the APK on a real Android 10 / API 29 device. Until this step passes, the build is only a CI-built compatibility candidate, not a verified working POCO M3 port.

## Current status

The first compatibility changes are committed. Helper tests and native compilation have passed; Gradle APK assembly has also passed in a prior run. The latest workflow is still being checked. No on-device runtime test has been performed, and no APK is being represented as fully verified.
