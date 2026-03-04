# CleverTap React Native SDK — Integration Reference

> Self-contained integration guide for the CleverTap React Native SDK.
> Covers installation, per-platform native setup, and the JavaScript API layer.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **React Native** | 0.60+ (autolinking supported) |
| **Android** | minSdkVersion 21+, compileSdkVersion 33+ |
| **iOS** | iOS 10+, Xcode 14+ |
| **CocoaPods** | Required for iOS native dependencies |
| **CleverTap account** | You need your **Account ID** and **Token** from the CleverTap dashboard (Settings > Project) |

---

## Installation

```bash
npm install --save clevertap-react-native
```

For iOS, install the native pod:

```bash
cd ios && pod install && cd ..
```

> Always open `ios/YourApp.xcworkspace` after running `pod install` — not `ios/YourApp.xcodeproj`.

---

## Native Setup

The CleverTap React Native SDK requires per-platform configuration. The native side handles SDK initialization; the JavaScript layer uses it after boot.

### Android

#### 1. Add CleverTap credentials to `AndroidManifest.xml`

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application ...>

    <meta-data
        android:name="CLEVERTAP_ACCOUNT_ID"
        android:value="YOUR_ACCOUNT_ID" />
    <meta-data
        android:name="CLEVERTAP_TOKEN"
        android:value="YOUR_TOKEN" />
    <!-- Optional: set your region. Omit for default (US). -->
    <!-- Values: us1, in1, sg1, aps3, mec1 -->
    <meta-data
        android:name="CLEVERTAP_REGION"
        android:value="YOUR_REGION" />

    <!-- Required for push notifications -->
    <meta-data
        android:name="CLEVERTAP_NOTIFICATION_ICON"
        android:value="ic_notification" />

    ...
</application>
```

#### 2. Extend the Application class

Create or update your custom `Application` class:

```kotlin
// android/app/src/main/java/com/yourapp/MainApplication.kt
import android.app.Application
import com.clevertap.android.sdk.ActivityLifecycleCallback
import com.clevertap.react.CleverTapRnAPI

class MainApplication : Application() {
    override fun onCreate() {
        ActivityLifecycleCallback.register(this)
        super.onCreate()
        // React Native initialization (SoLoader, etc.) follows...
    }
}
```

#### 3. Set initial deep link URI in `MainActivity`

```kotlin
// android/app/src/main/java/com/yourapp/MainActivity.kt
import android.os.Bundle
import com.clevertap.react.CleverTapRnAPI
import com.facebook.react.ReactActivity

class MainActivity : ReactActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        CleverTapRnAPI.setInitialUri(intent?.data)
    }
}
```

### iOS

#### 1. Add CleverTap keys to `Info.plist`

```xml
<!-- ios/YourApp/Info.plist -->
<key>CleverTapAccountID</key>
<string>YOUR_ACCOUNT_ID</string>
<key>CleverTapToken</key>
<string>YOUR_TOKEN</string>
<!-- Optional: set your region. Omit for default (US). -->
<key>CleverTapRegion</key>
<string>YOUR_REGION</string>
```

#### 2. Initialize in `AppDelegate`

```swift
// ios/YourApp/AppDelegate.swift
import CleverTapSDK
import clevertap_react_native  // CleverTapReactManager

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {

        // 1. Auto-integrate CleverTap (reads Account ID/Token from Info.plist)
        CleverTap.autoIntegrate()

        // 2. Connect CleverTap to the React Native bridge
        CleverTapReactManager.sharedInstance()?.applicationDidLaunch(options: launchOptions)

        // 3. Enable debug logging during development (remove for production)
        CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)

        return true
    }
}
```

---

## Initialization

Initialization happens on the native side (see above). On the JavaScript side, you only need to import the module:

```javascript
import CleverTap from 'clevertap-react-native';
```

No `init()` call is needed in JS — the native `autoIntegrate()` (iOS) and manifest meta-data (Android) handle it.

---

## Event Tracking

### Basic event (no properties)

```javascript
CleverTap.recordEvent('Product Viewed');
```

### Event with properties

```javascript
CleverTap.recordEvent('Product Viewed', {
  'Product Name': 'Wireless Headphones',
  'Category': 'Electronics',
  'Price': 79.99,
  'On Sale': true,
});
```

### Charged event (transactions)

```javascript
const chargeDetails = {
  'Amount': 119.98,
  'Currency': 'USD',
  'Charged ID': 'order-12345',
};

