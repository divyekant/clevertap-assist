# CleverTap iOS SDK — Debugging Reference

> Platform-specific debugging guide for the CleverTap iOS SDK (Swift).
> For cross-platform issues (identity, data constraints, dashboard lag), see `common.md`.

---

## Diagnostic Checklist

Run through these checks in order when something is not working:

1. **Info.plist keys** — Are `CleverTapAccountID` and `CleverTapToken` present and correct? Are they `<string>` types (not `<integer>` or `<data>`)?
2. **Region** — If your account is not on the default US region, is `CleverTapRegion` set in Info.plist?
3. **Initialization timing** — Is `CleverTap.autoIntegrate()` or `CleverTap.sharedInstance()` called inside `application(_:didFinishLaunchingWithOptions:)`, not later?
4. **SceneDelegate vs. AppDelegate** — If using SceneDelegate (iOS 13+ default), is `autoIntegrate()` still in AppDelegate, not in SceneDelegate?
5. **Debug logging** — Is `CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)` placed BEFORE `autoIntegrate()` or `sharedInstance()`?
6. **Xcode console output** — Filter the console for `[CleverTap]`. Are there any error or warning messages?
7. **Push entitlements** — Under Signing & Capabilities, are "Push Notifications" and "Background Modes > Remote notifications" enabled?
8. **APNs certificate/key** — Is the correct APNs certificate (or p8 key) uploaded to the CleverTap dashboard under Settings > Channels > Mobile Push?
9. **Provisioning profile** — Does your provisioning profile include the Push Notifications entitlement? Regenerate it in the Apple Developer portal if unsure.
10. **SDK version compatibility** — Check the [release notes](https://github.com/CleverTap/clevertap-ios-sdk/releases) for breaking changes between your Xcode version and the SDK version.

---

## Symptoms, Causes, and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| No events recorded at all | `autoIntegrate()` not called, or called after `didFinishLaunchingWithOptions` returns | Move `CleverTap.autoIntegrate()` to the very beginning of `application(_:didFinishLaunchingWithOptions:)` |
| No events recorded — Info.plist looks correct | Info.plist values are wrong type (e.g., `<integer>` instead of `<string>`) | Verify both `CleverTapAccountID` and `CleverTapToken` are `<string>` entries |
| Events recorded but under wrong project | Account ID or Token is from a different CleverTap project | Verify credentials match Settings > Project in the correct dashboard |
| `autoIntegrate()` called but sessions not tracked | Called too late — after a splash screen delay or in `viewDidLoad` of the first controller | Call in `didFinishLaunchingWithOptions` synchronously, not in a `DispatchQueue.asyncAfter` |
| Push notifications not received | APNs certificate not uploaded to dashboard, or wrong certificate type (sandbox vs. production) | Upload the correct APNs certificate or p8 key to CleverTap dashboard under Settings > Channels > Mobile Push |
| Push not received — certificate is correct | Entitlements missing in Xcode project | Add Push Notifications capability under Signing & Capabilities; also enable Background Modes > Remote notifications |
| Push not received — entitlements are correct | `setPushToken` not called, or device token not forwarded | Implement `application(_:didRegisterForRemoteNotificationsWithDeviceToken:)` and call `CleverTap.sharedInstance()?.setPushToken(deviceToken)` |
| Push not received — all setup looks correct | Foreground notification delegate missing; notification arrives but is not displayed | Implement `UNUserNotificationCenterDelegate.willPresent` and return `[.banner, .badge, .sound]` in the completion handler |
| Push works on device but not in simulator | APNs does not work in the iOS Simulator (pre-Xcode 14); Xcode 14+ supports simulated push but not real APNs | Test push on a physical device; use Xcode 14+ simulated push payloads for basic UI testing only |
| Crash on launch with `autoIntegrate()` | SDK version incompatible with current Xcode/Swift version, or missing framework linkage | Update the SDK to the latest version; verify CleverTapSDK.framework is linked in General > Frameworks |
| Crash on launch — bridging header issues | SDK < 3.1.4 requires a manual bridging header for Swift | Upgrade to SDK 3.1.4+ (no bridging header needed), or create `YourApp-Bridging-Header.h` with `#import <CleverTapSDK/CleverTap.h>` |
| User profiles merging incorrectly across apps | IDFV collision — multiple apps from the same vendor share an IDFV | Set `CleverTapDisableIDFV` to `true` in Info.plist |
| `onUserLogin` creates duplicate profiles instead of merging | Identity field (`Identity`, `Email`, or `Phone`) not included in the profile dictionary | Ensure at least one identity field is present in the dictionary passed to `onUserLogin()` |
| Xcode build error: "No such module 'CleverTapSDK'" | SPM package not resolved, or CocoaPods not installed | For SPM: File > Packages > Resolve Package Versions; for CocoaPods: run `pod install` and open `.xcworkspace` |
| Xcode build error after SDK upgrade | SPM package structure changed between major versions | Check release notes for migration steps; remove and re-add the package if needed |

---

## Debug Tooling

### Enable debug logging

```swift
// Place BEFORE autoIntegrate() or sharedInstance()
CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)
```

Log levels:
- `.off` — No output
- `.info` — Basic lifecycle events
- `.debug` — Detailed SDK activity including event payloads and network calls

### Filter Xcode console

In the Xcode Debug Console, use the filter field at the bottom to search for:

```
[CleverTap]
```

All SDK log entries are prefixed with `[CleverTap]`.

### Common log messages

| Console Message | Meaning |
|---|---|
| `[CleverTap]: CleverTap SDK initialized with Account ID: ...` | SDK successfully started |
| `[CleverTap]: Pushing event: ...` | Event being sent to the server |
| `[CleverTap]: Network error: ...` | Request failed — check connectivity or region mismatch |
| `[CleverTap]: Account ID or Token is missing` | Info.plist keys are missing or empty |
| `[CleverTap]: Push token registered` | Device token was sent to CleverTap |
| `[CleverTap]: User profile updated` | `onUserLogin` or `profilePush` succeeded locally |

### Verify push token registration

```swift
// In your didRegisterForRemoteNotificationsWithDeviceToken
func application(
    _ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
) {
    let tokenString = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    print("[DEBUG] APNs device token: \(tokenString)")
    CleverTap.sharedInstance()?.setPushToken(deviceToken)
}

// Check for registration failure
func application(
    _ application: UIApplication,
    didFailToRegisterForRemoteNotificationsWithError error: Error
) {
    print("[DEBUG] APNs registration failed: \(error.localizedDescription)")
}
```

### Verify CleverTap ID

```swift
if let ctID = CleverTap.sharedInstance()?.profileGetCleverTapID() {
    print("[DEBUG] CleverTap ID: \(ctID)")
}
```

---

## Known Issues

### IDFV collisions

| Issue | Details |
|---|---|
| Multiple apps sharing one CleverTap account | IDFV (Identifier for Vendor) is the same for all apps from the same developer. If multiple apps share a CleverTap account, user profiles will merge incorrectly. |
| Fix | Add `<key>CleverTapDisableIDFV</key><true/>` to Info.plist for each app. |

### SceneDelegate lifecycle (iOS 13+)

| Issue | Details |
|---|---|
| `autoIntegrate()` in SceneDelegate | Placing `autoIntegrate()` in `SceneDelegate` instead of `AppDelegate` causes the SDK to miss the app launch event. |
| Fix | Always call `autoIntegrate()` in `AppDelegate.application(_:didFinishLaunchingWithOptions:)`, even in SceneDelegate-based apps. |
| Scene-based lifecycle not tracked | `autoIntegrate()` uses method swizzling on AppDelegate lifecycle methods. If your app transitions to scenes without calling AppDelegate lifecycle methods, sessions may not be tracked. |
| Fix | For complex scene setups, consider manual initialization with explicit `recordAppLaunched()` calls in `sceneDidBecomeActive`. |

### Swift Package Manager version changes

| Issue | Details |
|---|---|
| Library target name changes | Between major SDK versions, the SPM library target name may change (e.g., from `CleverTap` to `CleverTapSDK`). This breaks the import. |
| Fix | Check release notes before upgrading. Remove and re-add the package dependency if the target name changed. |
| Version resolution conflicts | If other dependencies also depend on CleverTap with different version constraints, SPM resolution may fail. |
| Fix | Use "Up to Next Major Version" version rules to allow flexibility. |

### Version-specific notes

| SDK Version | Notes |
|---|---|
| < 3.1.4 | Objective-C only; Swift requires a bridging header. |
| 3.1.4+ | Module map included; no bridging header needed for Swift. |
| 4.0.0+ | Minimum deployment target raised. Check release notes for exact iOS version requirements. |
| 5.0.0+ | Significant API changes. Methods renamed for Swift conventions. Review migration guide. |
| 7.0.0+ | Dropped support for iOS 9 and 10. Minimum is iOS 11+. |
