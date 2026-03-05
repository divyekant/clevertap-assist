# CleverTap Assist — LLM Quickstart

> **Copy-paste this document into any LLM** (Claude, GPT, Codex, Gemini, etc.) to give it full context on how to use CleverTap Assist. No tool-specific setup required — just paste this text and start asking questions.

---

## What This Is

CleverTap Assist is a knowledge pack for AI coding assistants. It contains curated reference files covering CleverTap SDK integration, debugging, and auditing across 8 platforms. When you read these files, you get verified method signatures, init patterns, diagnostic checklists, and codebase search patterns — no hallucination needed.

**Three modes:**

| Mode | When to use | What it does |
|------|------------|--------------|
| **Integrate** | "Add CleverTap to my app" | Walks through SDK setup step by step: install, init, event tracking, user identity, push |
| **Debug** | "Events not showing", "Push not working" | Structured diagnostic checklist → platform-specific fixes |
| **Audit** | "Where is CleverTap used in our codebase?" | Scans all source files, groups findings by feature area, generates usage report |

---

## Repository Layout

```
clevertap-assist/
  core/                         ← READ THESE FILES (source of truth)
    integration/                ← SDK setup guides per platform
      web.md                    ← Web (React, Vue, Angular, Vanilla JS)
      ios.md                    ← iOS (Swift, ObjC, CocoaPods, SPM)
      android.md                ← Android (Kotlin, Java, Gradle)
      react-native.md           ← React Native (JS + native setup)
      flutter.md                ← Flutter (Dart + native setup)
      server/
        node.md                 ← Node.js REST API client
        python.md               ← Python REST API client
        java.md                 ← Java REST API client
    debugging/                  ← Diagnostic checklists
      common.md                 ← Cross-platform (always read first)
      web.md, ios.md, android.md, react-native.md, flutter.md, server.md
    audit/
      features.md               ← All 9 feature areas with regex patterns + report template
  adapters/                     ← Tool-specific wrappers (optional)
    claude-code/SKILL.md
    codex/AGENTS.md
    cursor/.cursorrules
```

---

## How to Use (Step by Step)

### 1. Detect the Mode

Read the user's request and classify it:

- **Integration** → words like "add", "integrate", "set up", "install"
- **Debug** → words like "not working", "events not showing", "errors", "fix", "dashboard empty"
- **Audit** → words like "audit", "where is CleverTap used", "coverage", "usage report"

If ambiguous, ask which mode they need.

### 2. Detect the Platform

Look for these files in the user's project:

| File | Platform |
|------|----------|
| `package.json` with `clevertap-web-sdk` | Web |
| `package.json` with `clevertap-react-native` | React Native |
| `pubspec.yaml` with `clevertap_plugin` | Flutter |
| `Podfile` or `*.xcodeproj` | iOS |
| `build.gradle` with `clevertap-android-sdk` | Android |
| `requirements.txt` / `pyproject.toml` | Python (Server) |
| `package.json` (no frontend framework) | Node.js (Server) |
| `pom.xml` | Java (Server) |

If multiple signals → ask the user which platform.
If no signals → ask the user what platform they're using.

### 3. Read the Right Reference File

Based on mode + platform, read the corresponding file from `core/`:

**Integration:** `core/integration/{platform}.md`
**Debug:** `core/debugging/common.md` first, then `core/debugging/{platform}.md`
**Audit:** `core/audit/features.md`

### 4. Follow the Reference File

The reference file contains everything you need: exact method signatures, code patterns, known pitfalls, and verification steps. **Follow it precisely. Do not improvise SDK APIs.**

---

## Integration Mode — Checklist

1. Detect platform (step 2 above)
2. Ask for credentials:
   - **Account ID** (alphanumeric with dashes)
   - **Token** (mobile/web) or **Passcode** (server-side)
   - **Region** (from dashboard URL: `in1`, `us1`, `sg1`, `eu1`, etc.)
3. Read `core/integration/{platform}.md`
4. Follow the file to: install SDK, initialize, add event tracking, add `onUserLogin`
5. Verify: init before events, credentials not placeholder, platform-specific ordering rules
6. Guide runtime verification: enable debug logging → run app → check console → check dashboard

---

