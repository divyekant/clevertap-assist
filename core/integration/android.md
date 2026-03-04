# CleverTap Android SDK — Integration Reference

> Self-contained integration guide for the CleverTap Android SDK.
> Covers Gradle setup, manifest configuration, event tracking, user identity, and push notifications.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Android API level** | 19+ (Android 4.4 KitKat) |
| **AndroidX** | Required — legacy Support Library is not supported |
| **Kotlin** | 1.5+ recommended (Java also supported) |
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

Add the CleverTap SDK and Firebase Messaging dependencies to your **app-level** `build.gradle`:

```groovy
dependencies {
    implementation 'com.clevertap.android:clevertap-android-sdk:7.8.0'
    implementation 'com.google.firebase:firebase-messaging:24.0.0'
}
```

Ensure the Google services plugin is applied at the bottom of your app-level `build.gradle`:

```groovy
apply plugin: 'com.google.gms.google-services'
```

And add the classpath to your **project-level** `build.gradle`:

```groovy
buildscript {
    dependencies {
        classpath 'com.google.gms:google-services:4.4.2'
    }
}
```

> **Note:** If you use Gradle version catalogs (`libs.versions.toml`), add the equivalent entries there instead.

---

## Configuration

Add CleverTap credentials to your `AndroidManifest.xml` inside the `<application>` tag:

```xml
<application ...>

    <!-- Required -->
    <meta-data
        android:name="CLEVERTAP_ACCOUNT_ID"
        android:value="YOUR_ACCOUNT_ID"/>
    <meta-data
        android:name="CLEVERTAP_TOKEN"
        android:value="YOUR_TOKEN"/>

    <!-- Optional: set your region (omit if us1) -->
    <meta-data
        android:name="CLEVERTAP_REGION"
        android:value="in1"/>

    <!-- Optional: enable Google Advertising ID tracking -->
    <meta-data
        android:name="CLEVERTAP_USE_GOOGLE_AD_ID"
        android:value="1"/>

</application>
```

| Meta-data Key | Required | Description |
|---|---|---|
| `CLEVERTAP_ACCOUNT_ID` | Yes | Your project Account ID |
| `CLEVERTAP_TOKEN` | Yes | Your project Token |
| `CLEVERTAP_REGION` | No | Data center region code. Defaults to `us1` if omitted |
| `CLEVERTAP_USE_GOOGLE_AD_ID` | No | Set to `1` to collect Google Advertising ID. Requires GDPR disclosure and `AD_ID` permission on Android 13+ |

---

## Initialization

### Option 1: Extend `CleverTapApplication` (simplest)

```kotlin
// MyApplication.kt
import com.clevertap.android.sdk.CleverTapApplication

class MyApplication : CleverTapApplication() {
    // That's it — the SDK initializes automatically
}
```

Register it in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApplication"
    ...>
```

### Option 2: Custom Application class

Use this when you already have a custom Application class or need more control:

```kotlin
// MyApplication.kt
import android.app.Application
import com.clevertap.android.sdk.ActivityLifecycleCallback
import com.clevertap.android.sdk.CleverTapAPI

class MyApplication : Application() {
    override fun onCreate() {
        // Must be called before super.onCreate()
        ActivityLifecycleCallback.register(this)
        super.onCreate()

        // Get the default instance (initializes the SDK)
        val clevertapDefaultInstance = CleverTapAPI.getDefaultInstance(applicationContext)
    }
}
```

> **Important:** `ActivityLifecycleCallback.register(this)` must be called **before** `super.onCreate()`. This is required for the SDK to track app lifecycle events correctly.

### Debug logging

Enable verbose logging during development:

```kotlin
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Call this before `ActivityLifecycleCallback.register(this)` for the earliest log output. Check Logcat for entries tagged `CleverTap`. Remove before shipping to production.

---

## Event Tracking

Get the default instance in any Activity or Fragment:

```kotlin
val clevertapDefaultInstance = CleverTapAPI.getDefaultInstance(context)
```

### Basic event (no properties)

```kotlin
clevertapDefaultInstance?.pushEvent("Product Viewed")
```

### Event with properties

```kotlin
val props = HashMap<String, Any>()
props["Product Name"] = "Wireless Headphones"
props["Category"] = "Electronics"
props["Price"] = 79.99
props["On Sale"] = true

clevertapDefaultInstance?.pushEvent("Product Viewed", props)
```

### Charged event (transactions)

```kotlin
val chargeDetails = HashMap<String, Any>()
chargeDetails["Amount"] = 119.98
chargeDetails["Currency"] = "USD"
chargeDetails["Charged ID"] = "order-12345"

val item1 = HashMap<String, Any>()
item1["Product Name"] = "Wireless Headphones"
item1["Category"] = "Electronics"
item1["Price"] = 79.99
item1["Quantity"] = 1

val item2 = HashMap<String, Any>()
item2["Product Name"] = "USB-C Cable"
item2["Category"] = "Accessories"
item2["Price"] = 39.99
item2["Quantity"] = 1

val items = ArrayList<HashMap<String, Any>>()
items.add(item1)
items.add(item2)

clevertapDefaultInstance?.pushChargedEvent(chargeDetails, items)
```

