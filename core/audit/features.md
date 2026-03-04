# CleverTap SDK — Feature Map for Audit Mode

> Machine-readable reference of all CleverTap feature areas, SDK method signatures per platform,
> and grep patterns. Used by AI tools to scan a codebase and generate a usage report.

---

## 1. Feature Area Overview

| # | Feature Area | Description |
|---|---|---|
| 1 | **Identity** | User login (`onUserLogin`) and profile management (`profilePush`). Creates, merges, and updates user profiles using identity fields (Identity, Email, Phone). |
| 2 | **Events** | Custom event tracking (`pushEvent`/`recordEvent`) and transaction tracking via Charged events. Core analytics instrumentation. |
| 3 | **Push Notifications** | FCM (Android) and APNs (iOS) push setup, device token registration, notification channel creation, and notification handlers. |
| 4 | **App Inbox** | In-app message inbox UI that displays campaign messages. Requires initialization and provides callbacks for message actions. |
| 5 | **In-App Notifications** | Popups, interstitials, and banners triggered by campaigns. Rendered automatically by the SDK after init; custom callbacks available. |
| 6 | **Web Push** | Browser push notification registration and prompt display. Web SDK only. Requires service worker setup. |
| 7 | **Product Experiences** | Feature flags, A/B test variants, and remote config values fetched from the CleverTap dashboard at runtime. |
| 8 | **Geofencing** | Location-based triggers using geofence regions. Requires location permissions and explicit SDK setup. iOS and Android only. |
| 9 | **Web Native Display** | Banner and popup campaigns rendered natively in the browser. Web SDK only. Uses display unit callbacks. |

---

## 2. SDK Method Signatures Per Platform

### 2.1 Identity

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Login / Identify** | `clevertap.onUserLogin.push({` | `CleverTap.sharedInstance()?.onUserLogin(` | `clevertapDefaultInstance?.onUserLogin(` | `CleverTap.onUserLogin({` | `CleverTapPlugin.onUserLogin(` | N/A |
| **Update Profile** | `clevertap.profile.push({` | `CleverTap.sharedInstance()?.profilePush(` | `clevertapDefaultInstance?.pushProfile(` | `CleverTap.profilePush({` | `CleverTapPlugin.profilePush(` | `POST /1/upload` with `"type": "profile"` |

### 2.2 Events

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Track Event** | `clevertap.event.push("EventName"` | `CleverTap.sharedInstance()?.recordEvent(` | `clevertapDefaultInstance?.pushEvent(` | `CleverTap.recordEvent(` | `CleverTapPlugin.recordEvent(` | `POST /1/upload` with `"type": "event"` |
| **Charged Event** | `clevertap.event.push("Charged"` | `CleverTap.sharedInstance()?.recordChargedEvent(` | `clevertapDefaultInstance?.pushChargedEvent(` | `CleverTap.recordChargedEvent(` | `CleverTapPlugin.recordChargedEvent(` | `"evtName": "Charged"` |

### 2.3 Push Notifications

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Register Token** | `clevertap.notifications.push({` | `CleverTap.sharedInstance()?.setPushToken(` | `clevertapDefaultInstance?.pushFcmRegistrationId(` | Native-side only (see iOS/Android) | Native-side only (see iOS/Android) | N/A |
| **Handle Notification** | N/A | `CleverTap.sharedInstance()?.handleNotification(` | `CTFcmMessageHandler().createNotification(` | `CleverTap.addListener(CleverTap.CleverTapPushNotificationClicked` | `setCleverTapPushClickedPayloadReceivedHandler(` | N/A |
| **Create Channel** | N/A | N/A | `CleverTapAPI.createNotificationChannel(` | `CleverTapAPI.createNotificationChannel(` | `CleverTapAPI.createNotificationChannel(` | N/A |
| **Permission (Android 13+)** | N/A | N/A | `POST_NOTIFICATIONS` permission | N/A | `CleverTapPlugin.promptPushPrimer(` | N/A |

