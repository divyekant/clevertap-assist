# CleverTap Web SDK — Debugging Reference

> Platform-specific debugging guide for the CleverTap Web SDK.
> For cross-platform issues (identity, data constraints, dashboard lag), see `common.md`.

---

## Diagnostic Checklist

Run through these checks in order when something is not working:

1. **Protocol check** — Is the page served over HTTPS? The SDK will not work on `file://` or plain `http://` in production.
2. **Initialization order** — Is `clevertap.init(accountId, region)` called before any `event.push` or `profile.push` calls? (CDN stub pattern is exempt because it queues calls.)
3. **Account ID and Region** — Are they correct? A wrong region causes silent failures. Match against your dashboard URL (e.g., `in1.dashboard.clevertap.com` means region `in1`).
4. **SPA flag** — If using React, Vue, Angular, or Next.js, is `clevertap.spa = true` set immediately after `init()`?
5. **Content Security Policy (CSP)** — Does your CSP allow connections to `*.clevertap.com` and `d2r1yp2w7bby2u.cloudfront.net`?
6. **Ad blockers / privacy extensions** — Disable uBlock Origin, Brave Shield, Privacy Badger, etc. and retest. These commonly block CleverTap network requests.
7. **Console errors** — Open DevTools console and filter for `[CT]` or `[CleverTap]`. Any errors there?
8. **Network tab** — Filter the Network tab for `clevertap` or `wzrk`. Are requests being sent? What status codes come back?
9. **Multiple instances** — Are you accidentally initializing the SDK more than once without using the named-instance pattern?
10. **Debug logging** — Enable `clevertap.setLogLevel(3)` and check console output for detailed SDK activity.

---

## Symptoms, Causes, and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Events not appearing in dashboard | SDK not initialized, wrong Account ID/Region, or ad blocker intercepting requests | Verify `init()` call, confirm Account ID and Region match dashboard, disable ad blockers and retest |
| Events not in dashboard but no console errors | CSP blocking outbound requests silently | Add `*.clevertap.com` and `d2r1yp2w7bby2u.cloudfront.net` to your CSP `connect-src` and `script-src` directives |
| `onUserLogin` not merging profiles | Missing or wrong identity field (`Identity`, `Email`, or `Phone`) in the `Site` object, or calling `profile.push` instead of `onUserLogin.push` | Verify at least one identity field is present inside the `"Site"` key; use `onUserLogin.push` for login, not `profile.push` |
| Duplicate page views in SPA | `clevertap.spa = true` not set | Add `clevertap.spa = true` immediately after `init()` |
| `[CT]` errors in console: "Account ID not set" | `init()` was not called, or called with empty/undefined Account ID | Verify your init call passes a valid string Account ID |
| `[CT]` errors in console: "Region not set" | Region parameter missing or empty | Pass the correct region string to `init()` or set `clevertap.region` |
| `[CT]` errors in console: "Invalid event structure" | Event name is empty, null, or exceeds 512 characters, or property values are invalid types | Validate event name is a non-empty string under 512 chars; property values must be string, number, boolean, or Date |
| Events fire on first page but not on subsequent SPA navigations | SDK loses track of route changes without `spa = true` | Set `clevertap.spa = true` |
| `clevertap is not defined` in console | SDK script not loaded yet, or import statement missing | For npm: verify `import clevertap from 'clevertap-web-sdk'`; for CDN: verify the script tag is present and loads before your code |
| Next.js: `window is not defined` | Importing SDK at top level in a server-rendered file | Use dynamic `import()` inside `useEffect`, or guard with `typeof window !== 'undefined'` |
| Next.js: `document is not defined` | Same SSR issue | Same fix: dynamic import in `useEffect` or `'use client'` directive |
| Network requests show 401 | Wrong Account ID or mismatched region | Verify credentials against Settings > Project in the CleverTap dashboard |
| Network requests show CORS errors | Proxy or CDN misconfiguration stripping CORS headers | Ensure your proxy forwards CleverTap CORS headers; do not proxy CleverTap endpoints unless required |
| `sessionStorage['WZRK_D']` returns undefined | SDK has not initialized yet, or running in a context without sessionStorage (e.g., SSR, incognito with storage blocked) | Verify SDK is initialized client-side; check browser privacy settings |
| Profile updates work but events do not | Calling `event.push` before `init()` completes (non-stub pattern) | Move event tracking after init, or use the stub object pattern for early queuing |

