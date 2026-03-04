# CleverTap SDK Skill — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a multi-platform, multi-tool skill that helps developers integrate, debug, and audit CleverTap SDKs.

**Architecture:** Monorepo with `core/` (tool-agnostic markdown knowledge) + `adapters/` (thin wrappers for Claude Code, Codex, Cursor). Three modes: integration, debugging, audit.

**Tech Stack:** Markdown reference files, Claude Code SKILL.md, Codex AGENTS.md, Cursor .cursorrules. Evals via skill-creator plugin.

---

### Task 1: Scaffold Project Directory Structure

**Files:**
- Create: `core/integration/.gitkeep`
- Create: `core/integration/server/.gitkeep`
- Create: `core/debugging/.gitkeep`
- Create: `core/audit/.gitkeep`
- Create: `adapters/claude-code/.gitkeep`
- Create: `adapters/codex/.gitkeep`
- Create: `adapters/cursor/.gitkeep`
- Create: `evals/.gitkeep`
- Create: `.gitignore`

**Step 1: Create all directories**

```bash
cd /Users/divyekant/Projects/clevertap-sdk
mkdir -p core/integration/server core/debugging core/audit adapters/claude-code adapters/codex adapters/cursor evals
```

**Step 2: Create .gitignore**

```
.DS_Store
*.swp
*.swo
*~
```

**Step 3: Commit**

```bash
git add -A && git commit -m "scaffold: create project directory structure"
```

---

### Task 2: Core Integration — Web SDK

**Files:**
- Create: `core/integration/web.md`

**Step 1: Write web integration reference**

This file covers: vanilla JS, React, Next.js, Vue, Angular. Contains:

- **Prerequisites**: Node 14+, or any browser for CDN
- **Installation**: npm (`clevertap-web-sdk`), yarn, CDN (legacy/deprecated)
- **Initialization**: stub object pattern with `account.push`, region config, `spa = true` for SPAs
- **Event tracking**: `clevertap.event.push(name, props)`
- **User identity**: `clevertap.onUserLogin.push({Site: {...}})`, `clevertap.profile.push({Site: {...}})`
- **Framework variants**:
  - React: import in App component, init in useEffect
  - Next.js: dynamic import with `ssr: false`, init in `_app.js`
  - Vue: plugin pattern or init in main.js
  - Angular: init in app.module.ts constructor
- **Common pitfalls**: file:// protocol doesn't work, init before track, legacy SDK deprecated, SPA flag, debug logging via `clevertap.setLogLevel(3)` or `sessionStorage['WZRK_D']`

**Step 2: Verify file is self-contained**

Read through and confirm a developer with zero CleverTap knowledge could integrate from this file alone.

**Step 3: Commit**

```bash
git add core/integration/web.md && git commit -m "docs: add web SDK integration reference"
```

---

### Task 3: Core Integration — iOS SDK

**Files:**
- Create: `core/integration/ios.md`

**Step 1: Write iOS integration reference**

Contains:
- **Prerequisites**: iOS 9.0+, Xcode 12+
- **Installation**: CocoaPods (`pod "CleverTap-iOS-SDK"`), SPM (`https://github.com/CleverTap/clevertap-ios-sdk`), manual
- **Initialization**: `CleverTap.autoIntegrate()` in AppDelegate, credentials via `Info.plist` keys (`CleverTapAccountID`, `CleverTapToken`)
- **Event tracking**: `CleverTap.sharedInstance()?.recordEvent("name")`
- **User identity**: `CleverTap.sharedInstance()?.onUserLogin(dict)`
- **Push notifications**: APNs setup, request authorization, register for remote notifications, delegate methods
- **Common pitfalls**: IDFV behavior with multiple apps (disable via `CleverTapDisableIDFV`), swift bridging headers for SDK <3.1.4, member/creator roles lack token access

**Step 2: Commit**

```bash
git add core/integration/ios.md && git commit -m "docs: add iOS SDK integration reference"
```

---

### Task 4: Core Integration — Android SDK

**Files:**
- Create: `core/integration/android.md`

**Step 1: Write Android integration reference**

