# CleverTap React Native SDK — Debugging Reference

> Platform-specific debugging guide for the CleverTap React Native SDK.
> For cross-platform issues (identity, data constraints, dashboard lag), see `common.md`.

---

## Diagnostic Checklist

Run through these checks in order when something is not working:

1. **Package installed** — Is `clevertap-react-native` listed in `package.json` dependencies?
2. **iOS pods installed** — Did you run `cd ios && pod install`? Are you opening `.xcworkspace` (not `.xcodeproj`)?
3. **Android manifest** — Are `CLEVERTAP_ACCOUNT_ID` and `CLEVERTAP_TOKEN` in `android/app/src/main/AndroidManifest.xml` inside `<application>`?
4. **iOS Info.plist** — Are `CleverTapAccountID` and `CleverTapToken` present as `<string>` values?
5. **Android Application class** — Is `ActivityLifecycleCallback.register(this)` called BEFORE `super.onCreate()` in `MainApplication`?
6. **iOS AppDelegate** — Is `CleverTap.autoIntegrate()` called in `didFinishLaunchingWithOptions`? Is `CleverTapReactManager.sharedInstance()?.applicationDidLaunch(options:)` called after it?
7. **JS import** — Is `import CleverTap from 'clevertap-react-native'` present in your JavaScript code?
8. **No JS init needed** — Initialization happens on the native side. Do NOT call an `init()` method from JS.
9. **Debug logging on BOTH platforms** — Enable native debug logging on Android (Logcat) and iOS (Xcode console) independently.
10. **React Native version compatibility** — Check the SDK's `package.json` or changelog for the minimum supported React Native version.

---

## Symptoms, Causes, and Fixes

### Module / linking issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Cannot find module 'clevertap-react-native'` | Package not installed | Run `npm install --save clevertap-react-native` |
| iOS build fails: "Module not found" | CocoaPods not installed or `.xcworkspace` not used | Run `cd ios && pod install && cd ..`; open `ios/YourApp.xcworkspace`, not `.xcodeproj` |
| Android build fails: "Could not resolve CleverTap" | SDK dependency not resolved | Run `cd android && ./gradlew clean && cd ..`; verify Gradle can reach Maven Central |
| iOS build fails after RN upgrade: `#import` errors with modular headers | Modular headers enabled but SDK imports use old syntax | Switch to `@import CleverTapSDK;` in any custom native code, or add `use_modular_headers!` to Podfile |

### Events not reaching dashboard

| Symptom | Likely Cause | Fix |
|---|---|---|
| Events from JS not appearing in dashboard | Native initialization missing on one or both platforms | Verify BOTH Android (`ActivityLifecycleCallback.register` + manifest meta-data) and iOS (`autoIntegrate` + Info.plist) are set up correctly |
| Events work on iOS but not Android | Android-side setup incomplete | Check Logcat for `CleverTap` — verify Account ID is found, `ActivityLifecycleCallback` is registered before `super.onCreate()` |
| Events work on Android but not iOS | iOS-side setup incomplete | Check Xcode console for `[CleverTap]` — verify Info.plist keys and `autoIntegrate()` call |
| JS `recordEvent` called but nothing happens | SDK not initialized on native side before JS bridge is ready | Ensure native init happens in `Application.onCreate()` (Android) and `didFinishLaunchingWithOptions` (iOS) — these run before JS loads |
| Events silently dropped | Event name or properties violate constraints (empty name, too many properties) | Enable debug logging on native side; check logs for validation errors |

### App crashes on start

| Symptom | Likely Cause | Fix |
|---|---|---|
| Android crash: `libhermes.so` load error | Conflict between CleverTap native module and Hermes in older RN versions | Upgrade to React Native 0.70+ with CleverTap SDK v5+; if stuck on older RN, disable Hermes in `android/app/build.gradle` |
| Android crash: `ClassNotFoundException` | SDK v6.1.1+ upgrade without background sync meta-data | Add `<meta-data android:name="CLEVERTAP_BACKGROUND_SYNC" android:value="1"/>` to manifest |
| iOS crash on launch | SDK version incompatible with Xcode version, or missing framework linkage | Update SDK; verify CleverTapSDK framework is linked in Xcode |
| iOS crash: "CleverTapReactManager not found" | `clevertap_react_native` not imported in AppDelegate, or pods not installed | Add `import clevertap_react_native` to AppDelegate; run `pod install` |
| Android crash: `HashMap` ClassCastException | Kotlin code using `mapOf()` instead of `HashMap` | Use `HashMap<String, Any>()` in any native Android code that interfaces with CleverTap |