### 2.4 SDK Initialization

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Init** | `clevertap.init(` | `CleverTap.autoIntegrate()` or `CleverTap.sharedInstance()` | `CleverTapAPI.getDefaultInstance(` or extend `CleverTapApplication` | Native-side: `CleverTap.autoIntegrate()` (iOS) + manifest (Android) | Native-side: `CleverTap.autoIntegrate()` (iOS) + `ActivityLifecycleCallback.register(` (Android) | Constructor / headers setup |
| **Lifecycle Register** | N/A | N/A | `ActivityLifecycleCallback.register(` | `ActivityLifecycleCallback.register(` | `ActivityLifecycleCallback.register(` | N/A |
| **Debug Logging** | `clevertap.setLogLevel(3)` or `sessionStorage['WZRK_D']` | `CleverTap.setDebugLevel(` | `CleverTapAPI.setDebugLevel(` | `CleverTap.setDebugLevel(` | `CleverTapPlugin.setDebugLevel(` | N/A |
| **SPA Flag** | `clevertap.spa = true` | N/A | N/A | N/A | N/A | N/A |

### 2.5 App Inbox

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Show Inbox** | `clevertap.showInbox(` | `CleverTap.sharedInstance()?.showAppInbox(` | `clevertapDefaultInstance?.showAppInbox(` | `CleverTap.showInbox(` | `CleverTapPlugin.showInbox(` | N/A |
| **Initialize Inbox** | `clevertap.initializeInbox` | `CleverTap.sharedInstance()?.initializeInbox(` | `clevertapDefaultInstance?.initializeInbox(` | `CleverTap.initializeInbox(` | `CleverTapPlugin.initializeInbox(` | N/A |
| **Get Inbox Count** | `clevertap.getInboxMessageCount` | `CleverTap.sharedInstance()?.getInboxMessageCount(` | `clevertapDefaultInstance?.inboxMessageCount(` | `CleverTap.getInboxMessageCount(` | `CleverTapPlugin.getInboxMessageCount(` | N/A |
| **Get Unread Count** | `clevertap.getInboxMessageUnreadCount` | `CleverTap.sharedInstance()?.getInboxMessageUnreadCount(` | `clevertapDefaultInstance?.inboxMessageUnreadCount(` | `CleverTap.getInboxMessageUnreadCount(` | `CleverTapPlugin.getInboxMessageUnreadCount(` | N/A |

### 2.6 In-App Notifications

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Display** | Automatic after init | Automatic after init | Automatic after init | Automatic after init | Automatic after init | N/A |
| **Suspend/Resume** | N/A | `CleverTap.sharedInstance()?.suspendInAppNotifications()` | `clevertapDefaultInstance?.suspendInAppNotifications()` | `CleverTap.suspendInAppNotifications()` | `CleverTapPlugin.suspendInAppNotifications()` | N/A |
| **Discard** | N/A | `CleverTap.sharedInstance()?.discardInAppNotifications()` | `clevertapDefaultInstance?.discardInAppNotifications()` | `CleverTap.discardInAppNotifications()` | `CleverTapPlugin.discardInAppNotifications()` | N/A |
| **Callback** | N/A | `inAppNotificationDismissed(` | `onInAppNotificationDismissed(` | `CleverTap.addListener(CleverTap.CleverTapInAppNotificationDismissed` | `setCleverTapInAppNotificationDismissedHandler(` | N/A |

### 2.7 Web Push

| Operation | Web | iOS | Android | React Native | Flutter | Server API |
|---|---|---|---|---|---|---|
| **Register** | `clevertap.notifications.push({"titleText":` | N/A | N/A | N/A | N/A | N/A |
| **Service Worker** | `clevertap-sw.js` or custom service worker | N/A | N/A | N/A | N/A | N/A |