### Property value rules

- Property names: strings, max 120 characters
- Property values: String, Number, Boolean
- Max 256 custom event properties per event
- Event names: strings, max 512 characters
- Avoid special characters in event names; use alphanumeric and spaces

---

## User Identity

### `onUserLogin` — Identify a user on login or signup

Use `onUserLogin` when a user logs in or creates an account. This creates a new profile or merges with an existing one based on the identity fields.

```kotlin
val profile = HashMap<String, Any>()
profile["Name"] = "Jane Doe"
profile["Identity"] = "user-12345"          // Your internal user ID
profile["Email"] = "jane@example.com"
profile["Phone"] = "+14155551234"
profile["Gender"] = "F"                      // M or F
profile["DOB"] = "1990-06-15"                // yyyy-MM-dd format

// Custom properties
profile["Plan"] = "Premium"
profile["Score"] = 42

clevertapDefaultInstance?.onUserLogin(profile)
```

**Identity fields** that trigger profile merge:
- `Identity` — your internal user/account ID
- `Email` — email address
- `Phone` — phone number with country code

Include at least one identity field. If a profile with that identity exists, the SDK merges the data. If not, it creates a new profile.

### `pushProfile` — Update the current user's profile

Use `pushProfile` to update properties on the already-identified user. This does **not** create or switch profiles.

```kotlin
val profileUpdate = HashMap<String, Any>()
profileUpdate["Plan"] = "Enterprise"
profileUpdate["Company"] = "Acme Corp"
profileUpdate["Score"] = 85

clevertapDefaultInstance?.pushProfile(profileUpdate)
```

### Standard profile keys

| Key | Type | Notes |
|---|---|---|
| `Name` | String | Full name |
| `Email` | String | Email address (identity field) |
| `Phone` | String | With country code, e.g. `+14155551234` (identity field) |
| `Identity` | String | Your internal user ID (identity field) |
| `Gender` | String | `M` or `F` |
| `DOB` | String | `yyyy-MM-dd` format |

### When to use which

| Scenario | Method |
|---|---|
| User logs in | `onUserLogin(profile)` |
| User signs up (new account) | `onUserLogin(profile)` |
| Update a field on current user | `pushProfile(profile)` |
| Anonymous user becomes known | `onUserLogin(profile)` |

---

## Push Notifications

### 1. FCM setup

Place your `google-services.json` file (downloaded from the Firebase console) in the `app/` directory.

Ensure Firebase Messaging is in your dependencies (already covered in Installation):

```groovy
implementation 'com.google.firebase:firebase-messaging:24.0.0'
```

### 2. Notification channels (required for Android 8+)

Android 8.0 (API 26) and above **require** a notification channel. Create it in your Application class:

```kotlin
import android.app.NotificationManager
import android.os.Build
import com.clevertap.android.sdk.CleverTapAPI

class MyApplication : Application() {
    override fun onCreate() {
        ActivityLifecycleCallback.register(this)
        super.onCreate()

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            CleverTapAPI.createNotificationChannel(
                applicationContext,
                "default_channel",     // channel ID
                "General",             // channel name
                "App notifications",   // channel description
                NotificationManager.IMPORTANCE_DEFAULT,
                true                   // show badge
            )
        }
    }
}
```

> **Critical:** If you skip this step, push notifications will **not appear** on Android 8+ devices. This is the most common push notification issue.

### 3. Pass the FCM token to CleverTap

#### Option A: CleverTap's `CTFcmMessageHandler` (simplest)

Add the service to your `AndroidManifest.xml`:

```xml
<service
    android:name="com.clevertap.android.sdk.pushnotification.fcm.FcmMessageListenerService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT"/>
    </intent-filter>
</service>
```

This handles token registration and message display automatically.

#### Option B: Custom `FirebaseMessagingService`

Use this when you have your own Firebase messaging logic:

```kotlin
import com.clevertap.android.sdk.CleverTapAPI
import com.clevertap.android.sdk.pushnotification.fcm.CTFcmMessageHandler
import com.google.firebase.messaging.FirebaseMessagingService
import com.google.firebase.messaging.RemoteMessage

class MyFcmService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        super.onNewToken(token)
        // Pass token to CleverTap
        CleverTapAPI.getDefaultInstance(applicationContext)
            ?.pushFcmRegistrationId(token, true)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        super.onMessageReceived(message)
        // Let CleverTap handle its own messages
        CTFcmMessageHandler().createNotification(applicationContext, message)
    }
}
```

Register it in `AndroidManifest.xml`:

```xml
<service
    android:name=".MyFcmService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT"/>
    </intent-filter>
</service>
```

### 4. Push notification permissions (Android 13+)