---

## Debug Tooling

### JavaScript-side logging

```javascript
// Set debug level (0=off, 1=info, 2=debug, 3=verbose)
CleverTap.setDebugLevel(3);
```

This affects the native SDK's log verbosity. Output appears in the native console (Logcat / Xcode), not in the React Native JS debugger.

### Android native-side logging

```kotlin
// In MainApplication.kt, BEFORE ActivityLifecycleCallback.register()
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Filter Logcat for tag `CleverTap`:

```bash
adb logcat -s CleverTap:V
```

### iOS native-side logging

```swift
// In AppDelegate.swift, BEFORE autoIntegrate()
CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)
```

Filter Xcode console for `[CleverTap]`.

### React Native remote debugger

The JS remote debugger (Chrome DevTools) shows JavaScript-level output but NOT native SDK logs. To see CleverTap SDK logs, you must check the native console:
- **Android:** Android Studio Logcat
- **iOS:** Xcode Debug Console

### Flipper / React Native Debugger

Network requests from the CleverTap native SDK do NOT appear in Flipper's network tab (Flipper only intercepts JS-side `fetch`/`XMLHttpRequest`). To inspect CleverTap network traffic:
- **Android:** Use `adb logcat` with `CleverTap` tag at VERBOSE level
- **iOS:** Use Xcode console with `[CleverTap]` filter, or use a network proxy like Charles/Proxyman

### Verify CleverTap ID from JS

```javascript
CleverTap.getCleverTapID((err, res) => {
  if (err) {
    console.error('CleverTap ID error:', err);
  } else {
    console.log('CleverTap ID:', res);
  }
});
```

---

## Known Issues

### Turbo Modules (New Architecture)

| Issue | Details |
|---|---|
| SDK does not work with New Architecture enabled | Turbo Module support requires CleverTap React Native SDK v2.0.0+ and React Native 0.71+. |
| Build fails with `newArchEnabled=true` | Older SDK versions do not have Turbo Module specs. Upgrade to v2.0.0+. |
| iOS: pods fail with `RCT_NEW_ARCH_ENABLED=1` | Run `RCT_NEW_ARCH_ENABLED=1 pod install --project-directory=ios` to install with New Architecture support. |

### Media3 for App Inbox (v3.0.0+)

| Issue | Details |
|---|---|
| Inbox messages with audio/video crash on Android | Starting with SDK v3.0.0, AndroidX Media3 is required instead of legacy ExoPlayer. |
| Fix | Add to `android/app/build.gradle`: `implementation 'androidx.media3:media3-exoplayer:1.2.0'`, `implementation 'androidx.media3:media3-exoplayer-hls:1.2.0'`, `implementation 'androidx.media3:media3-ui:1.2.0'`. |
| Do not mix Media3 and ExoPlayer | Using both Media3 and legacy ExoPlayer dependencies causes class conflicts. Remove the old ExoPlayer dependencies when upgrading. |

### Modular headers (iOS)

| Issue | Details |
|---|---|
| Build fails with `#import` errors after CocoaPods upgrade | CocoaPods may enable modular headers which changes how headers are resolved. |
| Fix | Add `use_modular_headers!` to your `Podfile`, or switch custom native code from `#import <CleverTapSDK/CleverTap.h>` to `@import CleverTapSDK;`. |

### Hermes engine compatibility

| Issue | Details |
|---|---|
| Crash on Android with `libhermes.so` error | Older versions of Hermes conflict with certain native modules. |
| Fix | Upgrade to React Native 0.70+ and CleverTap SDK v5+. If upgrading is not possible, disable Hermes: set `enableHermes: false` in `android/app/build.gradle`. |

### Version-specific notes

| SDK Version | RN Version | Notes |
|---|---|---|
| 1.x | 0.60+ | Legacy Bridge architecture only. |
| 2.0.0+ | 0.71+ | Added Turbo Module support for New Architecture. |
| 3.0.0+ | 0.71+ | Migrated from ExoPlayer to AndroidX Media3 for Inbox media. |