### 2.8 Product Experiences

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Feature Flag (Bool)** | `clevertap.getFeatureFlag(` | `CleverTap.sharedInstance()?.get(` (Bool overload) | `clevertapDefaultInstance?.getFeatureFlag(` | `CleverTap.getFeatureFlag(` | `CleverTapPlugin.getFeatureFlag(` | N/A |
| **Product Config** | `clevertap.productConfig` | `CleverTap.sharedInstance()?.productConfig()` | `clevertapDefaultInstance?.productConfig()` | `CleverTap.productConfigFetch(` | `CleverTapPlugin.productConfigFetch(` | N/A |

### 2.9 Geofencing

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Set Location** | N/A | `CleverTap.sharedInstance()?.setLocationForGeofences(` | `clevertapDefaultInstance?.setLocationForGeofences(` | N/A (native-side) | N/A (native-side) | N/A |
| **Init Geofence SDK** | N/A | `CTGeofenceAPI.shared.start(` | `CTGeofenceAPI.init(` | N/A | N/A | N/A |

### 2.10 Web Native Display

| Operation | Web | iOS (Swift) | Android (Kotlin) | React Native | Flutter (Dart) | Server API |
|---|---|---|---|---|---|---|
| **Record Viewed** | `clevertap.renderNotificationViewed(` | `CleverTap.sharedInstance()?.recordDisplayUnitViewed(` | `clevertapDefaultInstance?.pushDisplayUnitViewedEvent(` | `CleverTap.pushDisplayUnitViewedEvent(` | `CleverTapPlugin.pushDisplayUnitViewedEvent(` | N/A |
| **Record Clicked** | `clevertap.renderNotificationClicked(` | `CleverTap.sharedInstance()?.recordDisplayUnitClicked(` | `clevertapDefaultInstance?.pushDisplayUnitClickedEvent(` | `CleverTap.pushDisplayUnitClickedEvent(` | `CleverTapPlugin.pushDisplayUnitClickedEvent(` | N/A |
| **Get Units** | `clevertap.getDisplayUnits(` | `CleverTap.sharedInstance()?.getAllDisplayUnits(` | `clevertapDefaultInstance?.allDisplayUnits` | `CleverTap.getAllDisplayUnits(` | `CleverTapPlugin.getAllDisplayUnits(` | N/A |

---

## 3. Search Patterns (Regex)

Each pattern is designed for `grep -rE` or equivalent ripgrep usage. Patterns cover all supported platforms in a single expression.

