# CleverTap Flutter SDK — Integration Reference

> Self-contained integration guide for the CleverTap Flutter Plugin.
> Covers Dart API, Android native setup, iOS native setup, and web support.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Flutter** | 2.0+ |
| **Dart** | 2.12+ (null safety) |
| **Android** | `minSdkVersion 21`, AndroidX required |
| **iOS** | iOS 10+, Xcode 14+ |
| **Web** | Modern browser (Chrome, Firefox, Safari, Edge) |
| **CleverTap account** | You need your **Account ID** and **Token** from the CleverTap dashboard (Settings > Project) |

### Regions

| Region Code | Data Center |
|---|---|
| `us1` | United States |
| `in1` | India |
| `sg1` | Singapore |
| `aps3` | Asia Pacific (Sydney) |
| `mec1` | Middle East |

If your dashboard URL contains `in1.dashboard.clevertap.com`, your region is `in1`. Match accordingly.

---

## Installation

Add the plugin to your `pubspec.yaml`:

```yaml
dependencies:
  clevertap_plugin: ^3.6.0
```

Then install:

```bash
flutter packages get
```

---

## Native Setup

The Flutter plugin bridges to native SDKs. Each platform requires its own configuration.

### Android

#### 1. Add credentials to `AndroidManifest.xml`

In `android/app/src/main/AndroidManifest.xml`, inside the `<application>` tag:

```xml
<application
    android:name=".MyApplication"
    ...>

    <meta-data
        android:name="CLEVERTAP_ACCOUNT_ID"
        android:value="YOUR_ACCOUNT_ID"/>
    <meta-data
        android:name="CLEVERTAP_TOKEN"
        android:value="YOUR_TOKEN"/>
    <!-- Optional: set your region. Omit for default (us1). -->
    <meta-data
        android:name="CLEVERTAP_REGION"
        android:value="YOUR_REGION"/>

    ...
</application>
```

#### 2. Create the Application class

In `android/app/src/main/kotlin/.../MyApplication.kt`:

```kotlin
import android.app.Application
import com.clevertap.android.sdk.ActivityLifecycleCallback
import com.clevertap.android.sdk.CleverTapAPI

class MyApplication : Application() {
    override fun onCreate() {
        // CRITICAL: register BEFORE super.onCreate()
        ActivityLifecycleCallback.register(this)
        super.onCreate()

        // Optional: set debug level (0=off, 1=info, 2=debug, 3=verbose)
        CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
    }
}
```

> **Warning:** `ActivityLifecycleCallback.register(this)` **MUST** be called before `super.onCreate()`. If placed after, the SDK will not track app launches and sessions correctly.

#### 3. Gradle configuration (Flutter 3.16+)

Flutter 3.16+ uses declarative `plugins {}` blocks. In `android/app/build.gradle`:

```groovy
plugins {
    id "com.android.application"
    id "kotlin-android"
    id "dev.flutter.flutter-gradle-plugin"
}

android {
    namespace "com.example.yourapp"
    compileSdk 34

    defaultConfig {
        minSdk 21
        targetSdk 34
    }
}

dependencies {
    // Required for App Inbox audio/video (v2.5.0+)
    implementation "androidx.media3:media3-exoplayer:1.3.0"
    implementation "androidx.media3:media3-exoplayer-hls:1.3.0"
    implementation "androidx.media3:media3-ui:1.3.0"
    // OR legacy ExoPlayer (pre-2.5.0)
    // implementation "com.google.android.exoplayer:exoplayer:2.19.1"
    // implementation "com.google.android.exoplayer:exoplayer-hls:2.19.1"
    // implementation "com.google.android.exoplayer:exoplayer-ui:2.19.1"
}
```

### iOS

#### 1. Add credentials to `Info.plist`

In `ios/Runner/Info.plist`:

```xml
<dict>
    ...
    <key>CleverTapAccountID</key>
    <string>YOUR_ACCOUNT_ID</string>
    <key>CleverTapToken</key>
    <string>YOUR_TOKEN</string>
    <!-- Optional: set your region. Omit for default (us1). -->
    <key>CleverTapRegion</key>
    <string>YOUR_REGION</string>
    ...
</dict>
```

