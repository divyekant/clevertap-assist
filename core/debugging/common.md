# CleverTap Common Debugging Reference — Cross-Platform

> Check this file FIRST when a developer reports a CleverTap issue.
> These gotchas apply to ALL platforms (Web, iOS, Android, React Native, Flutter, Server).
> Walk through each numbered item as a diagnostic checklist.

---

## 1. Init-Before-Track Rule

The SDK must be fully initialized before any event, profile, or identity call. Events fired before initialization are silently dropped or queued without delivery guarantees depending on the platform.

**Symptom:** Events appear in local logs but never reach the CleverTap dashboard.
**Cause:** `pushEvent` / `event.push` is called before the SDK has finished initialization.
**Fix:** Always call the SDK init method in the app entry point (Application class, AppDelegate, root component mount) before any user-interaction code runs.

```
WRONG order:
  trackEvent("Page Loaded")   <-- SDK not ready yet
  initCleverTap(accountId)

CORRECT order:
  initCleverTap(accountId)
  trackEvent("Page Loaded")
```

Platform-specific init timing:

| Platform | Where to init |
|---|---|
| Web | Top of entry JS file, or `useEffect([], ...)` in React |
| iOS | `application(_:didFinishLaunchingWithOptions:)` in AppDelegate |
| Android | `Application.onCreate()`, after `ActivityLifecycleCallback.register(this)` |
| React Native | Native-side Application/AppDelegate, before JS bridge loads |
| Flutter | `main()` before `runApp()`, or in native Application/AppDelegate |
| Server | At module/service startup, before any API call function |

---

## 2. Credential Validation

Incorrect or mismatched credentials are the most common cause of SDK silence — no errors thrown, just no data reaching the dashboard.

### Account ID format

- Alphanumeric with dashes, e.g. `W67-774-7Z5Z`
- Found in CleverTap dashboard: Settings > Project
- Case-sensitive: `w67-774-7z5z` will NOT work if the actual ID uses uppercase

### Region must match the account's data center

| Region Code | Dashboard URL contains |
|---|---|
| `us1` | `us1.dashboard.clevertap.com` |
| `in1` | `in1.dashboard.clevertap.com` |
| `sg1` | `sg1.dashboard.clevertap.com` |
| `aps3` | `aps3.dashboard.clevertap.com` |
| `mec1` | `mec1.dashboard.clevertap.com` |

**Symptom:** SDK initializes without errors, events fire in debug logs, but nothing appears in the dashboard.
**Cause:** Region code does not match the data center where the account was provisioned.
**Fix:** Check the dashboard URL and set the region code accordingly.

### Token vs Passcode — different credentials for different contexts

| Credential | Where to find | Used in |
|---|---|---|
| **Token** | Settings > Project > Tokens > Account Token | Mobile SDKs (iOS, Android, React Native, Flutter), Web SDK |
| **Passcode** | Settings > Project > Tokens > Account Passcode | Server-side REST API calls |

**Symptom:** `401 Unauthorized` or silent failures in API calls.
**Cause:** Using the Passcode in a mobile SDK (expects Token), or using the Token in a REST API call (expects Passcode).
**Fix:** Verify the credential type matches the context:
- Mobile/Web SDK: use **Token**
- Server-side REST API: use **Passcode** (passed as `X-CleverTap-Passcode` header)

---

## 3. Protocol Requirement

All CleverTap SDKs communicate over HTTPS. The `file://` protocol will not work.

**Symptom:** Web SDK reports no errors but no network requests appear in the browser Network tab. Or mobile SDK requests fail with SSL/network errors.
**Cause:** Page opened via `file://` protocol (double-clicking an HTML file), or HTTP used where HTTPS is required.
**Fix:** Use a local development server for web development.

```bash
# Quick local servers for web testing
npx serve .
# or
python3 -m http.server 8000
# or
php -S localhost:8000
```

Why `file://` fails completely:
- Cookies cannot be set (SDK uses cookies for session/identity tracking on web)
- XHR/Fetch requests are blocked by browser security policy
- CORS preflight requests cannot succeed

