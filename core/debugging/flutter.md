# CleverTap Flutter Plugin — Debugging Reference

> Platform-specific debugging guide for the CleverTap Flutter Plugin.
> For cross-platform issues (identity, data constraints, dashboard lag), see `common.md`.

---

## Diagnostic Checklist

Run through these checks in order when something is not working:

1. **Plugin installed** — Is `clevertap_plugin` listed in `pubspec.yaml` dependencies? Did you run `flutter packages get`?
2. **Android manifest** — Are `CLEVERTAP_ACCOUNT_ID` and `CLEVERTAP_TOKEN` present as `<meta-data>` inside `<application>` in `android/app/src/main/AndroidManifest.xml`?
3. **Android Application class** — Is `ActivityLifecycleCallback.register(this)` called BEFORE `super.onCreate()` in `MyApplication.kt`?
4. **Android Application registered** — Is `android:name=".MyApplication"` set on the `<application>` tag in the manifest?
5. **iOS Info.plist** — Are `CleverTapAccountID` and `CleverTapToken` present and typed as `<string>` (not `<integer>` or other types)?
6. **iOS AppDelegate** — Is `CleverTap.autoIntegrate()` called in `didFinishLaunchingWithOptions`? Is `CleverTapPlugin.sharedInstance()?.applicationDidLaunch(options:)` called after it?
7. **Dart import** — Is `import 'package:clevertap_plugin/clevertap_plugin.dart'` present in your Dart code?
8. **Empty map for no-property events** — Are you passing `{}` (not `null`) as the second argument to `recordEvent` when there are no properties?
9. **Debug logging on BOTH native sides** — Enable verbose logging in both Android (`CleverTapAPI.setDebugLevel`) and iOS (`CleverTap.setDebugLevel`) independently.
10. **Gradle configuration** — If using Flutter 3.16+, is `android/app/build.gradle` using the declarative `plugins {}` block?

---

## Symptoms, Causes, and Fixes

### Plugin not initializing

| Symptom | Likely Cause | Fix |
|---|---|---|
| Android: SDK not initializing, no CleverTap logs in Logcat | `ActivityLifecycleCallback.register(this)` called AFTER `super.onCreate()` | Move `ActivityLifecycleCallback.register(this)` to before `super.onCreate()` in `MyApplication.kt` |
| Android: SDK not initializing — lifecycle order is correct | Application class not registered in manifest | Add `android:name=".MyApplication"` to the `<application>` tag in `AndroidManifest.xml` |
| iOS: SDK not initializing, no `[CleverTap]` logs | `CleverTap.autoIntegrate()` not called in AppDelegate, or Info.plist keys missing | Add `autoIntegrate()` call and verify Info.plist keys |
| iOS: silent initialization failure | Info.plist credential values are wrong type (e.g., `<integer>` instead of `<string>`) | Change `CleverTapAccountID` and `CleverTapToken` to `<string>` type in Info.plist |
| Both platforms: no errors but nothing works | Wrong Account ID or Region; data goes to a different project | Verify credentials match Settings > Project on the correct CleverTap dashboard |

### Events lost or not recorded

| Symptom | Likely Cause | Fix |
|---|---|---|
| `recordEvent` throws error or silently fails | Passing `null` instead of `{}` for events with no properties | The Flutter plugin requires an explicit empty map: `CleverTapPlugin.recordEvent("Name", {})` |
| Events work on one platform but not the other | Native setup incomplete on the failing platform | Check native logs on both platforms independently; fix the platform-specific setup |
| Events appear in logs but not in dashboard | Region mismatch: events sent to wrong data center | Verify `CLEVERTAP_REGION` (Android manifest) and `CleverTapRegion` (iOS Info.plist) match your dashboard URL |
| Charged event not recorded | Items array is empty or missing required fields | Ensure the items list has at least one item with `"Product Name"` and `"Quantity"` |

### Web platform not working

| Symptom | Likely Cause | Fix |
|---|---|---|
| Flutter web: no events tracked | CleverTap script tag missing from `web/index.html` | Add the CleverTap Web SDK script tag inside `<head>` in `web/index.html` with your Account ID and Region |
| Flutter web: script tag present but still not working | `CleverTapPlugin.init()` not called in Dart for web | Call `CleverTapPlugin.init("ACCOUNT_ID", "REGION", "TARGET_DOMAIN")` in your Dart code for web platform |
| Flutter web: events sent to wrong endpoint | Missing or wrong target domain in `CleverTapPlugin.init()` | The third parameter (`targetDomain`) is required for proxy/custom domain setups |
| Flutter web: CSP errors in browser console | Content Security Policy blocks CleverTap requests | Add `*.clevertap.com` and `d2r1yp2w7bby2u.cloudfront.net` to your CSP `connect-src` and `script-src` |

### Build failures

