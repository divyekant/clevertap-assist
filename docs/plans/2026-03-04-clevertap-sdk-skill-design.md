# CleverTap SDK Skill — Design Document

**Date:** 2026-03-04
**Status:** Approved

## Overview

A multi-platform, multi-tool skill that helps developers integrate and debug CleverTap SDKs. Works across Claude Code, Codex, and Cursor via thin adapter files sharing a single source of truth.

## Goals

- Help developers add CleverTap SDK to any project (Web, iOS, Android, React Native, Flutter, server-side)
- Help developers diagnose and fix CleverTap SDK issues
- Work in Claude Code, Codex, and Cursor from the same knowledge base
- Stay current as CleverTap's SDK ecosystem evolves
- Open source and extensible to third-party integrations later

## Architecture: Monorepo with Adapter Layer

```
clevertap-sdk/
├── core/                          # Tool-agnostic knowledge (pure markdown)
│   ├── integration/
│   │   ├── web.md                 # Web SDK (vanilla JS, React, Next.js, Vue, Angular)
│   │   ├── ios.md                 # iOS (Swift, ObjC)
│   │   ├── android.md             # Android (Kotlin, Java)
│   │   ├── react-native.md
│   │   ├── flutter.md
│   │   └── server/
│   │       ├── node.md
│   │       ├── python.md
│   │       └── java.md
│   └── debugging/
│       ├── common.md              # Cross-platform gotchas
│       ├── web.md                 # Web-specific diagnosis
│       ├── ios.md
│       ├── android.md
│       ├── react-native.md
│       ├── flutter.md
│       └── server.md
├── adapters/
│   ├── claude-code/
│   │   └── SKILL.md               # CC skill with trigger phrases and tool usage
│   ├── codex/
│   │   └── AGENTS.md              # Codex agent config
│   └── cursor/
│       └── .cursorrules           # Cursor rules
├── evals/
│   └── evals.json                 # Test cases per platform
└── README.md
```

### Key Principle

`core/` = pure knowledge (what). `adapters/` = workflow mechanics (how). Adapters never duplicate knowledge — they reference core files.

## Workflow

### Integration Mode

Trigger: developer asks to add CleverTap to their app.

1. **Detect platform** — scan project files for signature markers
2. **Ask for credentials** — Account ID, Token, Region (every time)
3. **Load reference** — read matching `core/integration/<platform>.md`
4. **Generate code** — write SDK install, init, basic event tracking into the project
5. **Verify** — static + runtime checks to confirm correct setup

### Debug Mode

Trigger: developer reports CleverTap SDK issue.

1. **Detect platform** — same scanning logic
2. **Quick checklist** — load `core/debugging/common.md` + platform-specific debug file, pattern-match
3. **Active diagnosis** — if checklist doesn't resolve, scan codebase for SDK usage and identify the issue
4. **Fix + verify** — apply fix and confirm resolution

## Knowledge Layer

### Curated Reference Files (Primary)

Each file is self-contained. Loading one file gives the AI everything it needs for that platform.

**Integration files** contain:
- Prerequisites (minimum SDK version, platform requirements)
- Installation (package manager commands)
- Initialization (exact init code with credential placeholders)
- Basic event tracking (push event, user profile, push tokens)
- Framework variants (subsections for React, Next.js, Vue, etc.)
- Common pitfalls (platform-specific gotchas)

**Debugging files** contain:
- Diagnostic checklist (ordered)
- Symptoms-to-causes mapping
- Debug tooling (how to enable verbose logging)
- Known issues (version-specific)

### Runtime Doc Fetch (Fallback)

When reference files don't cover a case (new SDK feature, unfamiliar framework), the skill falls back to fetching from `developer.clevertap.com`. Reference files are checked first for speed and reliability.

## Platform Detection

| Signal File | Platform |
|---|---|
| `package.json` (no `react-native`) | Web (JS/TS) |
| `package.json` + `react-native` dep | React Native |
| `Podfile` or `*.xcodeproj` | iOS |
| `build.gradle` or `build.gradle.kts` | Android |
| `pubspec.yaml` | Flutter |
| `requirements.txt` or `pyproject.toml` | Python (server) |
| `package.json` + no frontend framework | Node (server) |
| `pom.xml` or `build.gradle` (no Android manifest) | Java (server) |

**Ambiguous projects** (monorepos): ask the developer which platform to target.
**Unrecognized projects**: ask the developer, attempt runtime doc fetch.

**Web framework sub-detection**: check dependencies for `react`, `next`, `vue`, `@angular/core`.

## Verification Strategy

### Static Verification (all platforms)
- SDK dependency installed (check lock file / build output)
- Init code exists and runs before event tracking calls
- Credentials present and non-placeholder
- No duplicate init calls
- Region matches account

### Runtime Verification (platform-dependent)
- **Web**: browser console `[CT]` prefix logs at debug level 3
- **iOS**: Xcode console CleverTap init success log
- **Android**: logcat CleverTap initialization confirmation
- **React Native / Flutter**: bridge-specific logs
- **Server-side**: test API call + HTTP response check

### Scope Limits
- Does NOT spin up servers or emulators — tells developer what to run and what to look for
- Does NOT access CleverTap dashboard — verifies client-side setup only

## Adapter Design

All adapters describe the same workflow (detect → load → generate/diagnose → verify). Only format and tool-specific mechanics differ.

- **Claude Code** (`SKILL.md`): skill frontmatter, trigger phrases, Read/Bash/Glob tool usage
- **Codex** (`AGENTS.md`): agent config format, context file references
- **Cursor** (`.cursorrules`): rules format, file references

Adding a new tool = one new adapter file pointing at `core/`.

## Distribution

Open source standalone repo. Users clone/install and point their AI tool at the appropriate adapter.

## Future Extensibility

- New platform: add `core/integration/<platform>.md` + `core/debugging/<platform>.md`
- Third-party integrations (Segment, mParticle, Rudderstack, GTM): add `core/integration/integrations/<name>.md`
- New AI tool: add `adapters/<tool>/` with one config file

## Credentials

Asked every time via the adapter's prompt mechanism. Not stored. The skill uses placeholders in generated code that the developer fills in, or accepts values interactively during the session.