> **Warning:** All credential values in `Info.plist` **must** be `<string>` types. Using other types (e.g., `<integer>`) will cause silent initialization failure.

#### 2. Initialize in AppDelegate

In `ios/Runner/AppDelegate.swift`:

```swift
import UIKit
import Flutter
import CleverTapSDK
import clevertap_plugin

@main
@objc class AppDelegate: FlutterAppDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        // Initialize CleverTap
        CleverTap.autoIntegrate()
        CleverTapPlugin.sharedInstance()?.applicationDidLaunch(options: launchOptions)

        GeneratedPluginRegistrant.register(with: self)
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
}
```

> **Note:** If you handle push notifications manually (custom delegate methods), remove `CleverTap.autoIntegrate()` and use the manual integration approach instead. See the Push Notifications section below.

### Web

#### 1. Add the CleverTap script to `web/index.html`

In `web/index.html`, add the following inside the `<head>` tag:

```html
<script>
  var clevertap = {event:[], profile:[], account:[], onUserLogin:[], notifications:[], privacy:[]};
  clevertap.account.push({"id": "YOUR_ACCOUNT_ID"});
  clevertap.region = "YOUR_REGION";
  (function () {
    var wzrk = document.createElement('script');
    wzrk.type = 'text/javascript';
    wzrk.async = true;
    wzrk.src = 'https://d2r1yp2w7bby2u.cloudfront.net/js/clevertap.min.js';
    var s = document.getElementsByTagName('script')[0];
    s.parentNode.insertBefore(wzrk, s);
  })();
</script>
```

#### 2. Initialize in Dart

```dart
import 'package:clevertap_plugin/clevertap_plugin.dart';

// Initialize for web platform
CleverTapPlugin.init("YOUR_ACCOUNT_ID", "YOUR_REGION", "YOUR_TARGET_DOMAIN");
```

The third parameter (`targetDomain`) is required for proxy/custom domain setups. For standard setups, pass your CleverTap region endpoint (e.g., `"us1.clevertap-prod.com"`).

---

## Event Tracking

All event tracking is done in Dart, shared across platforms.

### Basic event (no properties)

```dart
// Empty map is required — do NOT pass null
CleverTapPlugin.recordEvent("Product Viewed", {});
```

### Event with properties

```dart
CleverTapPlugin.recordEvent("Product Viewed", {
  "Product Name": "Wireless Headphones",
  "Category": "Electronics",
  "Price": 79.99,
  "On Sale": true
});
```

### Charged event (transactions)

```dart
var chargeDetails = {
  "Amount": 119.98,
  "Currency": "USD",
  "Charged ID": "order-12345"
};

var items = [
  {
    "Product Name": "Wireless Headphones",
    "Category": "Electronics",
    "Price": 79.99,
    "Quantity": 1
  },
  {
    "Product Name": "USB-C Cable",
    "Category": "Accessories",
    "Price": 39.99,
    "Quantity": 1
  }
];

CleverTapPlugin.recordChargedEvent(chargeDetails, items);
```

### Property value rules

- Property names: strings, max 120 characters
- Property values: string, number, boolean, or Date
- Max 256 custom event properties per event
- Event names: strings, max 512 characters
- Avoid special characters in event names; use alphanumeric and spaces
- The properties map `{}` is required even when there are no properties

---

## User Identity

### `onUserLogin` — Identify a user on login

Use `onUserLogin` when a user logs in. This creates a new profile or merges with an existing one based on the identity fields.

```dart
var profileMap = {
  "Name": "Jane Doe",
  "Identity": "user-12345",          // Your internal user ID
  "Email": "jane@example.com",
  "Phone": "+14155551234",
  "Gender": "F",                      // M or F
  // Custom properties
  "Plan": "Premium",
};

CleverTapPlugin.onUserLogin(profileMap);
```

**Identity fields** that trigger profile merge:
- `Identity` — your internal user/account ID
- `Email` — email address
- `Phone` — phone number with country code