| Symptom | Likely Cause | Fix |
|---|---|---|
| Android build fails on Flutter 3.16+ | Gradle configuration uses old `apply plugin:` syntax | Migrate `android/app/build.gradle` to declarative `plugins {}` block |
| Android build fails: Media3 class not found | Missing AndroidX Media3 dependencies for App Inbox | Add `media3-exoplayer`, `media3-exoplayer-hls`, and `media3-ui` to `android/app/build.gradle` dependencies |
| Android build fails: duplicate class errors | Both Media3 and legacy ExoPlayer in dependencies | Remove legacy ExoPlayer dependencies when using plugin v2.5.0+; use only Media3 |
| iOS build fails: "No such module" | Pods not installed or outdated | Run `cd ios && pod install --repo-update && cd ..` |

---

## Debug Tooling

### Dart-side logging

```dart
// Set debug level (0=off, 1=info, 2=debug, 3=verbose)
CleverTapPlugin.setDebugLevel(3);
```

This sets the log level on the native SDK. Output appears in native consoles, not in Flutter DevTools console.

### Android native-side logging

```kotlin
// In MyApplication.kt, after ActivityLifecycleCallback.register()
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Filter Logcat:

```bash
adb logcat -s CleverTap:V
```

### iOS native-side logging

```swift
// In AppDelegate.swift, BEFORE autoIntegrate()
CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)
```

Filter Xcode console for `[CleverTap]`.

### Flutter DevTools

Flutter DevTools does NOT show native SDK logs or network requests from the CleverTap plugin. To inspect CleverTap activity:

- **Android:** Use Android Studio Logcat with `CleverTap` tag filter
- **iOS:** Use Xcode Debug Console with `[CleverTap]` filter

For network-level inspection, use a proxy tool like Charles or Proxyman.

### Verify initialization from Dart

```dart
// Check if CleverTap has a device ID assigned
CleverTapPlugin.getCleverTapID().then((id) {
  debugPrint("CleverTap ID: $id");
});
```

If this returns `null`, the SDK has not initialized on the native side.

### Debug push token

```dart
// Android: Check FCM token
CleverTapPlugin.getCleverTapID().then((id) {
  debugPrint("CleverTap device ID: $id");
});

// Listen for push permission response (Android 13+)
_clevertapPlugin.setCleverTapPushPermissionResponseReceivedHandler((accepted) {
  debugPrint("Push permission granted: $accepted");
});
```

---

## Known Issues

### Gradle declarative plugins (Flutter 3.16+)

| Issue | Details |
|---|---|
| Build fails after upgrading Flutter to 3.16+ | Flutter 3.16+ generates new projects with the declarative `plugins {}` block. If you upgraded from an older Flutter and your `build.gradle` still uses `apply plugin:`, it can conflict with the CleverTap plugin's Gradle integration. |
| Fix | Migrate `android/app/build.gradle` to use `plugins { id "com.android.application"; id "kotlin-android"; id "dev.flutter.flutter-gradle-plugin" }`. |

### Media3 vs. ExoPlayer

| Issue | Details |
|---|---|
| App Inbox audio/video crashes on Android | CleverTap Flutter Plugin v2.5.0+ requires AndroidX Media3 instead of legacy ExoPlayer for Inbox media rendering. |
| Fix | Add to `android/app/build.gradle`: `implementation "androidx.media3:media3-exoplayer:1.3.0"`, `implementation "androidx.media3:media3-exoplayer-hls:1.3.0"`, `implementation "androidx.media3:media3-ui:1.3.0"`. |
| Class conflict if both present | Do not include both Media3 and legacy ExoPlayer dependencies. Remove `com.google.android.exoplayer:exoplayer-*` when upgrading to plugin v2.5.0+. |

### Empty map requirement

| Issue | Details |
|---|---|
| `recordEvent` fails or throws with `null` properties | Unlike the Web SDK, the Flutter plugin requires an explicit empty map `{}` when recording events with no properties. Passing `null` causes a runtime error on the native bridge. |
| Fix | Always use `CleverTapPlugin.recordEvent("Event Name", {})` for no-property events. |

### Info.plist type sensitivity

| Issue | Details |
|---|---|
| iOS: SDK silently fails to initialize | `CleverTapAccountID` or `CleverTapToken` stored as `<integer>` in Info.plist instead of `<string>`. The SDK reads these as string types and gets `nil` for non-string entries. |
| Fix | Ensure all CleverTap keys in Info.plist use `<string>` type. |

### ActivityLifecycleCallback ordering

| Issue | Details |
|---|---|
| Android: sessions and app launches not tracked | `ActivityLifecycleCallback.register(this)` called after `super.onCreate()` in `MyApplication`. This is the single most common Flutter integration error. |
| Fix | Move `register(this)` to before `super.onCreate()`. |

### Version-specific notes

| Plugin Version | Flutter Version | Notes |
|---|---|---|
| 1.x | 2.0+ | Legacy API, null-safety support added. |
| 2.5.0+ | 2.0+ | Migrated from ExoPlayer to AndroidX Media3 for Inbox. |
| 3.0.0+ | 2.0+ | Web platform support added. Requires script tag in `index.html`. |
| 3.6.0+ | 3.16+ | Supports declarative Gradle `plugins {}` block. |
