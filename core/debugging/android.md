# CleverTap Android SDK — Debugging Reference

> Platform-specific debugging guide for the CleverTap Android SDK (Kotlin/Java).
> For cross-platform issues (identity, data constraints, dashboard lag), see `common.md`.

---

## Diagnostic Checklist

Run through these checks in order when something is not working:

1. **Manifest meta-data** — Are `CLEVERTAP_ACCOUNT_ID` and `CLEVERTAP_TOKEN` present inside the `<application>` tag in `AndroidManifest.xml`?
2. **Region** — If your account is not on the default US region, is `CLEVERTAP_REGION` set in the manifest?
3. **Application class** — Is your custom Application class registered in the manifest with `android:name=".MyApplication"`?
4. **Lifecycle callback order** — Is `ActivityLifecycleCallback.register(this)` called BEFORE `super.onCreate()` in your Application class?
5. **SDK version vs. Gradle version** — If using Gradle 8.0+ / AGP 8.0.0+, are you on CleverTap SDK v6.1.1 or later?
6. **Debug logging** — Is `CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)` called before `ActivityLifecycleCallback.register()`?
7. **Logcat output** — Filter Logcat for tag `CleverTap`. Are there any error or warning messages?
8. **Notification channel** — For Android 8+ (API 26+), is at least one notification channel created via `CleverTapAPI.createNotificationChannel()`?
9. **FCM setup** — Is `google-services.json` in the `app/` directory? Is the Google Services plugin applied?
10. **POST_NOTIFICATIONS permission** — For Android 13+ (API 33+), is `android.permission.POST_NOTIFICATIONS` declared in the manifest and requested at runtime?

---

## Symptoms, Causes, and Fixes

### Events not tracked

| Symptom | Likely Cause | Fix |
|---|---|---|
| No events recorded at all | Manifest `<meta-data>` for Account ID or Token is missing or outside the `<application>` tag | Add `CLEVERTAP_ACCOUNT_ID` and `CLEVERTAP_TOKEN` as `<meta-data>` inside `<application>` |
| No events — manifest looks correct | Application class not registered in manifest | Add `android:name=".MyApplication"` to the `<application>` tag |
| No events — Application class is registered | `ActivityLifecycleCallback.register(this)` called AFTER `super.onCreate()` | Move `register(this)` to before `super.onCreate()` |
| No events — lifecycle callback is before super | SDK not actually instantiated | Call `CleverTapAPI.getDefaultInstance(applicationContext)` after `super.onCreate()` to force initialization |
| Events fire once then stop | Activity or Fragment destroyed and instance not re-obtained | Get `CleverTapAPI.getDefaultInstance(context)` in each Activity or Fragment where you track events; do not cache across lifecycle boundaries |

### Push not showing

| Symptom | Likely Cause | Fix |
|---|---|---|
| Push not showing on Android 8+ | No notification channel created | Call `CleverTapAPI.createNotificationChannel()` in `Application.onCreate()` for API 26+ |
| Push not showing — channel exists | FCM token not registered with CleverTap | If using a custom `FirebaseMessagingService`, call `CleverTapAPI.getDefaultInstance(context)?.pushFcmRegistrationId(token, true)` in `onNewToken` |
| Push not showing — FCM token registered | `google-services.json` missing or from wrong Firebase project | Verify the file is in `android/app/` and matches the Firebase project linked to your CleverTap dashboard |
| Push not showing on Android 13+ | `POST_NOTIFICATIONS` permission not requested at runtime | Add `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` to manifest and request permission at runtime for API 33+ |
| Push shows but with wrong icon | `CLEVERTAP_NOTIFICATION_ICON` meta-data missing or pointing to nonexistent drawable | Add `<meta-data android:name="CLEVERTAP_NOTIFICATION_ICON" android:value="ic_notification"/>` and ensure `ic_notification` exists as a drawable resource |
| Push not showing — everything above checks out | Notification channel importance set to `IMPORTANCE_NONE` or channel disabled by user | Check channel importance is at least `IMPORTANCE_DEFAULT`; user may have manually disabled the channel in system settings |