## Debug Mode — Checklist

1. Detect platform
2. Read `core/debugging/common.md` — walk through the 7-step diagnostic:
   1. Is debug logging enabled?
   2. Does the SDK initialize?
   3. Are events in debug logs?
   4. Are events reaching the network?
   5. Does the dashboard show them?
   6. Are events duplicated?
   7. Are profile properties correct?
3. Read `core/debugging/{platform}.md` for platform-specific symptoms
4. If checklist resolves it → apply fix
5. If not → scan codebase for CleverTap method calls, check init ordering, check credentials

---

## Audit Mode — Checklist

1. Detect platform
2. Read `core/audit/features.md`
3. Run the 12 search patterns from the file against the codebase:
   - `INIT_ALL`, `IDENTITY_ALL`, `EVENTS_ALL`, `PUSH_ALL`
   - `INBOX_ALL`, `INAPP_ALL`, `WEBPUSH_ALL`, `PRODUCTEXP_ALL`
   - `GEOFENCE_ALL`, `DISPLAY_ALL`, `DEBUG_LOGGING`, `SERVER_API_ALL`
4. Check ordering rules: init before track, identity before profile, push token after init
5. Fill in the report template from the file with actual findings

---

## Critical Rules

1. **Always read the reference file first.** Never generate CleverTap code from memory alone — the reference files have verified, up-to-date method signatures.
2. **Never fabricate SDK APIs.** If something isn't in the reference file, say so and check `https://developer.clevertap.com/docs/` as a fallback.
3. **Ask, don't assume.** Missing platform? Ask. Missing credentials? Ask. Ambiguous mode? Ask.
4. **One step at a time in debug mode.** Walk through the checklist sequentially — don't jump to conclusions.
5. **Warn about placeholder credentials.** If using `YOUR_ACCOUNT_ID`, explicitly tell the user the SDK won't work until they replace it.

---

## Supported Platforms

| Platform | Languages | SDK Type |
|----------|-----------|----------|
| Web | JS, React, Next.js, Vue, Angular | `clevertap-web-sdk` (npm) |
| iOS | Swift, Objective-C | `CleverTap-iOS-SDK` (CocoaPods/SPM) |
| Android | Kotlin, Java | `clevertap-android-sdk` (Gradle) |
| React Native | JS/TS + native | `clevertap-react-native` (npm + native) |
| Flutter | Dart + native | `clevertap_plugin` (pub.dev + native) |
| Node.js | JavaScript | REST API (`/1/upload`) |
| Python | Python | REST API (`/1/upload`) |
| Java | Java | REST API (`/1/upload`) |

---

## Example Prompts

```
Add CleverTap analytics to my React app
```
→ Integration mode, Web (React), reads `core/integration/web.md`

```
My CleverTap events aren't showing in the dashboard. iOS app, Swift.
```
→ Debug mode, iOS, reads `core/debugging/common.md` then `core/debugging/ios.md`

```
Audit our Android codebase for CleverTap usage across all login flows
```
→ Audit mode, Android, reads `core/audit/features.md`

```
Set up CleverTap push notifications in my Flutter app
```
→ Integration mode, Flutter, reads `core/integration/flutter.md`

```
Why am I getting 429 errors from CleverTap's API in my Node.js backend?
```
→ Debug mode, Node.js (Server), reads `core/debugging/server.md`

---

## Quick Reference: Key CleverTap Methods

| Operation | Web | iOS | Android |
|-----------|-----|-----|---------|
| Initialize | `clevertap.init(id, region)` | `CleverTap.autoIntegrate()` | `ActivityLifecycleCallback.register(this)` |
| Track event | `clevertap.event.push(name, props)` | `.recordEvent(name, props)` | `.pushEvent(name, props)` |
| User login | `clevertap.onUserLogin.push({Site:{}})` | `.onUserLogin(profile)` | `.onUserLogin(profile)` |
| Push token | N/A | `.setPushToken(token)` | `.pushFcmRegistrationId(token, true)` |
| Debug logs | `clevertap.setLogLevel(3)` | `CleverTap.setDebugLevel(.debug)` | `CleverTapAPI.setDebugLevel(3)` |

> These are abbreviated — always read the full reference file for complete signatures and edge cases.