For mobile SDKs, ensure App Transport Security (iOS) or network security config (Android) permits HTTPS connections to CleverTap domains:
- `*.clevertap-prod.com`
- `*.wzrk.com`

---

## 4. Duplicate Initialization

The CleverTap SDK uses a singleton pattern. Only ONE instance should exist per app. Multiple init calls cause state conflicts, duplicate events, and unpredictable identity behavior.

**Symptom:** Duplicate events appearing in the dashboard (exactly 2x or 3x the expected count). Or profile properties flapping between values.
**Cause:** SDK initialized more than once — common in SPAs with hot module replacement, or when init is placed inside a component that re-renders.
**Fix:** Initialize exactly once in the app entry point and reference that instance globally.

Common scenarios that cause duplicate init:

| Scenario | Problem | Solution |
|---|---|---|
| React `useEffect` without `[]` | Init runs on every render | Add empty dependency array: `useEffect(() => { ... }, [])` |
| Hot Module Replacement (HMR) | Init re-runs on code change | Guard with `if (!window.__ctInitialized)` flag |
| Multiple Activity/Fragment init (Android) | Each Activity calls `getDefaultInstance` with new config | Init once in `Application.onCreate()`, call only `getDefaultInstance(context)` elsewhere |
| Multiple `CleverTap.autoIntegrate()` (iOS) | Called in both AppDelegate and SceneDelegate | Call only in `application(_:didFinishLaunchingWithOptions:)` |

### Pattern: singleton reference

```
// Pseudocode — applicable to any platform
APP ENTRY POINT:
  cleverTapInstance = initCleverTap(accountId, region)
  store cleverTapInstance globally (singleton, DI, provide/inject, etc.)

ANYWHERE ELSE:
  ct = getGlobalCleverTapInstance()
  ct.pushEvent(...)
```

---

## 5. Event Naming and Properties

CleverTap enforces strict rules on event names and property values. Violations are silently dropped — no error is thrown.

### Reserved prefixes

- `wzrk_` — Reserved by CleverTap for internal system events and properties
- Any event or property name starting with `wzrk_` will be rejected

**Symptom:** Event fires in debug logs but does not appear in the dashboard.
**Cause:** Event name or property key uses the `wzrk_` prefix.
**Fix:** Rename the event or property to avoid the reserved prefix.

### Character limits

| Element | Limit |
|---|---|
| Event name | 512 characters max |
| Property key | 120 characters max |
| Property value (string) | 512 characters max |
| Properties per event | 256 max |

**Symptom:** Event appears in the dashboard but specific properties are missing.
**Cause:** Property key or value exceeds character limit, or too many properties on a single event.
**Fix:** Shorten names/values or reduce the number of properties.

### Property type constraints

Allowed property value types:

| Type | Example | Notes |
|---|---|---|
| String | `"Electronics"` | Max 512 characters |
| Number | `79.99` | Integer or float |
| Boolean | `true` | |
| Date | `new Date()` or platform equivalent | |

**NOT allowed:**
- Nested objects / dictionaries / maps
- Arrays (except in the `Items` array of a `Charged` event)
- Null / nil / undefined values

**Symptom:** Event appears but a specific property is missing or shows unexpected value.
**Cause:** Property value is a nested object, array, or null.
**Fix:** Flatten nested structures into separate top-level properties.

```
WRONG:
  pushEvent("Purchase", {
    "product": {
      "name": "Headphones",    // nested object — will be dropped
      "price": 79.99
    }
  })

CORRECT:
  pushEvent("Purchase", {
    "Product Name": "Headphones",
    "Product Price": 79.99
  })
```

### Event name best practices

- Use Title Case with spaces: `"Product Viewed"`, not `"product_viewed"` or `"productViewed"`
- Avoid special characters beyond alphanumeric and spaces
- Be consistent: `"Product Viewed"` and `"product viewed"` are treated as DIFFERENT events

---

## 6. Identity Resolution