Contains:
- **Prerequisites**: Android API 19+, AndroidX
- **Installation**: Gradle (`com.clevertap.android:clevertap-android-sdk:7.8.0` + firebase-messaging)
- **Initialization**: manifest meta-data (`CLEVERTAP_ACCOUNT_ID`, `CLEVERTAP_TOKEN`), extend `CleverTapApplication` or register `ActivityLifecycleCallback`
- **Event tracking**: `clevertapDefaultInstance.pushEvent("name")`
- **User identity**: `clevertapDefaultInstance.onUserLogin(hashMap)`
- **Push notifications**: FCM integration, `google-services.json`, notification channels for Android 8+
- **Common pitfalls**: R8/minification with Gradle 8.0+/AGP 8.0.0+ needs SDK v6.1.1+, Google Ad ID requires GDPR disclosure + `AD_ID` permission on Android 13+, `ClassNotFoundException` during background sync on v6.1.1 upgrade

**Step 2: Commit**

```bash
git add core/integration/android.md && git commit -m "docs: add Android SDK integration reference"
```

---

### Task 5: Core Integration — React Native SDK

**Files:**
- Create: `core/integration/react-native.md`

**Step 1: Write React Native integration reference**

Contains:
- **Prerequisites**: React Native 0.60+
- **Installation**: `npm install --save clevertap-react-native`
- **Initialization**: dual setup — Android manifest + `CleverTapRnAPI.setInitialUri()` in MainActivity, iOS Info.plist + `CleverTap.autoIntegrate()` + `CleverTapReactManager.sharedInstance()` in AppDelegate
- **Event tracking** (JS): `CleverTap.recordEvent('name', props)`
- **User identity** (JS): `CleverTap.onUserLogin({...})`, `CleverTap.profilePush({...})`
- **Common pitfalls**: Hermes `libhermes.so` load errors, `.xcworkspace` not `.xcodeproj` after pod install, Media3 libraries for inbox audio/video, Turbo Module support via `RCT_NEW_ARCH_ENABLED=1`

**Step 2: Commit**

```bash
git add core/integration/react-native.md && git commit -m "docs: add React Native SDK integration reference"
```

---

### Task 6: Core Integration — Flutter SDK

**Files:**
- Create: `core/integration/flutter.md`

**Step 1: Write Flutter integration reference**

Contains:
- **Prerequisites**: Flutter 2.0+, Dart 2.12+
- **Installation**: `clevertap_plugin: 3.6.0` in pubspec.yaml
- **Initialization**: triple setup — Android manifest + `ActivityLifecycleCallback.register()` BEFORE `super.onCreate()`, iOS Info.plist + autoIntegrate + `CleverTapPlugin.sharedInstance()`, Web script tag + `CleverTapPlugin.init(accountId, region, targetDomain)` in Dart
- **Event tracking** (Dart): `CleverTapPlugin.recordEvent("name", {})`
- **User identity** (Dart): `CleverTapPlugin.onUserLogin(map)`, `CleverTapPlugin.profilePush(map)`
- **Common pitfalls**: `ActivityLifecycleCallback.register()` MUST be before `super.onCreate()`, AndroidX Media3 vs ExoPlayer for inbox, declarative Gradle plugins for Flutter 3.16+, Info.plist values must be strings

**Step 2: Commit**

```bash
git add core/integration/flutter.md && git commit -m "docs: add Flutter SDK integration reference"
```

---

### Task 7: Core Integration — Server SDKs (Node, Python, Java)

**Files:**
- Create: `core/integration/server/node.md`
- Create: `core/integration/server/python.md`
- Create: `core/integration/server/java.md`

**Step 1: Research server SDK patterns**

Use WebFetch to pull from:
- `developer.clevertap.com/docs` — server API / backend SDK docs
- CleverTap REST API reference for event upload, profile update endpoints

**Step 2: Write Node.js server reference**

Contains: npm package, init with account ID + passcode, upload events API, profile push API, error handling patterns.

**Step 3: Write Python server reference**

Contains: pip package or REST API usage, same patterns as Node adapted to Python.

**Step 4: Write Java server reference**

Contains: Maven/Gradle dependency, init, REST API client pattern.

**Step 5: Commit**

```bash
git add core/integration/server/ && git commit -m "docs: add server SDK integration references (Node, Python, Java)"
```

---

### Task 8: Core Debugging — Common Cross-Platform

**Files:**
- Create: `core/debugging/common.md`

**Step 1: Write common debugging reference**

Contains:
- **Init-before-track rule**: SDK must be initialized before any event/profile call, events may be silently dropped otherwise
- **Credential validation**: account ID format, region must match account, token vs passcode distinction
- **Protocol requirement**: HTTPS required, file:// protocol blocks all SDK communication
- **Duplicate init**: singleton pattern enforcement, multiple inits cause state conflicts
- **Event naming**: reserved prefixes, character limits, property type constraints
- **Identity resolution**: `onUserLogin` vs `profilePush` ordering, profile merge behavior, multi-device identity
- **Network issues**: corporate proxies, WARP/VPN interference, CSP headers blocking SDK endpoints
- **Debug logging**: how to enable verbose logging per platform (summary table)