### Build failures

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ClassNotFoundException` after upgrading to v6.1.1+ | Missing background sync meta-data | Add `<meta-data android:name="CLEVERTAP_BACKGROUND_SYNC" android:value="1"/>` inside `<application>` |
| Build fails with R8/ProGuard errors on Gradle 8.0+ | Old SDK version lacks R8 rules for full R8 mode | Upgrade to CleverTap SDK v6.1.1+ which includes required ProGuard/R8 rules |
| Build fails: "Cannot resolve symbol CleverTapAPI" | SDK dependency not added or Gradle sync not run | Add `implementation 'com.clevertap.android:clevertap-android-sdk:7.8.0'` to `app/build.gradle` and sync |
| Build fails: duplicate class errors | Multiple versions of CleverTap or conflicting transitive dependencies | Run `./gradlew app:dependencies` to find conflicts; use `exclude group:` or `force` to resolve |
| Runtime crash: `ClassCastException` on HashMap | Using Kotlin `mapOf()` instead of `HashMap<String, Any>()` | The SDK requires `HashMap<String, Any>`; replace `mapOf()` and `mutableMapOf()` with explicit `HashMap<String, Any>()` |

---

## Debug Tooling

### Enable debug logging

```kotlin
// Call BEFORE ActivityLifecycleCallback.register() for earliest output
CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)
```

Log levels:
- `OFF` — No output
- `INFO` — Basic lifecycle events
- `DEBUG` — Moderate detail
- `VERBOSE` — Full SDK activity including payloads and network calls

### Filter Logcat

In Android Studio Logcat, filter by tag:

```
CleverTap
```

Or use the command line:

```bash
adb logcat -s CleverTap:V
```

### Common Logcat messages

| Log Message | Meaning |
|---|---|
| `CleverTap: Account ID: ... Token: ...` | SDK initialized with these credentials |
| `CleverTap: Pushing event: ...` | Event queued for network send |
| `CleverTap: Flushing queue: N events` | Batch of events being sent to server |
| `CleverTap: Network response: 200` | Server accepted the request |
| `CleverTap: Error: Account ID is missing` | `CLEVERTAP_ACCOUNT_ID` not found in manifest |
| `CleverTap: FCM token registered` | FCM push token sent to CleverTap |
| `CleverTap: Notification channel created: ...` | Channel registered with the system |
| `CleverTap: Error: Activity lifecycle not registered` | `ActivityLifecycleCallback.register()` was not called or called too late |

### Verify FCM token

```kotlin
import com.google.firebase.messaging.FirebaseMessaging

FirebaseMessaging.getInstance().token.addOnCompleteListener { task ->
    if (task.isSuccessful) {
        val token = task.result
        Log.d("DEBUG", "FCM token: $token")
    } else {
        Log.e("DEBUG", "FCM token fetch failed", task.exception)
    }
}
```

### Verify CleverTap ID

```kotlin
val cleverTapId = CleverTapAPI.getDefaultInstance(context)?.cleverTapID
Log.d("DEBUG", "CleverTap ID: $cleverTapId")
```

### Inspect notification channels (Android 8+)

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val manager = getSystemService(NotificationManager::class.java)
    for (channel in manager.notificationChannels) {
        Log.d("DEBUG", "Channel: ${channel.id}, importance: ${channel.importance}")
    }
}
```

---

## Known Issues

### `ClassNotFoundException` on v6.1.1

| Issue | Details |
|---|---|
| Crash after upgrading to SDK v6.1.1+ | The SDK reorganized internal classes. If the manifest lacks `CLEVERTAP_BACKGROUND_SYNC`, the old class path is referenced and throws `ClassNotFoundException`. |
| Fix | Add `<meta-data android:name="CLEVERTAP_BACKGROUND_SYNC" android:value="1"/>` inside `<application>` in `AndroidManifest.xml`. |

### Google Advertising ID permission (Android 13+)

| Issue | Details |
|---|---|
| Ad ID returns all zeros (`00000000-...`) | On Android 13+, the `AD_ID` permission is required. Without it, the system returns a zeroed-out ID. |
| Fix | Add `<uses-permission android:name="com.google.android.gms.permission.AD_ID"/>` to the manifest. Also disclose Ad ID usage in the Google Play Data Safety section. |
| Not using GAID at all | If you do not need the Google Ad ID, remove `CLEVERTAP_USE_GOOGLE_AD_ID` meta-data from the manifest entirely to avoid unnecessary permission requirements. |

### HashMap type issues (Kotlin)

| Issue | Details |
|---|---|
| `ClassCastException` or compile error when passing profile/event data | Kotlin's `mapOf()` returns `LinkedHashMap` or `Map` interface, not `HashMap`. The CleverTap SDK explicitly expects `HashMap<String, Any>`. |
| Fix | Use `HashMap<String, Any>()` with explicit put operations, or use `hashMapOf("key" to value as Any)`. |

### R8/Minification with Gradle 8+ / AGP 8.0.0+

| Issue | Details |
|---|---|
| Runtime crashes after enabling minification | Gradle 8.0+ uses full R8 mode by default, which aggressively strips classes. SDK versions before v6.1.1 do not ship the required R8 keep rules. |
| Fix | Upgrade to CleverTap SDK v6.1.1+ which bundles correct ProGuard/R8 consumer rules. |

### Version-specific notes

| SDK Version | Notes |
|---|---|
| < 5.0.0 | Uses legacy ExoPlayer for App Inbox media. |
| 5.0.0+ | Migrated to AndroidX Media3 for Inbox audio/video. Add `media3-exoplayer` dependencies. |
| 6.1.1+ | Required for Gradle 8.0+ / AGP 8.0.0+. Includes R8 rules and background sync changes. |
| 7.0.0+ | Dropped support for API < 21. `minSdkVersion` must be 21+. |
| 7.8.0 | Current recommended version. |
