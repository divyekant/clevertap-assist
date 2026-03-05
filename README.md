# CleverTap Assist

AI-powered SDK integration, debugging, and auditing for CleverTap -- across every platform.

---

## What It Does

CleverTap Assist is a knowledge pack that plugs into AI coding tools (Claude Code, Codex, Cursor) and gives them deep understanding of the CleverTap SDK. It operates in three modes:

- **Integrate** -- Add the CleverTap SDK to a project from scratch. Detects your platform, asks for credentials, writes real code into your codebase, and verifies the setup.
- **Debug** -- Diagnose why events aren't showing, push notifications aren't arriving, or the SDK isn't initializing. Walks through a structured checklist and scans your code for known issues.
- **Audit** -- Scan an entire codebase and produce a usage report: which CleverTap features are integrated, where they're called, and whether anything is misconfigured.

---

## Supported Platforms

| Platform | Languages / Frameworks |
|---|---|
| **Web** | Vanilla JS, React, Next.js, Vue, Angular |
| **iOS** | Swift, Objective-C (CocoaPods, SPM) |
| **Android** | Kotlin, Java (Gradle) |
| **React Native** | JavaScript/TypeScript with native iOS + Android setup |
| **Flutter** | Dart with native iOS + Android + Web setup |
| **Server -- Node.js** | REST API client via Node.js |
| **Server -- Python** | REST API client via Python |
| **Server -- Java** | REST API client via Maven/Gradle |

---

## Installation

CleverTap Assist works through adapters -- one per AI tool. Pick the tool you use and follow the steps below.

### Claude Code

CleverTap Assist ships as a [Claude Code custom slash command](https://docs.anthropic.com/en/docs/claude-code/tutorials#create-custom-slash-commands). Install it by symlinking or copying the skill file.

**Option A: Symlink (recommended -- stays in sync with updates)**

```bash
# From your project directory
mkdir -p .claude/commands
ln -s /path/to/clevertap-assist/adapters/claude-code/SKILL.md .claude/commands/clevertap.md
```

**Option B: Copy**

```bash
mkdir -p .claude/commands
cp /path/to/clevertap-assist/adapters/claude-code/SKILL.md .claude/commands/clevertap.md
```

Then invoke it in Claude Code with `/clevertap`.

### Codex (OpenAI)

Copy the `AGENTS.md` file to your project root. Codex reads it automatically.

```bash
cp /path/to/clevertap-assist/adapters/codex/AGENTS.md ./AGENTS.md
```

### Cursor

Copy the `.cursorrules` file to your project root. Cursor reads it automatically on every prompt.

```bash
cp /path/to/clevertap-assist/adapters/cursor/.cursorrules ./.cursorrules
```

> **Note:** All adapters reference the `core/` directory for SDK knowledge. Keep the full `clevertap-assist` repository accessible on disk so the AI tool can read the reference files at runtime.

---

## Usage Examples

Once installed, use natural language in your AI tool. Here are some example prompts:

**Integration**

```
Add CleverTap to my React app
```

The assistant will detect your web framework, ask for your Account ID, Token, and Region, then write initialization code, a sample event call, and user identity setup directly into your project files.

**Debugging**

```
My CleverTap events aren't showing up in the dashboard
```

The assistant will walk through a diagnostic checklist: is the SDK initialized before events are tracked? Are credentials valid? Is debug logging enabled? It scans your code for known issues and applies fixes.

```
Debug why push notifications aren't working
```

The assistant checks platform-specific push setup -- FCM token registration on Android, APNs entitlements on iOS, notification channel creation for Android 8+, and service worker configuration for web push.

**Auditing**

```
Show me how CleverTap is used across my codebase
```

The assistant scans every source file for CleverTap SDK method calls, groups them by feature area (Identity, Events, Push, App Inbox, In-App, Web Push, Product Experiences, Geofencing, Native Display), and generates a structured usage report with file locations and issue flags.

---

## How It Works

```
clevertap-assist/
  core/                     # Platform knowledge (the source of truth)
    integration/            # Step-by-step SDK setup per platform
      web.md
      ios.md
      android.md
      react-native.md
      flutter.md
      server/
        node.md
        python.md
        java.md
    debugging/              # Diagnostic checklists and known issues
      common.md             # Cross-platform checklist (always read first)
      web.md
      ios.md
      android.md
      react-native.md
      flutter.md
      server.md
    audit/                  # Feature map with grep patterns and report template
      features.md
  adapters/                 # Tool-specific entry points
    claude-code/
      SKILL.md              # Claude Code custom slash command
    codex/
      AGENTS.md             # Codex agent instructions
    cursor/
      .cursorrules          # Cursor rules file
```

The `core/` directory contains all SDK knowledge: method signatures, initialization patterns, debugging checklists, and audit search patterns. Each file is self-contained -- reading one file gives the AI everything it needs for that platform and mode.

The `adapters/` directory contains tool-specific wrappers. Each adapter teaches its respective AI tool how to detect the developer's platform, select the right mode (integrate/debug/audit), read the appropriate `core/` reference file, and act on it. The adapters contain no SDK knowledge themselves -- they only define the workflow.

This separation means updating SDK knowledge (e.g., a new API method or a new platform) only requires editing files in `core/`. All adapters pick up the changes automatically.

---

## Docs

- **[LLM Quickstart](docs/llm-quickstart.md)** — Copy-paste this into any LLM (Claude, GPT, Codex, Gemini) to give it full CleverTap Assist context. No tool-specific setup required.

---

## Contributing

### Adding a New Platform

1. **Integration reference** -- Create `core/integration/<platform>.md` with:
   - Package installation commands
   - SDK initialization code
   - Event tracking example
   - User identity setup (`onUserLogin` equivalent)
   - Static verification checklist

2. **Debugging reference** -- Create `core/debugging/<platform>.md` with:
   - Platform-specific symptoms and fixes
   - Debug logging enablement instructions
   - Common pitfalls for that platform

3. **Audit patterns** -- Update `core/audit/features.md`:
   - Add SDK method signatures for the new platform in section 2
   - Add grep patterns in section 3
   - Add platform detection signals in section 6

4. **Test** -- Use the installed adapter to run all three modes against a sample project on the new platform. Verify that integration code compiles, debugging advice is actionable, and the audit report captures all SDK calls.

### Adding a New Adapter

Create a new directory under `adapters/<tool-name>/` with the tool's configuration file format. The adapter should:

- Detect the developer's platform from project files
- Determine the mode (integrate, debug, audit) from the developer's request
- Read the appropriate `core/` reference file
- Follow the workflow defined in the reference file

---

## Found a Case That Doesn't Work?

[Open an eval case issue](../../issues/new?template=eval-case.yml) — describe what you asked, what platform you're on, and what went wrong. We'll add it to the test suite so future versions handle it correctly.

---

## License

MIT