const items = [
  {
    'Product Name': 'Wireless Headphones',
    'Category': 'Electronics',
    'Price': 79.99,
    'Quantity': 1,
  },
  {
    'Product Name': 'USB-C Cable',
    'Category': 'Accessories',
    'Price': 39.99,
    'Quantity': 1,
  },
];

CleverTap.recordChargedEvent(chargeDetails, items);
```

### Property value rules

- Property names: strings, max 120 characters
- Property values: string, number, boolean, or Date
- Max 256 custom event properties per event
- Event names: strings, max 512 characters
- Avoid special characters in event names; use alphanumeric and spaces

---

## User Identity

### `onUserLogin` — Identify a user on login

Use `onUserLogin` when a user logs in. This creates a new profile or merges with an existing one based on the identity fields.

```javascript
CleverTap.onUserLogin({
  'Name': 'Jane Doe',
  'Identity': 'user-12345',          // Your internal user ID
  'Email': 'jane@example.com',
  'Phone': '+14155551234',
  'Gender': 'F',                      // M or F
  'DOB': new Date('1990-06-15'),
  // Custom properties
  'Plan': 'Premium',
});
```

**Identity fields** that trigger profile merge:
- `Identity` — your internal user/account ID
- `Email` — email address
- `Phone` — phone number with country code

Include at least one identity field. If a profile with that identity exists, the SDK merges the data. If not, it creates a new profile.

### `profilePush` — Update the current user's profile

Use `profilePush` to update properties on the already-identified user. This does **not** create or switch profiles.

```javascript
CleverTap.profilePush({
  'Plan': 'Enterprise',
  'Company': 'Acme Corp',
  'Score': 42,
});
```

### When to use which

| Scenario | Method |
|---|---|
| User logs in | `CleverTap.onUserLogin({...})` |
| Update a field on current user | `CleverTap.profilePush({...})` |
| User signs up (new account) | `CleverTap.onUserLogin({...})` |
| Anonymous user becomes known | `CleverTap.onUserLogin({...})` |

---

## Push Notifications

Push requires per-platform native setup plus a JavaScript listener for handling notification events.

### Android (FCM)

#### 1. Add Firebase to your Android project

Follow the standard Firebase setup: add `google-services.json` to `android/app/` and apply the Google Services plugin in your Gradle files.

```groovy
// android/build.gradle (project-level)
buildscript {
    dependencies {
        classpath 'com.google.gms:google-services:4.4.0'
    }
}
```

```groovy
// android/app/build.gradle
apply plugin: 'com.google.gms.google-services'

dependencies {
    implementation 'com.google.firebase:firebase-messaging:23.4.0'
}
```

#### 2. Create a push notification channel (Android 8+)

```kotlin
// android/app/src/main/java/com/yourapp/MainApplication.kt
import android.app.NotificationManager
import com.clevertap.android.sdk.CleverTapAPI

class MainApplication : Application() {
    override fun onCreate() {
        CleverTapAPI.createNotificationChannel(
            this,
            "generic",                  // channel ID
            "General Notifications",    // channel name
            "App notifications",        // channel description
            NotificationManager.IMPORTANCE_HIGH,
            true                        // show badge
        )
        ActivityLifecycleCallback.register(this)
        super.onCreate()
    }
}
```

### iOS (APNs)

#### 1. Enable Push in Xcode

- Open your project in Xcode via `ios/YourApp.xcworkspace`
- Go to **Signing & Capabilities** > **+ Capability** > **Push Notifications**
- Also add **Background Modes** and check **Remote notifications**

#### 2. Register for push in `AppDelegate`

```swift
// ios/YourApp/AppDelegate.swift
import UserNotifications

extension AppDelegate: UNUserNotificationCenterDelegate {
    func application(
        _ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
    ) {
        CleverTap.sharedInstance()?.setPushToken(deviceToken)
    }
}

// Call this during didFinishLaunchingWithOptions:
func registerForPush(_ application: UIApplication) {
    UNUserNotificationCenter.current().delegate = self
    UNUserNotificationCenter.current().requestAuthorization(
        options: [.alert, .badge, .sound]
    ) { granted, error in
        if granted {
            DispatchQueue.main.async {
                application.registerForRemoteNotifications()
            }
        }
    }
}
```

### JavaScript — Listen for push events

```javascript
import CleverTap from 'clevertap-react-native';
import { useEffect } from 'react';