```
# -------------------------------------------------------------------
# Identity
# -------------------------------------------------------------------
# Login / identify user
IDENTITY_LOGIN: /onUserLogin/

# Profile update
IDENTITY_PROFILE: /profilePush|profile\.push|pushProfile/

# Combined identity
IDENTITY_ALL: /onUserLogin|profilePush|profile\.push\(\{|pushProfile/

# -------------------------------------------------------------------
# Events
# -------------------------------------------------------------------
# Custom event tracking
EVENTS_TRACK: /recordEvent|pushEvent|event\.push\(/

# Charged event (transactions)
EVENTS_CHARGED: /recordChargedEvent|pushChargedEvent|event\.push\(\s*["']Charged["']/

# Combined events
EVENTS_ALL: /recordEvent|pushEvent|event\.push\(|recordChargedEvent|pushChargedEvent/

# -------------------------------------------------------------------
# Push Notifications
# -------------------------------------------------------------------
# Token registration
PUSH_TOKEN: /setPushToken|pushFcmRegistrationId|registerForRemoteNotifications/

# Notification handling
PUSH_HANDLER: /handleNotification|CTFcmMessageHandler|FcmMessageListenerService|CleverTapPushNotificationClicked|PushClickedPayloadReceived/

# Notification channel (Android 8+)
PUSH_CHANNEL: /createNotificationChannel/

# Push permission (Android 13+)
PUSH_PERMISSION: /POST_NOTIFICATIONS|promptPushPrimer/

# Combined push
PUSH_ALL: /setPushToken|pushFcmRegistrationId|registerForRemoteNotifications|handleNotification|CTFcmMessageHandler|FcmMessageListenerService|createNotificationChannel|CleverTapPushNotificationClicked|PushClickedPayloadReceived|POST_NOTIFICATIONS|promptPushPrimer/

# -------------------------------------------------------------------
# SDK Initialization
# -------------------------------------------------------------------
INIT_ALL: /clevertap\.init\(|CleverTap\.autoIntegrate|CleverTap\.sharedInstance|CleverTapAPI\.getDefaultInstance|CleverTapApplication|ActivityLifecycleCallback\.register|clevertap\.account\.push/

# Debug logging
DEBUG_LOGGING: /setLogLevel|setDebugLevel|WZRK_D/

# SPA mode
SPA_MODE: /clevertap\.spa\s*=/

# -------------------------------------------------------------------
# App Inbox
# -------------------------------------------------------------------
INBOX_ALL: /showInbox|showAppInbox|initializeInbox|getInboxMessageCount|inboxMessageCount|getInboxMessageUnreadCount|inboxMessageUnreadCount/

# -------------------------------------------------------------------
# In-App Notifications
# -------------------------------------------------------------------
INAPP_ALL: /suspendInAppNotifications|discardInAppNotifications|resumeInAppNotifications|InAppNotificationDismissed|inAppNotificationDismissed|onInAppNotificationDismissed|InAppNotificationButtonTapped/

# -------------------------------------------------------------------
# Web Push
# -------------------------------------------------------------------
WEBPUSH_ALL: /notifications\.push\(\s*\{|titleText|bodyText|okButtonText|rejectButtonText|okButtonColor|clevertap-sw\.js/

# -------------------------------------------------------------------
# Product Experiences (Feature Flags, Product Config)
# -------------------------------------------------------------------
PRODUCTEXP_ALL: /getFeatureFlag|productConfig|productConfigFetch|productConfigActivate/

# -------------------------------------------------------------------
# Geofencing
# -------------------------------------------------------------------
GEOFENCE_ALL: /setLocationForGeofences|CTGeofenceAPI|CTGeofence|clevertap-geofence/

# -------------------------------------------------------------------
# Web Native Display (Display Units)
# -------------------------------------------------------------------
DISPLAY_ALL: /renderNotificationViewed|renderNotificationClicked|recordDisplayUnit|pushDisplayUnit|getDisplayUnits|getAllDisplayUnits|allDisplayUnits|DisplayUnitViewed|DisplayUnitClicked/

# -------------------------------------------------------------------
# SDK Package / Dependency Detection
# -------------------------------------------------------------------
DEPENDENCY_WEB: /clevertap-web-sdk/
DEPENDENCY_IOS: /CleverTap-iOS-SDK|clevertap-ios-sdk|CleverTapSDK/
DEPENDENCY_ANDROID: /com\.clevertap\.android:clevertap-android-sdk/
DEPENDENCY_RN: /clevertap-react-native/
DEPENDENCY_FLUTTER: /clevertap_plugin/
DEPENDENCY_ALL: /clevertap-web-sdk|CleverTap-iOS-SDK|clevertap-ios-sdk|CleverTapSDK|com\.clevertap\.android:clevertap-android-sdk|clevertap-react-native|clevertap_plugin/

# -------------------------------------------------------------------
# Server API
# -------------------------------------------------------------------
SERVER_API_AUTH: /X-CleverTap-Account-Id|X-CleverTap-Passcode/
SERVER_API_UPLOAD: /\/1\/upload/
SERVER_API_ALL: /X-CleverTap-Account-Id|X-CleverTap-Passcode|\/1\/upload|api\.clevertap\.com/
```

---

## 4. SDK Call Ordering Rules

These rules define the expected ordering of SDK calls. Violations indicate integration issues.

### 4.1 Initialization Must Come First