Android 13 (API 33) requires runtime permission for notifications:

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
```

```kotlin
// In your Activity (API 33+)
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    requestPermissions(
        arrayOf(android.Manifest.permission.POST_NOTIFICATIONS),
        REQUEST_CODE_NOTIFICATIONS
    )
}
```

---

## Common Pitfalls

### 1. R8/minification with Gradle 8.0+ / AGP 8.0.0+

Gradle 8.0+ with Android Gradle Plugin 8.0.0+ uses full R8 mode by default, which can strip SDK classes. You **must** use CleverTap SDK v6.1.1 or later, which includes the required ProGuard/R8 rules.

```groovy
// WRONG: old SDK version with new Gradle — will crash at runtime
implementation 'com.clevertap.android:clevertap-android-sdk:5.2.0'

// CORRECT: use v6.1.1+ with Gradle 8.0+ / AGP 8.0.0+
implementation 'com.clevertap.android:clevertap-android-sdk:7.8.0'
```

### 2. Google Ad ID requires GDPR disclosure and `AD_ID` permission

If you set `CLEVERTAP_USE_GOOGLE_AD_ID` to `1`, you must:
- Disclose Ad ID usage in your Google Play Data Safety section
- On Android 13+, declare the `AD_ID` permission:

```xml
<uses-permission android:name="com.google.android.gms.permission.AD_ID"/>
```

Without the permission on Android 13+, the Ad ID will silently return `00000000-0000-0000-0000-000000000000`.

### 3. `ClassNotFoundException` on v6.1.1 upgrade

After upgrading to SDK v6.1.1+, you may see `ClassNotFoundException` if the manifest entry is missing. Add the following inside your `<application>` tag:

```xml
<meta-data
    android:name="CLEVERTAP_BACKGROUND_SYNC"
    android:value="1"/>
```

### 4. Notification channels MUST be created for Android 8+

Without a notification channel, push notifications are silently dropped on Android 8+ (API 26+). This is the number one support issue for push.

```kotlin
// WRONG: no notification channel — pushes silently fail on Android 8+
// (no channel creation code at all)

// CORRECT: create at least one channel in Application.onCreate()
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    CleverTapAPI.createNotificationChannel(
        applicationContext,
        "default_channel",
        "General",
        "App notifications",
        NotificationManager.IMPORTANCE_DEFAULT,
        true
    )
}
```

### 5. Kotlin: use `HashMap<String, Any>()` not `mapOf()`

The CleverTap SDK requires `HashMap<String, Any>`. Kotlin's `mapOf()` returns an immutable `Map` which is not compatible.

```kotlin
// WRONG: mapOf returns immutable Map, not HashMap
val profile = mapOf("Name" to "Jane", "Email" to "jane@example.com")
clevertapDefaultInstance?.onUserLogin(profile) // Compile error or ClassCastException

// CORRECT: use HashMap explicitly
val profile = HashMap<String, Any>()
profile["Name"] = "Jane"
profile["Email"] = "jane@example.com"
clevertapDefaultInstance?.onUserLogin(profile)
```

### 6. `ActivityLifecycleCallback.register()` must come before `super.onCreate()`

If you call `register()` after `super.onCreate()`, the SDK will miss the initial Activity creation and may not track sessions correctly.

```kotlin
// WRONG: register after super — SDK misses lifecycle events
override fun onCreate() {
    super.onCreate()
    ActivityLifecycleCallback.register(this) // Too late
}

// CORRECT: register before super
override fun onCreate() {
    ActivityLifecycleCallback.register(this)
    super.onCreate()
}
```

### 7. Debug logging

Enable verbose logging during development:

```kotlin
// Call before ActivityLifecycleCallback.register() for earliest output
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Check Logcat with tag filter `CleverTap`. Remove before shipping to production.

---

## Quick Reference

```kotlin
// === build.gradle (app) ===
// implementation 'com.clevertap.android:clevertap-android-sdk:7.8.0'
// implementation 'com.google.firebase:firebase-messaging:24.0.0'

// === Application class ===
import android.app.Application
import android.app.NotificationManager
import android.os.Build
import com.clevertap.android.sdk.ActivityLifecycleCallback
import com.clevertap.android.sdk.CleverTapAPI

class MyApplication : Application() {
    override fun onCreate() {
        CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE) // dev only
        ActivityLifecycleCallback.register(this)
        super.onCreate()

        // Create notification channel (required for Android 8+)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            CleverTapAPI.createNotificationChannel(
                applicationContext, "default_channel", "General",
                "App notifications", NotificationManager.IMPORTANCE_DEFAULT, true
            )
        }
    }
}

// === In any Activity or Fragment ===
val cleverTap = CleverTapAPI.getDefaultInstance(context)

// 1. Identify user on login
val profile = HashMap<String, Any>()
profile["Name"] = "Jane Doe"
profile["Identity"] = "user-12345"
profile["Email"] = "jane@example.com"
cleverTap?.onUserLogin(profile)

// 2. Track events
cleverTap?.pushEvent("Feature Used", hashMapOf("Feature" to "Search" as Any))

// 3. Update profile
val update = HashMap<String, Any>()
update["Plan"] = "Premium"
cleverTap?.pushProfile(update)

// 4. Pass FCM token
cleverTap?.pushFcmRegistrationId(token, true)
```