function App() {
    useEffect(() => {
        // Fired when a push notification is received
        CleverTap.addListener(
            CleverTap.CleverTapPushNotificationClicked,
            (event) => {
                console.log('Push clicked:', event);
                // Handle deep link or navigation here
            }
        );

        return () => {
            CleverTap.removeListener(CleverTap.CleverTapPushNotificationClicked);
        };
    }, []);

    return (/* your app */);
}
```

#### Creating a local push notification (testing)

```javascript
CleverTap.createNotification({
  'title': 'Test Notification',
  'body': 'This is a test push from JS',
  'sound': 'default',
});
```

---

## Common Pitfalls

### 1. Hermes `libhermes.so` load errors

If you see a crash on Android with errors related to `libhermes.so`, the CleverTap native module may conflict with Hermes in older React Native versions. Disable Hermes in `android/app/build.gradle` if needed:

```groovy
project.ext.react = [
    enableHermes: false
]
```

> Hermes is generally stable with CleverTap SDK v5+ and React Native 0.70+. Only disable if you hit this specific crash.

### 2. Use `.xcworkspace` not `.xcodeproj`

After running `pod install`, always open:

```
ios/YourApp.xcworkspace   ← correct
ios/YourApp.xcodeproj     ← wrong (missing pod dependencies)
```

Opening `.xcodeproj` directly will cause build failures because CocoaPods-managed dependencies (including CleverTap) will not be linked.

### 3. Media3 libraries for audio/video inbox (v3.0.0+)

Starting with CleverTap React Native SDK v3.0.0, the App Inbox uses AndroidX Media3 for audio and video rendering. Add these dependencies to `android/app/build.gradle`:

```groovy
dependencies {
    implementation 'androidx.media3:media3-exoplayer:1.2.0'
    implementation 'androidx.media3:media3-exoplayer-hls:1.2.0'
    implementation 'androidx.media3:media3-ui:1.2.0'
}
```

Without these, inbox messages with audio/video will crash or fail to render.

### 4. Turbo Module support (New Architecture)

To enable CleverTap with React Native's New Architecture (Turbo Modules + Fabric), set the flag when building:

```bash
# iOS
RCT_NEW_ARCH_ENABLED=1 pod install --project-directory=ios
```

```groovy
// android/gradle.properties
newArchEnabled=true
```

> Turbo Module support requires CleverTap React Native SDK v2.0.0+ and React Native 0.71+.

### 5. Modular headers (iOS)

If you encounter `#import` errors after upgrading CocoaPods or enabling modular headers, switch to `@import` syntax in any custom native code that references CleverTap:

```swift
// WRONG — may fail with modular headers
#import <CleverTapSDK/CleverTap.h>

// CORRECT — works with modular headers
@import CleverTapSDK;
```

In your `Podfile`, you can also enable modular headers globally:

```ruby
use_modular_headers!
```

### 6. Debug logging

Enable verbose logging during development:

```javascript
// JavaScript — set debug level (0=off, 1=info, 2=debug, 3=verbose)
CleverTap.setDebugLevel(3);
```

```swift
// iOS (AppDelegate.swift)
CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)
```

```kotlin
// Android (MainApplication.kt)
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Check logs for entries prefixed with `[CleverTap]`. Remove debug logging before shipping to production.

---

## Quick Reference

```javascript
import CleverTap from 'clevertap-react-native';

// 1. No JS init needed — native side auto-initializes

// 2. Identify user on login
CleverTap.onUserLogin({
  'Name': 'Jane Doe',
  'Identity': 'user-12345',
  'Email': 'jane@example.com',
});

// 3. Track events
CleverTap.recordEvent('Feature Used', { 'Feature': 'Search' });

// 4. Charged event
CleverTap.recordChargedEvent(
  { 'Amount': 49.99, 'Currency': 'USD' },
  [{ 'Product Name': 'Widget', 'Quantity': 1 }]
);

// 5. Update profile
CleverTap.profilePush({ 'Plan': 'Premium' });

// 6. Listen for push
CleverTap.addListener(
  CleverTap.CleverTapPushNotificationClicked,
  (e) => console.log('Push:', e)
);

// 7. Debug (dev only)
CleverTap.setDebugLevel(3);
```