**Step 2: Commit**

```bash
git add core/debugging/common.md && git commit -m "docs: add common cross-platform debugging reference"
```

---

### Task 9: Core Debugging — Platform-Specific Files

**Files:**
- Create: `core/debugging/web.md`
- Create: `core/debugging/ios.md`
- Create: `core/debugging/android.md`
- Create: `core/debugging/react-native.md`
- Create: `core/debugging/flutter.md`
- Create: `core/debugging/server.md`

**Step 1: Write web debugging reference**

Symptoms → causes:
- "Events not in dashboard" → check protocol, init, account ID, region, CSP headers, ad blockers
- "onUserLogin not merging profiles" → check identity field, duplicate profiles
- "[CT] errors in console" → decode common error codes
- Debug tools: `clevertap.setLogLevel(3)`, `sessionStorage['WZRK_D']`, network tab inspection

**Step 2: Write iOS debugging reference**

Symptoms → causes:
- "No events recorded" → check autoIntegrate called, Info.plist keys correct, debug level set
- "Push not received" → APNs cert, entitlements, foreground delegate
- Debug tools: `CleverTap.setDebugLevel(CleverTapLogLevel.debug.rawValue)`, Xcode console filter `[CleverTap]`

**Step 3: Write Android debugging reference**

Similar structure: logcat filters, FCM token issues, notification channels, R8 minification.

**Step 4: Write React Native debugging reference**

Bridge-specific issues, native module linking, Hermes compatibility.

**Step 5: Write Flutter debugging reference**

Platform channel issues, lifecycle callback ordering.

**Step 6: Write server debugging reference**

REST API errors, auth failures, rate limits, payload validation.

**Step 7: Commit**

```bash
git add core/debugging/ && git commit -m "docs: add platform-specific debugging references"
```

---

### Task 10: Core Audit — Feature Map

**Files:**
- Create: `core/audit/features.md`

**Step 1: Write feature audit reference**

For each feature area, list the SDK method signatures to search for per platform:

| Feature | Web | iOS | Android | RN | Flutter |
|---|---|---|---|---|---|
| Identity | `onUserLogin.push`, `profile.push` | `onUserLogin`, `profilePush` | `onUserLogin`, `pushProfile` | `CleverTap.onUserLogin`, `CleverTap.profilePush` | `CleverTapPlugin.onUserLogin`, `CleverTapPlugin.profilePush` |
| Events | `event.push` | `recordEvent` | `pushEvent` | `CleverTap.recordEvent` | `CleverTapPlugin.recordEvent` |
| Push | `notifications.push` | `registerForPush`, `setPushToken` | `pushFcmRegistrationId` | `CleverTap.registerForPush` | `CleverTapPlugin.registerForPush` |
| App Inbox | `showInbox` | `showAppInbox` | `showAppInbox` | `CleverTap.showInbox` | `CleverTapPlugin.showInbox` |
| In-App | (automatic) | (automatic) | (automatic) | (automatic) | (automatic) |
| Web Push | `notifications.push` | N/A | N/A | N/A | N/A |
| Product Experiences | `getFeatureFlag` | `getFeatureFlag` | `getFeatureFlag` | `CleverTap.getFeatureFlag` | `CleverTapPlugin.getFeatureFlag` |
| Geofencing | N/A | `setLocationForGeofences` | `setLocationForGeofences` | N/A | N/A |
| Native Display | `renderNotificationViewed` | `recordDisplayUnit` | `recordDisplayUnit` | similar | similar |

Include:
- Search patterns (regex) per feature per platform
- Expected ordering rules (e.g., `onUserLogin` before `profilePush`)
- Report template format (the example from the design doc)

**Step 2: Commit**

```bash
git add core/audit/features.md && git commit -m "docs: add audit feature map with SDK method signatures"
```

---

### Task 11: Claude Code Adapter — SKILL.md

**Files:**
- Create: `adapters/claude-code/SKILL.md`

**Step 1: Write SKILL.md**

Frontmatter:
```yaml
---
name: clevertap-sdk
description: "Use when integrating CleverTap SDK into a project, debugging CleverTap SDK issues, or auditing CleverTap SDK usage. Triggers on: 'clevertap', 'add clevertap', 'integrate clevertap', 'clevertap not working', 'clevertap debug', 'clevertap audit', 'SDK events not showing'."
---
```