```
RULE: SDK init MUST precede all other SDK calls.

Web:      clevertap.init() must be called before event.push(), profile.push(), onUserLogin.push()
          Exception: CDN stub pattern queues calls automatically.
iOS:      CleverTap.autoIntegrate() or CleverTap.sharedInstance() must be in
          application(_:didFinishLaunchingWithOptions:), before any recordEvent/onUserLogin.
Android:  ActivityLifecycleCallback.register(this) must be called BEFORE super.onCreate()
          in the Application class. CleverTapAPI.getDefaultInstance() after super.
RN:       Native init (autoIntegrate on iOS, manifest + register on Android) handles this.
          No JS init call needed.
Flutter:  Same as RN — native init on both platforms. Dart calls come after.
```

### 4.2 Identity Before Profile Updates

```
RULE: onUserLogin MUST be called before profilePush in any login flow.

Rationale: profilePush updates the CURRENT user's profile. If onUserLogin has not been
called, profilePush updates the anonymous profile, which may be lost on next login.

Correct:
  onUserLogin({ Identity: "user-123", Email: "..." })
  profilePush({ Plan: "Premium" })

Wrong:
  profilePush({ Plan: "Premium" })    // Updates anonymous profile
  onUserLogin({ Identity: "user-123" }) // Creates/switches to real profile — Plan update is lost
```

### 4.3 Push Token After Init

```
RULE: Push token registration should happen after SDK initialization.

iOS:      setPushToken() in didRegisterForRemoteNotificationsWithDeviceToken,
          which fires after autoIntegrate() in didFinishLaunchingWithOptions.
Android:  pushFcmRegistrationId() after CleverTapAPI.getDefaultInstance().
          Or use FcmMessageListenerService (auto-handled).
```

### 4.4 Event Tracking After Init

```
RULE: Event tracking calls should only happen after SDK initialization completes.

Web:      clevertap.init() must resolve before event.push() fires.
          Exception: CDN stub pattern (arrays queue calls before SDK loads).
iOS:      recordEvent() after autoIntegrate() or sharedInstance() returns.
Android:  pushEvent() after getDefaultInstance() returns non-null.
```

### 4.5 Notification Channel Before Push (Android)

```
RULE: On Android 8+ (API 26+), a notification channel MUST be created
      before push notifications can be displayed.

createNotificationChannel() should be called in Application.onCreate(),
BEFORE any push messages arrive. Without it, pushes are silently dropped.
```

### 4.6 Debug Logging Before Init

```
RULE: Debug log level should be set BEFORE SDK initialization for earliest output.

Web:      clevertap.setLogLevel(3) before clevertap.init()
iOS:      CleverTap.setDebugLevel() BEFORE CleverTap.autoIntegrate()
Android:  CleverTapAPI.setDebugLevel() BEFORE ActivityLifecycleCallback.register()
```

---

## 5. Audit Report Template

When generating an audit report, use the following template. Replace bracketed placeholders with actual values.