Identity is the most misunderstood part of CleverTap. Using the wrong method at the wrong time causes profile fragmentation, data attached to anonymous profiles, or duplicate profiles.

### `onUserLogin` vs `profilePush`/`profile.push`

| Method | Purpose | When to call | What it does |
|---|---|---|---|
| `onUserLogin` | Identify a user | Login, signup, anonymous-to-known transition | Creates new profile OR merges with existing profile based on identity fields (`Identity`, `Email`, `Phone`) |
| `profilePush` / `profile.push` | Update the current profile | After the user is already identified | Updates properties on the current profile — does NOT create or switch profiles |

### Critical sequence rules

1. **Always call `onUserLogin` before `profilePush` for a new session.**
   Calling `profilePush` before `onUserLogin` attaches data to the anonymous profile, not the identified user. That data is lost if the anonymous profile is not later merged.

2. **Call `onUserLogin` only at login/signup moments.**
   Calling it on every app open (even for already-logged-in users) can cause unnecessary profile merges or new profile creation.

3. **Include at least one identity field in `onUserLogin`.**
   Identity fields are: `Identity` (your internal user ID), `Email`, `Phone`. Without one, the call creates a new anonymous profile.

**Symptom:** Dashboard shows duplicate profiles for the same user, or profile properties are missing.
**Cause:** `profilePush` called before `onUserLogin`, or `onUserLogin` called without identity fields.
**Fix:** Ensure the call sequence is: init -> `onUserLogin` (with identity field) -> then `profilePush` for updates.

### Multi-device identity

When the same user logs in on multiple devices, `onUserLogin` with the same identity field value merges the profiles. The result is a single profile in the dashboard with activity from all devices. This is the correct behavior — do NOT try to prevent it or work around it.

**Symptom:** User on Device A sees push notifications intended for Device B.
**Cause:** This is expected behavior after profile merge. Both device tokens are associated with the single merged profile.
**Fix:** No fix needed — this is by design. If this is undesirable, review your identity field strategy with your CleverTap account team.

### Common identity mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Calling `onUserLogin` on every app open | May create new profiles if identity fields differ slightly | Call only at actual login/signup; use `profilePush` for returning users |
| Using email as the sole identity field when users change emails | Old and new profiles will not merge | Use a stable internal user ID in the `Identity` field |
| Calling `profilePush` with identity fields | Does NOT trigger a profile merge | Use `onUserLogin` when you need to identify or merge profiles |
| Passing an empty identity field value | Creates a new anonymous profile | Always validate that identity values are non-empty before calling |

---

## 7. Network Issues

SDK communication can be blocked by network infrastructure without clear error messages.

### Corporate proxies

**Symptom:** SDK works on personal WiFi but fails on corporate network. No events reach the dashboard from the office.
**Cause:** Corporate proxy blocking outbound requests to CleverTap domains.
**Fix:** Ask IT to whitelist these domains:
- `*.clevertap-prod.com`
- `*.wzrk.com`
- `clevertap.com`

### VPN / Cloudflare WARP interference

**Symptom:** Intermittent SDK failures, timeouts, or partial data loss when VPN/WARP is active.
**Cause:** VPN or WARP routing interferes with SDK requests, especially if the VPN routes traffic through a different geographic region than the CleverTap data center.
**Fix:**
- Test with VPN/WARP disabled to confirm this is the cause
- If confirmed, add CleverTap domains to the VPN/WARP split-tunnel exclusion list
- For development, consider excluding CleverTap endpoints from the tunnel

### Content Security Policy (Web only)

**Symptom:** Browser console shows CSP violation errors mentioning CleverTap domains. SDK loads but cannot send data.
**Cause:** The website's Content-Security-Policy header does not permit connections to CleverTap endpoints.
**Fix:** Add the following to your CSP header:

```
connect-src 'self' https://*.clevertap-prod.com https://*.wzrk.com;
script-src 'self' https://d2r1yp2w7bby2u.cloudfront.net;
img-src 'self' https://*.clevertap-prod.com https://*.wzrk.com;
```

