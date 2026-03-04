# CleverTap iOS SDK — Integration Reference

> Self-contained integration guide for the CleverTap iOS SDK.
> All examples use Swift. Objective-C equivalents are available in the [official docs](https://developer.clevertap.com/docs/ios-quickstart-guide).

---

## Prerequisites

| Requirement | Details |
|---|---|
| **iOS deployment target** | iOS 9.0+ |
| **Xcode** | 12.0+ |
| **CleverTap account** | You need your **Account ID** and **Token** from the CleverTap dashboard (Settings > Project) |

### Regions (optional)

| Region Code | Data Center |
|---|---|
| `us1` | United States |
| `in1` | India |
| `sg1` | Singapore |
| `aps3` | Asia Pacific (Sydney) |
| `mec1` | Middle East |

Only required if your account is **not** on the default US region. Set via `CleverTapRegion` in Info.plist.

---

## Installation

### Option 1: CocoaPods (recommended)

Add to your `Podfile`:

```ruby
pod "CleverTap-iOS-SDK"
```

Then install:

```bash
pod install
```

Open the generated `.xcworkspace` (not `.xcodeproj`) going forward.

### Option 2: Swift Package Manager

1. In Xcode, go to **File > Add Packages...**
2. Enter the repository URL:
   ```
   https://github.com/CleverTap/clevertap-ios-sdk
   ```
3. Select the version rule (e.g., "Up to Next Major Version") and add the `CleverTapSDK` library to your target.

### Option 3: Manual

1. Clone or download the [clevertap-ios-sdk](https://github.com/CleverTap/clevertap-ios-sdk) repository.
2. Drag `CleverTapSDK.xcodeproj` into your project's navigator.
3. In your target's **General > Frameworks, Libraries, and Embedded Content**, add `CleverTapSDK.framework`.
4. If you plan to use **App Inbox** with images, also add [SDWebImage](https://github.com/SDWebImage/SDWebImage) as a dependency.

---

## Configuration

Add these keys to your app's `Info.plist`:

```xml
<key>CleverTapAccountID</key>
<string>YOUR_ACCOUNT_ID</string>
<key>CleverTapToken</key>
<string>YOUR_TOKEN</string>
```

### Optional keys

```xml
<!-- Only if your account is not on the default US region -->
<key>CleverTapRegion</key>
<string>YOUR_REGION</string>

<!-- Set to 1 if multiple iOS apps share one CleverTap account -->
<key>CleverTapDisableIDFV</key>
<true/>
```

| Key | Type | Purpose |
|---|---|---|
| `CleverTapAccountID` | String | Your project's Account ID |
| `CleverTapToken` | String | Your project's Token |
| `CleverTapRegion` | String | Data center region code (e.g., `in1`, `sg1`) |
| `CleverTapDisableIDFV` | Boolean | Disables IDFV-based identity; set `true` for multi-app setups sharing one CT account |

---

## Initialization

### Option A: Auto-integrate (recommended)

The simplest approach. CleverTap automatically hooks into app lifecycle events (launch, foreground, background) via method swizzling.

```swift
// AppDelegate.swift
import CleverTapSDK

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {

    // Must be called here — not later in the lifecycle
    CleverTap.autoIntegrate()

    return true
}
```

### Option B: Manual initialization

If you need more control (e.g., delayed init, multiple instances):

```swift
// AppDelegate.swift
import CleverTapSDK

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {

    CleverTap.sharedInstance()  // Initializes using Info.plist credentials

    return true
}
```

With manual init, you must forward lifecycle events yourself:

```swift
func applicationDidBecomeActive(_ application: UIApplication) {
    CleverTap.sharedInstance()?.recordAppLaunched()
}
```

### Debug logging

Enable during development to see SDK activity in the Xcode console:

```swift
// Place BEFORE autoIntegrate() or sharedInstance()
CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)
```

Log levels: `.off`, `.info`, `.debug`. Remove or set to `.off` before shipping to production.

---

## Event Tracking

### Basic event (no properties)

```swift
CleverTap.sharedInstance()?.recordEvent("Product Viewed")
```

### Event with properties

```swift
CleverTap.sharedInstance()?.recordEvent("Product Viewed", withProps: [
    "Product Name": "Wireless Headphones",
    "Category": "Electronics",
    "Price": 79.99,
    "On Sale": true
])
```

### Charged event (transactions)

```swift
let chargeDetails: [String: Any] = [
    "Amount": 119.98,
    "Currency": "USD",
    "Charged ID": "order-12345"
]

let item1: [String: Any] = [
    "Product Name": "Wireless Headphones",
    "Category": "Electronics",
    "Price": 79.99,
    "Quantity": 1
]

let item2: [String: Any] = [
    "Product Name": "USB-C Cable",
    "Category": "Accessories",
    "Price": 39.99,
    "Quantity": 1
]

CleverTap.sharedInstance()?.recordChargedEvent(
    withDetails: chargeDetails,
    andItems: [item1, item2]
)
```

### Property value rules

- Property names: strings, max 120 characters
- Property values: `String`, `Int`, `Double`, `Bool`, or `Date`
- Max 256 custom properties per event
- Event names: strings, max 512 characters
- Avoid special characters in event names; use alphanumeric and spaces

---

## User Identity

### `onUserLogin` — Identify a user on login or signup

Creates a new profile or merges with an existing one based on identity fields.

```swift
let profile: [String: Any] = [
    "Name": "Jane Doe",
    "Identity": "user-12345",       // Your internal user ID
    "Email": "jane@example.com",
    "Phone": "+14155551234",
    "Gender": "F",                   // M or F
    "DOB": NSDate(),                 // Date of birth
    // Custom properties
    "Plan": "Premium",
    "Signup Date": NSDate()
]

CleverTap.sharedInstance()?.onUserLogin(profile)
```

**Identity fields** that trigger profile merge:
- `Identity` — your internal user/account ID
- `Email` — email address
- `Phone` — phone number with country code

Include at least one identity field. If a profile with that identity exists, the SDK merges the data. If not, it creates a new profile.

### `profilePush` — Update the current user's profile

Use `profilePush` to update properties on the already-identified user. This does **not** create or switch profiles.

```swift
CleverTap.sharedInstance()?.profilePush([
    "Plan": "Enterprise",
    "Company": "Acme Corp",
    "Score": 42
])
```

### When to use which

| Scenario | Method |
|---|---|
| User logs in | `onUserLogin(...)` |
| Update a field on current user | `profilePush(...)` |
| User signs up (new account) | `onUserLogin(...)` |
| Anonymous user becomes known | `onUserLogin(...)` |

---

## Push Notifications

### 1. Request notification permission

```swift
// AppDelegate.swift or a dedicated notification manager
import UserNotifications

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {

    CleverTap.autoIntegrate()

    // Request push permission
    UNUserNotificationCenter.current().requestAuthorization(
        options: [.alert, .badge, .sound]
    ) { granted, error in
        if granted {
            DispatchQueue.main.async {
                UIApplication.shared.registerForRemoteNotifications()
            }
        }
    }

    // Set the notification delegate
    UNUserNotificationCenter.current().delegate = self

    return true
}
```

### 2. Pass the device token to CleverTap

```swift
func application(
    _ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
) {
    CleverTap.sharedInstance()?.setPushToken(deviceToken)
}
```

### 3. Handle received notifications

```swift
// Called when a notification is tapped (app in background or killed)
func userNotificationCenter(
    _ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse,
    withCompletionHandler completionHandler: @escaping () -> Void
) {
    CleverTap.sharedInstance()?.handleNotification(
        withData: response.notification.request.content.userInfo
    )
    completionHandler()
}

// Called when a notification arrives while app is in foreground
func userNotificationCenter(
    _ center: UNUserNotificationCenter,
    willPresent notification: UNNotification,
    withCompletionHandler completionHandler:
        @escaping (UNNotificationPresentationOptions) -> Void
) {
    CleverTap.sharedInstance()?.handleNotification(
        withData: notification.request.content.userInfo
    )
    completionHandler([.banner, .badge, .sound])
}
```

### 4. AppDelegate conformance

Make sure your AppDelegate conforms to `UNUserNotificationCenterDelegate`:

```swift
@main
class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    // ...
}
```

> **Note:** If using `autoIntegrate()`, CleverTap swizzles the push token and notification delegate methods automatically. The explicit forwarding above ensures correct behavior when swizzling is disabled or when you need custom handling.

---

## Common Pitfalls

### 1. Dashboard role lacks Project Token access

Users with **Member** or **Creator** roles on the CleverTap dashboard cannot view the Project Token. Ask an **Admin** for the Account ID and Token, or request a role upgrade under Settings > Users.

### 2. IDFV collision with multiple apps

If you ship multiple iOS apps that share a single CleverTap account, the SDK uses IDFV (Identifier for Vendor) by default, which is the same across all apps from the same vendor. This causes user profiles to merge incorrectly.

Fix: add to `Info.plist`:

```xml
<key>CleverTapDisableIDFV</key>
<true/>
```

### 3. Swift bridging header (SDK < 3.1.4 only)

Versions before 3.1.4 were Objective-C only. If you are on an older version, create a bridging header:

```
// YourApp-Bridging-Header.h
#import <CleverTapSDK/CleverTap.h>
#import <CleverTapSDK/CleverTapInstanceConfig.h>
```

SDK 3.1.4+ ships with a module map — no bridging header needed.

### 4. `autoIntegrate()` must be called in `didFinishLaunchingWithOptions`

Calling `autoIntegrate()` later (e.g., after a splash screen delay or in `viewDidLoad`) means the SDK misses the launch event and may not track the session correctly.

```swift
// WRONG: delayed init
DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) {
    CleverTap.autoIntegrate()  // Misses app launch
}

// CORRECT: call immediately in didFinishLaunchingWithOptions
func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    CleverTap.autoIntegrate()
    return true
}
```

### 5. SPM versioning

The CleverTap iOS SDK's SPM package structure may change between major versions. When upgrading, check the [release notes](https://github.com/CleverTap/clevertap-ios-sdk/releases) for any changes to library names or module targets.

### 6. SceneDelegate apps (iOS 13+)

If your app uses `SceneDelegate` (the default for new Xcode projects), `autoIntegrate()` still goes in `AppDelegate.application(_:didFinishLaunchingWithOptions:)`. Do not put it in `SceneDelegate`.

---

## Quick Reference

```swift
import CleverTapSDK

// 1. Enable debug logging (dev only)
CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)

// 2. Initialize (in AppDelegate didFinishLaunchingWithOptions)
CleverTap.autoIntegrate()

// 3. Identify user on login
CleverTap.sharedInstance()?.onUserLogin([
    "Name": "Jane Doe",
    "Identity": "user-12345",
    "Email": "jane@example.com"
])

// 4. Track events
CleverTap.sharedInstance()?.recordEvent("Feature Used", withProps: [
    "Feature": "Search"
])

// 5. Update profile
CleverTap.sharedInstance()?.profilePush([
    "Plan": "Premium"
])

// 6. Push token (in didRegisterForRemoteNotificationsWithDeviceToken)
CleverTap.sharedInstance()?.setPushToken(deviceToken)
```