```markdown
## CleverTap SDK Usage Map

**Platform:** [detected platform, e.g., "React Native (iOS + Android)"]
**SDK Version:** [version from package.json / Podfile / build.gradle, or "unknown"]
**SDK Package:** [e.g., "clevertap-react-native@2.3.0"]
**Files Scanned:** [total count of files searched]
**Scan Date:** [current date]

---

### Initialization
- Init method: [e.g., "CleverTap.autoIntegrate() in AppDelegate.swift:12"]
- Lifecycle registration: [e.g., "ActivityLifecycleCallback.register() in MainApplication.kt:8"]
- Debug logging: [enabled/disabled, location if found]
- SPA mode: [true/false/N/A, location if found]
- Status: [OK / ISSUE]
- Issues: [list any ordering or missing issues, e.g., "register() called AFTER super.onCreate()"]

### Identity
- onUserLogin: [count] call site(s)
  - [file:line — brief context for each]
- profilePush: [count] call site(s)
  - [file:line — brief context for each]
- Status: [OK / ISSUE]
- Issues: [e.g., "profilePush called before onUserLogin in LoginScreen.kt:45"]

### Events
- Custom events: [count] unique event names across [count] file(s)
  - Event names: [list of unique event name strings found]
- Charged events: [count] call site(s)
  - [file:line — brief context for each]
- Status: [OK / ISSUE]
- Issues: [e.g., "event.push called before clevertap.init in analytics.js:3"]

### Push Notifications
- Token registration: [count] call site(s)
  - [file:line — brief context for each]
- Notification channels: [count] channel(s) created
  - [channel IDs if detectable]
- Notification handlers: [count] handler(s)
  - [file:line — brief context for each]
- Push permission (Android 13+): [present/missing]
- Status: [OK / ISSUE]
- Issues: [e.g., "No notification channel created — pushes will fail on Android 8+"]

### App Inbox
- Initialization: [count] call site(s) or "Not integrated"
- Show inbox: [count] call site(s)
- Status: [OK / NOT INTEGRATED]
- Issues: [any issues found]

### In-App Notifications
- Callback handlers: [count] or "None (using defaults)"
- Suspend/resume: [count] call site(s) or "None"
- Status: [OK / NOT INTEGRATED]
- Issues: [any issues found]

### Web Push
- Registration: [count] call site(s) or "Not integrated" or "N/A (mobile app)"
- Service worker: [found/not found]
- Status: [OK / NOT INTEGRATED / N/A]
- Issues: [any issues found]

### Product Experiences
- Feature flags: [count] call site(s) or "Not integrated"
- Product config: [count] call site(s) or "Not integrated"
- Status: [OK / NOT INTEGRATED]
- Issues: [any issues found]

### Geofencing
- Location setup: [count] call site(s) or "Not integrated"
- Geofence SDK init: [count] call site(s) or "Not integrated"
- Status: [OK / NOT INTEGRATED]
- Issues: [any issues found]

### Native Display
- Display unit callbacks: [count] call site(s) or "Not integrated"
- Status: [OK / NOT INTEGRATED]
- Issues: [any issues found]

---

### Summary

| Feature Area | Status |
|---|---|
| Initialization | [OK / ISSUE] |
| Identity | [OK / ISSUE] |
| Events | [OK / ISSUE] |
| Push Notifications | [OK / ISSUE / NOT INTEGRATED] |
| App Inbox | [OK / NOT INTEGRATED] |
| In-App Notifications | [OK / NOT INTEGRATED] |
| Web Push | [OK / NOT INTEGRATED / N/A] |
| Product Experiences | [OK / NOT INTEGRATED] |
| Geofencing | [OK / NOT INTEGRATED] |
| Native Display | [OK / NOT INTEGRATED] |

**Integrated:** [comma-separated list of active feature areas]
**Not integrated:** [comma-separated list of unused feature areas]
**Issues found:** [total count]
**Critical issues:** [count of ordering / missing init issues]
```

---

## 6. Platform Detection Signals

Use these file patterns to auto-detect the target platform before scanning.

| Signal File | Platform | SDK Package |
|---|---|---|
| `package.json` with `clevertap-web-sdk` | Web | `clevertap-web-sdk` |
| `package.json` with `clevertap-react-native` | React Native | `clevertap-react-native` |
| `pubspec.yaml` with `clevertap_plugin` | Flutter | `clevertap_plugin` |
| `Podfile` with `CleverTap-iOS-SDK` | iOS | `CleverTap-iOS-SDK` |
| `build.gradle` with `clevertap-android-sdk` | Android | `com.clevertap.android:clevertap-android-sdk` |
| `requirements.txt` or `pyproject.toml` with `requests` + Server API headers in code | Python (Server) | REST API |
| `package.json` (Node) + Server API headers in code | Node.js (Server) | REST API |
| `pom.xml` or `build.gradle` (no Android manifest) + Server API headers in code | Java (Server) | REST API |

### Multi-platform projects

React Native and Flutter projects contain both iOS and Android native code. When auditing:
1. Check the Dart/JS layer for cross-platform calls.
2. Also scan `ios/` and `android/` directories for native-side setup (init, push token, channels).
3. Report native-side findings separately if they differ from the cross-platform layer.