### Firewall rules (Server-side API)

**Symptom:** Server-side API calls return connection timeout or connection refused errors.
**Cause:** Outbound firewall rules block HTTPS requests to CleverTap API endpoints.
**Fix:** Allow outbound HTTPS (port 443) to:
- `us1.api.clevertap.com` (or your region's API endpoint)
- `in1.api.clevertap.com`
- `sg1.api.clevertap.com`
- `aps3.api.clevertap.com`
- `mec1.api.clevertap.com`

### Quick network diagnostic

```bash
# Test connectivity to CleverTap API (replace region as needed)
curl -v https://us1.api.clevertap.com/1/upload \
  -H "X-CleverTap-Account-Id: YOUR_ACCOUNT_ID" \
  -H "X-CleverTap-Passcode: YOUR_PASSCODE" \
  -H "Content-Type: application/json" \
  -d '{"d":[]}'

# Expected: HTTP 200 with JSON response
# If timeout/refused: network issue
# If 401: credential issue (see section 2)
```

---

## 8. Debug Logging Quick Reference

Enable verbose logging to see exactly what the SDK is sending and receiving. This is the single most useful step when diagnosing any issue.

| Platform | Method | Log output location |
|---|---|---|
| **Web** | `clevertap.setLogLevel(3)` | Browser DevTools Console |
| **Web** (alt) | `sessionStorage['WZRK_D'] = ''` | Browser DevTools Console (works before SDK loads) |
| **iOS** | `CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)` | Xcode Console |
| **Android** | `CleverTapAPI.setDebugLevel(CleverTapAPI.LogLevel.VERBOSE)` | Logcat (filter tag: `CleverTap`) |
| **React Native** | Native-side logging (use iOS or Android method above) | Xcode Console / Logcat |
| **Flutter** | `CleverTapPlugin.setDebugLevel(3)` + native-side logging | Xcode Console / Logcat / `flutter logs` |
| **Server** | Check HTTP response body for error JSON | Your server logs / stdout |

### What to look for in debug logs

1. **Initialization confirmation** — Look for a log entry confirming the Account ID and region were accepted.
2. **Event payloads** — Each event should appear as a JSON payload in the logs before being sent.
3. **Network responses** — Look for HTTP status codes. `200` means success. `401` means bad credentials. `400` means malformed payload.
4. **Error messages** — The SDK logs specific reasons for dropped events (invalid property types, reserved names, etc.).

**IMPORTANT:** Remove or disable debug logging before shipping to production. Debug logs may expose account credentials and user data.

---

## Diagnostic Walkthrough

When an issue is reported, walk through these steps in order:

```
Step 1: Is debug logging enabled?
  NO  -> Enable it (see section 8), reproduce the issue, check logs
  YES -> Continue

Step 2: Does the SDK initialize successfully?
  NO  -> Check credentials (section 2) and protocol (section 3)
  YES -> Continue

Step 3: Are events appearing in debug logs?
  NO  -> Check init-before-track order (section 1)
  YES -> Continue

Step 4: Are events reaching the network (visible in Network tab / proxy)?
  NO  -> Check network issues (section 7) and protocol (section 3)
  YES -> Continue

Step 5: Does the dashboard show the events?
  NO  -> Check event naming (section 5), credential/region mismatch (section 2)
  YES -> Continue

Step 6: Are events duplicated?
  YES -> Check duplicate initialization (section 4)
  NO  -> Continue

Step 7: Are profile properties wrong or missing?
  YES -> Check identity resolution (section 6) and property types (section 5)
  NO  -> Issue may be platform-specific — check the platform debugging file
```

---

## See Also

- Platform-specific debugging references (when they exist) will cover issues unique to each SDK
- Integration references: `core/integration/web.md`, `core/integration/ios.md`, `core/integration/android.md`, `core/integration/react-native.md`, `core/integration/flutter.md`
- Server-side references: `core/integration/server/node.md`, `core/integration/server/python.md`, `core/integration/server/java.md`