Include at least one identity field. If a profile with that identity exists, the SDK merges the data. If not, it creates a new profile.

### `profilePush` — Update the current user's profile

Use `profilePush` to update properties on the already-identified user. This does **not** create or switch profiles.

```dart
CleverTapPlugin.profilePush({
  "Plan": "Enterprise",
  "Company": "Acme Corp",
  "Score": 42
});
```

### When to use which

| Scenario | Method |
|---|---|
| User logs in | `CleverTapPlugin.onUserLogin(profileMap)` |
| Update a field on current user | `CleverTapPlugin.profilePush(map)` |
| User signs up (new account) | `CleverTapPlugin.onUserLogin(profileMap)` |
| Anonymous user becomes known | `CleverTapPlugin.onUserLogin(profileMap)` |

---

## Push Notifications

Push notification setup requires per-platform native configuration plus a Dart listener.

### Android — Firebase Cloud Messaging (FCM)

#### 1. Add Firebase to your Android app

Follow the [Firebase setup for Flutter](https://firebase.google.com/docs/flutter/setup) to add `google-services.json` to `android/app/`.

In `android/app/build.gradle`:

```groovy
plugins {
    id "com.google.gms.google-services"
}

dependencies {
    implementation platform("com.google.firebase:firebase-bom:33.0.0")
    implementation "com.google.firebase:firebase-messaging"
}
```

#### 2. Create a push notification channel (Android 8+)

In `MyApplication.kt`:

```kotlin
import android.app.NotificationChannel
import android.app.NotificationManager
import android.os.Build

class MyApplication : Application() {
    override fun onCreate() {
        ActivityLifecycleCallback.register(this)
        super.onCreate()

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                "clevertap_default",
                "Default",
                NotificationManager.IMPORTANCE_HIGH
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(channel)
        }
    }
}
```

### iOS — APNs

#### 1. Enable push capabilities

In Xcode, select your Runner target, go to **Signing & Capabilities**, and add:
- **Push Notifications**
- **Background Modes** > check **Remote notifications**

#### 2. Register for push in AppDelegate

```swift
override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    CleverTap.autoIntegrate()
    CleverTapPlugin.sharedInstance()?.applicationDidLaunch(options: launchOptions)

    // Register for push
    UNUserNotificationCenter.current().delegate = self
    UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, error in
        if granted {
            DispatchQueue.main.async {
                application.registerForRemoteNotifications()
            }
        }
    }

    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
}

override func application(
    _ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
) {
    CleverTap.sharedInstance()?.setPushToken(deviceToken)
}
```

### Dart — Push listener

Set up the push notification handler in your Dart code:

```dart
import 'package:clevertap_plugin/clevertap_plugin.dart';

class _MyAppState extends State<MyApp> {
  late CleverTapPlugin _clevertapPlugin;

  @override
  void initState() {
    super.initState();
    _clevertapPlugin = CleverTapPlugin();

    // Listen for push notification clicks
    _clevertapPlugin.setCleverTapPushClickedPayloadReceivedHandler(
      (Map<String, dynamic> payload) {
        debugPrint("Push clicked: $payload");
        // Handle deep link or navigation
      }
    );

    // Listen for push permission response (Android 13+)
    _clevertapPlugin.setCleverTapPushPermissionResponseReceivedHandler(
      (bool accepted) {
        debugPrint("Push permission: $accepted");
      }
    );
  }
}
```

### Request push permission (Android 13+)

```dart
// Check and request notification permission on Android 13+
CleverTapPlugin.promptPushPrimer({
  "inAppType": "half-interstitial",
  "titleText": "Enable Notifications",
  "messageText": "Get updates on your orders and offers",
  "followDeviceOrientation": true,
  "positiveBtnText": "Allow",
  "negativeBtnText": "Not Now",
  "fallbackToSettings": true
});
```

---

## Common Pitfalls

### 1. `ActivityLifecycleCallback.register()` MUST be before `super.onCreate()`

This is the single most common Flutter integration error. If `register()` is called after `super.onCreate()`, the SDK silently fails to capture the app launch event and sessions will not track correctly.

```kotlin
// WRONG
override fun onCreate() {
    super.onCreate()
    ActivityLifecycleCallback.register(this) // Too late
}

// CORRECT
override fun onCreate() {
    ActivityLifecycleCallback.register(this) // Before super
    super.onCreate()
}
```

### 2. AndroidX Media3 required for App Inbox audio/video (v2.5.0+)

Starting with CleverTap Flutter Plugin v2.5.0+, App Inbox media (audio/video) requires AndroidX Media3 instead of the legacy ExoPlayer:

```groovy
// Required for v2.5.0+
implementation "androidx.media3:media3-exoplayer:1.3.0"
implementation "androidx.media3:media3-exoplayer-hls:1.3.0"
implementation "androidx.media3:media3-ui:1.3.0"
```

If you are still on an older plugin version, use the legacy ExoPlayer dependencies instead. Do not mix both.

### 3. Declarative `plugins {}` block for Gradle (Flutter 3.16+)

Flutter 3.16+ generates projects with declarative Gradle syntax. If you upgrade from an older Flutter version and your `android/app/build.gradle` still uses the imperative `apply plugin:` pattern, you may see build errors. Migrate to:

```groovy
plugins {
    id "com.android.application"
    id "kotlin-android"
    id "dev.flutter.flutter-gradle-plugin"
}
```

### 4. Remove `[CleverTap autoIntegrate]` if handling push manually on iOS

If you implement your own `UNUserNotificationCenterDelegate` or custom push handling, **remove** the `CleverTap.autoIntegrate()` call. `autoIntegrate` swizzles delegate methods and will conflict with your custom implementation:

```swift
// Manual push handling — do NOT use autoIntegrate
// CleverTap.autoIntegrate()  // Remove this line
CleverTap.sharedInstance()  // Use sharedInstance directly instead
CleverTapPlugin.sharedInstance()?.applicationDidLaunch(options: launchOptions)
```

### 5. Info.plist credentials must be strings

All CleverTap keys in `Info.plist` must use `<string>` type. A common mistake is using `<integer>` for the account ID if it looks numeric:

```xml
<!-- WRONG -->
<key>CleverTapAccountID</key>
<integer>123456789</integer>

<!-- CORRECT -->
<key>CleverTapAccountID</key>
<string>YOUR_ACCOUNT_ID</string>
```

### 6. Web: region and target domain params required for proxy setups

When using CleverTap through a proxy or custom domain, you must pass all three parameters to `init`:

```dart
// Standard setup
CleverTapPlugin.init("YOUR_ACCOUNT_ID", "YOUR_REGION", "YOUR_TARGET_DOMAIN");
```

Omitting the region or target domain when behind a proxy causes events to route to the wrong endpoint or fail silently.

### 7. Empty map required for events with no properties

Unlike the web SDK, the Flutter plugin requires an explicit empty map for events without properties. Passing `null` will throw an error:

```dart
// WRONG
CleverTapPlugin.recordEvent("Page Viewed", null);

// CORRECT
CleverTapPlugin.recordEvent("Page Viewed", {});
```

### 8. Debug logging

Enable verbose logging during development:

```dart
// Set debug level (0=off, 1=info, 2=debug, 3=verbose)
CleverTapPlugin.setDebugLevel(3);
```

On Android, also set the log level in the Application class:

```kotlin
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Remove debug logging before shipping to production.

---

## Quick Reference

```dart
import 'package:clevertap_plugin/clevertap_plugin.dart';

// 1. Identify user on login
CleverTapPlugin.onUserLogin({
  "Name": "Jane Doe",
  "Identity": "user-12345",
  "Email": "jane@example.com"
});

// 2. Track events
CleverTapPlugin.recordEvent("Feature Used", {"Feature": "Search"});

// 3. Charged event
CleverTapPlugin.recordChargedEvent(
  {"Amount": 49.99, "Currency": "USD"},
  [{"Product Name": "Widget", "Quantity": 1}]
);

// 4. Update profile
CleverTapPlugin.profilePush({"Plan": "Premium"});

// 5. Debug (dev only)
CleverTapPlugin.setDebugLevel(3);
```