Body contains:
- **Mode detection**: parse user intent into integrate/debug/audit
- **Platform detection**: glob patterns to scan (`package.json`, `Podfile`, `build.gradle`, `pubspec.yaml`, etc.)
- **Integration workflow**: ask credentials → read `core/integration/<platform>.md` → generate code → verify
- **Debug workflow**: read `core/debugging/common.md` + `core/debugging/<platform>.md` → checklist → active scan → fix → verify
- **Audit workflow**: read `core/audit/features.md` → grep codebase for all SDK methods → generate report
- **Fallback**: WebFetch from `developer.clevertap.com` if reference files don't cover the case
- **Verification steps**: static checks (file exists, init order) + runtime guidance (what logs to look for)

**Step 2: Validate SKILL.md is under 500 lines**

Reference files are loaded on demand — SKILL.md stays lean.

**Step 3: Commit**

```bash
git add adapters/claude-code/SKILL.md && git commit -m "feat: add Claude Code adapter (SKILL.md)"
```

---

### Task 12: Codex Adapter — AGENTS.md

**Files:**
- Create: `adapters/codex/AGENTS.md`

**Step 1: Write AGENTS.md**

Same three-mode workflow as CC adapter, formatted for Codex agent conventions. References `core/` files as context. Codex-specific: uses `codex` sandbox modes, file read patterns.

**Step 2: Commit**

```bash
git add adapters/codex/AGENTS.md && git commit -m "feat: add Codex adapter (AGENTS.md)"
```

---

### Task 13: Cursor Adapter — .cursorrules

**Files:**
- Create: `adapters/cursor/.cursorrules`

**Step 1: Write .cursorrules**

Same workflow, Cursor format. References `core/` files. Cursor-specific: uses `@file` references, codebase context patterns.

**Step 2: Commit**

```bash
git add adapters/cursor/.cursorrules && git commit -m "feat: add Cursor adapter (.cursorrules)"
```

---

### Task 14: Evals via Skill-Creator

**Files:**
- Create: `evals/evals.json`

**Step 1: Write eval cases**

Test scenarios:
1. **Web integration** — "Add CleverTap to my React app" → should detect React, ask credentials, generate init + event code
2. **iOS integration** — "Integrate CleverTap in my Swift project" → should detect iOS, use CocoaPods/SPM, generate AppDelegate code
3. **Android integration** — "Add CleverTap to my Kotlin app" → should detect Android, modify manifest + gradle
4. **Debug: events missing** — "My CleverTap events aren't showing up on the dashboard" in a web project → should run checklist, check init order
5. **Debug: push not working** — "Push notifications not arriving" in iOS project → should check APNs setup, entitlements
6. **Audit** — "Show me how CleverTap is used in my app" → should scan codebase, produce usage map
7. **Unknown platform** — "Add CleverTap to my Rust backend" → should gracefully fallback to asking user + doc fetch
8. **Monorepo** — project has both `Podfile` and `build.gradle` → should ask which platform

**Step 2: Run evals using skill-creator**

```bash
# Use skill-creator to run and validate
```

**Step 3: Iterate on failures**

Fix reference files or SKILL.md based on eval results.

**Step 4: Commit**

```bash
git add evals/ && git commit -m "test: add eval cases for integration, debug, and audit modes"
```

---

### Task 15: README and Installation Guide

**Files:**
- Create: `README.md`

**Step 1: Write README**

Sections:
- What this skill does (3 modes)
- Supported platforms
- Installation per tool (Claude Code, Codex, Cursor)
- Usage examples
- Contributing (how to add a new platform)
- License

**Step 2: Commit**

```bash
git add README.md && git commit -m "docs: add README with installation and usage guide"
```

---

## Execution Order

Tasks 1-10 are sequential (each builds on the scaffold).
Tasks 11-13 (adapters) can be parallelized — they're independent.
Task 14 (evals) depends on adapters being complete.
Task 15 (README) can happen anytime after Task 1.

```
Task 1 (scaffold)
  → Task 2 (web)
  → Task 3 (iOS)
  → Task 4 (Android)
  → Task 5 (React Native)
  → Task 6 (Flutter)
  → Task 7 (Server SDKs)
  → Task 8 (Common debugging)
  → Task 9 (Platform debugging)
  → Task 10 (Audit feature map)
  → Tasks 11, 12, 13 (adapters — parallel)
  → Task 14 (evals)
  → Task 15 (README)
```