---

## Debug Tooling

### Enable debug logging

```javascript
// Method 1: Programmatic (0=off, 1=error, 2=warn, 3=debug)
clevertap.setLogLevel(3);

// Method 2: sessionStorage flag (works even before SDK loads)
sessionStorage['WZRK_D'] = '';
```

After enabling, open the browser console and filter for `[CT]` to see all SDK log output.

### Inspect network requests

1. Open DevTools > **Network** tab.
2. Filter by `clevertap` or `wzrk`.
3. Look for POST requests to endpoints like `https://<region>.clevertap-prod.com/a?...`.
4. Check:
   - **Status code** — 200 means accepted; 401 means auth failure; no request means SDK did not fire.
   - **Request payload** — Verify the JSON body contains your event name and properties.
   - **Response body** — Look for `"status": "success"` or error details.

### Inspect stored data

```javascript
// View the CleverTap GUID (anonymous device ID)
console.log(clevertap.getCleverTapID());

// View queued events (if using stub pattern before SDK loads)
console.log(clevertap.event);
console.log(clevertap.profile);
```

### Console prefix reference

All CleverTap Web SDK log entries are prefixed with `[CT]`. Common log patterns:

| Console Message Pattern | Meaning |
|---|---|
| `[CT] Account ID: ...` | SDK initialized with this Account ID |
| `[CT] Region: ...` | SDK using this region endpoint |
| `[CT] Event pushed: ...` | Event was queued for sending |
| `[CT] Error: ...` | SDK encountered an error; read the message for details |
| `[CT] Profile updated` | Profile push was accepted locally |

---

## Known Issues

### Legacy SDK vs. modern SDK

| Issue | Details |
|---|---|
| CDN `clevertap.min.js` from `static.clevertap.com` | This is the deprecated legacy SDK. It lacks SPA support, modern event APIs, and receives no updates. Migrate to `clevertap-web-sdk` via npm. |
| Stub object pattern not compatible with npm import | If you import via npm, do not also set up the CDN stub object. Use one pattern or the other. |
| Legacy SDK does not support `setLogLevel()` | Use `sessionStorage['WZRK_D'] = ''` instead for debug output on the legacy SDK. |

### SPA routing issues

| Issue | Details |
|---|---|
| Duplicate page events on route change | Without `clevertap.spa = true`, every route change may trigger a new "page view" event. Set the flag immediately after `init()`. |
| Hash-based routing (`#/path`) | The SDK tracks hash changes only when `spa = true`. Without it, hash changes are ignored. |
| Next.js App Router | Use a `'use client'` provider component for initialization. Server components cannot access the SDK. |

### Browser-specific issues

| Issue | Details |
|---|---|
| Safari ITP (Intelligent Tracking Prevention) | Safari limits third-party cookie lifetimes. CleverTap uses first-party cookies, but if you proxy through a different domain, cookies may be capped at 7 days. Use a first-party subdomain for the CleverTap endpoint if needed. |
| Brave browser blocks by default | Brave's built-in ad blocker blocks CleverTap requests. Users must whitelist your domain or disable shields. This is not a bug in the SDK. |
| Firefox Enhanced Tracking Protection | Strict mode may block CleverTap. Standard mode typically allows it. |

### Version-specific notes

| SDK Version | Notes |
|---|---|
| < 1.4.0 | No support for `setLogLevel()`. Use `sessionStorage['WZRK_D']` only. |
| 1.4.0+ | Added `setLogLevel()`, improved SPA handling, better error messages in console. |
| 1.6.0+ | Added `setOffline()` for GDPR pause/resume. |
| 1.8.0+ | Named instances for multi-account support via `clevertap.init(id, region, name)`. |
