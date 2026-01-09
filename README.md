# Mobile UI Testing Plugin

A Claude Code plugin for writing and running YAML-based mobile UI tests with [mobile-mcp](https://github.com/anthropics/mobile-mcp).

[![npm version](https://img.shields.io/npm/v/claude-plugin-mobile-ui-testing.svg)](https://www.npmjs.com/package/claude-plugin-mobile-ui-testing)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Features

- **Auto-permissions** - mobile-mcp tools auto-approved, no manual confirmation needed
- **`/run-test`** - Execute YAML test files with detailed output and HTML reports
- **`/create-test`** - Scaffold new test files with proper structure
- **`/generate-test`** - Generate tests from natural language descriptions
- **`/record-test`** - Record user actions and generate YAML automatically
- **YAML Test Schema Knowledge** - Claude understands the complete test format
- **No compilation** - Just write YAML and run

## Prerequisites

Before using this plugin, ensure you have:

1. **Claude Code CLI** installed
   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

2. **mobile-mcp** configured in Claude Code
   - Follow [mobile-mcp setup guide](https://github.com/anthropics/mobile-mcp)

3. **Connected device**
   - Android: Device connected via `adb` (run `adb devices` to verify)
   - iOS: Simulator running or device connected

## Installation

### Option 1: Marketplace (Recommended)

```bash
# Add the marketplace (one time)
claude plugin marketplace add vladkarpman/vladkarpman-plugins

# Install the plugin
claude plugin install mobile-ui-testing
```

### Option 2: Project-level

Clone into your project's plugins directory:

```bash
cd your-project
mkdir -p .claude/plugins
cd .claude/plugins
git clone https://github.com/vladkarpman/mobile-ui-testing.git
```

### Option 3: Session-only

Load for a single session:

```bash
claude --plugin-dir /path/to/mobile-ui-testing
```

## Uninstallation

### If installed via marketplace:

```bash
claude plugin uninstall mobile-ui-testing
```

### If installed as project-level:

```bash
rm -rf .claude/plugins/mobile-ui-testing
```

## Quick Start

### 1. Create your first test

```bash
/create-test login
```

This creates `tests/login.test.yaml` with a template.

### 2. Edit the test file

```yaml
config:
  app: com.your.app.package

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: User can login
    steps:
      - tap: "Email"
      - type: "user@example.com"
      - tap: "Password"
      - type: "secret123"
      - tap: "Login"
      - wait: 2s
      - verify_screen: "Home screen after login"
```

### 3. Run the test

```bash
/run-test tests/login.test.yaml
```

## Commands

### `/run-test <file> [--report]`

Execute a YAML test file on a connected device.

```bash
/run-test tests/login.test.yaml
/run-test tests/onboarding.test.yaml --report
```

**With `--report` flag:** Generates HTML/JSON reports in `tests/reports/`.

**Output format:**
```
Running: User login flow
────────────────────────────────────────

  [1/5] wait_for "Login"
        ✓ Found in 1.2s

  [2/5] tap "Email"
        ✓ Tapped at (540, 320)

  [3/5] type "user@example.com"
        ✓ Typed 16 characters

────────────────────────────────────────
✓ PASSED  (5/5 steps in 4.2s)
```

### `/create-test <name>`

Create a new test file from a template.

```bash
/create-test login
/create-test checkout-flow
```

### `/generate-test <description>`

Generate a YAML test from a natural language description.

```bash
/generate-test "user logs in with email and password, sees home screen"
/generate-test "scroll through feed and like the first post"
```

**Example output:**
```yaml
tests:
  - name: User login flow
    steps:
      - wait_for: "Login"
      - tap: "Email"
      - type: "user@example.com"
      - tap: "Password"
      - type: "password123"
      - tap: "Login"
      - verify_screen: "Home screen"
```

### `/record-test <name>`

Start recording user actions to generate a test automatically.

```bash
/record-test login-flow
```

Then interact with your app. Claude watches and records:
- Taps and clicks
- Text input
- Swipes and scrolls
- Screen changes

### `/stop-recording`

Stop recording and generate YAML from captured actions.

```bash
/stop-recording
```

Generates `tests/{name}.test.yaml` with all recorded steps.

## Test File Structure

```yaml
# Configuration
config:
  app: com.your.app.package    # Required: Android package or iOS bundle ID
  device: emulator-5554         # Optional: specific device

# Runs once before all tests
setup:
  - terminate_app
  - launch_app
  - wait: 3s

# Runs once after all tests
teardown:
  - terminate_app

# Test cases
tests:
  - name: Test name here
    description: Optional description
    timeout: 60s              # Optional: default 60s
    tags: [smoke, critical]   # Optional: for filtering
    steps:
      - tap: "Button"
      - type: "text"
      - verify_screen: "Expected state"
```

## Test Folder Structure (v2.0+)

New recordings create an organized folder structure:

```
tests/
└── login-flow/
    ├── test.yaml              # Test definition
    ├── screenshots/           # Captured during recording
    ├── baselines/             # Golden images for validation
    └── reports/               # Run results and history
```

### Recording with Guided Mode

The recording flow guides you step-by-step:

```
/record-test login-flow

> Recording started. Do your first action, then say "done".

*tap Login button*
> done

> Captured 3 states: Home → Loading → Login Form
> Ready for next action.

*type email*
> done

> Captured: type in Email field
> Ready for next action.

...

> stop

> Recording complete! Generated tests/login-flow/test.yaml
```

### Running Folder-Format Tests

```bash
/run-test tests/login-flow/
/run-test tests/login-flow/ --report
```

## Available Actions

### Basic Actions

| Action | Example | Description |
|--------|---------|-------------|
| `tap` | `- tap: "Button"` | Tap element by text |
| `tap` | `- tap: [100, 200]` | Tap at coordinates |
| `double_tap` | `- double_tap: "Element"` | Double tap |
| `long_press` | `- long_press: "Element"` | Long press (500ms default) |
| `type` | `- type: "Hello"` | Type into focused field |
| `swipe` | `- swipe: up` | Swipe direction |
| `press` | `- press: back` | Press button (back, home) |
| `wait` | `- wait: 2s` | Wait for duration |

### App Control

| Action | Example | Description |
|--------|---------|-------------|
| `launch_app` | `- launch_app` | Launch configured app |
| `terminate_app` | `- terminate_app` | Stop configured app |
| `set_orientation` | `- set_orientation: landscape` | Change orientation |
| `open_url` | `- open_url: "https://..."` | Open URL in browser |

### Verification

| Action | Example | Description |
|--------|---------|-------------|
| `verify_screen` | `- verify_screen: "Home with tabs"` | AI verifies screen state |
| `verify_contains` | `- verify_contains: "Welcome"` | Check element exists |
| `verify_no_element` | `- verify_no_element: "Error"` | Check element absent |

### Flow Control

| Action | Example | Description |
|--------|---------|-------------|
| `wait_for` | `- wait_for: "Continue"` | Wait until element appears |
| `if_present` | `- if_present: "Skip"` | Conditional execution |
| `retry` | `- retry: {attempts: 3, steps: [...]}` | Retry on failure |
| `repeat` | `- repeat: {times: 5, steps: [...]}` | Repeat steps |

## Templates

### Basic Navigation Test

```yaml
config:
  app: com.example.app

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: Navigate to Settings
    steps:
      - wait_for: "Home"
      - tap: "Settings"
      - wait: 2s
      - verify_screen: "Settings screen displayed"
      - screenshot: settings_screen
```

### Form Input Test

```yaml
config:
  app: com.example.app

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: Submit contact form
    steps:
      - wait_for: "Contact"
      - tap: "Contact"
      - wait: 1s

      - tap: "Name"
      - type: "John Doe"

      - tap: "Email"
      - type: "john@example.com"

      - tap: "Message"
      - type: "Hello, this is a test message."

      - tap: "Submit"
      - wait_for: "Thank you"
      - verify_screen: "Success confirmation displayed"
```

### Onboarding Flow Test

```yaml
config:
  app: com.example.app

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: Complete onboarding
    timeout: 120s
    steps:
      # Page 1
      - wait_for: "Welcome"
      - screenshot: onboarding_page_1
      - tap: "Next"
      - wait: 1s

      # Page 2
      - wait_for: "Features"
      - screenshot: onboarding_page_2
      - tap: "Next"
      - wait: 1s

      # Page 3
      - wait_for: "Get Started"
      - screenshot: onboarding_page_3
      - tap: "Get Started"
      - wait: 2s

      - verify_screen: "Home screen after onboarding"
```

### Handling Optional Elements

```yaml
tests:
  - name: Login with optional popup handling
    steps:
      - wait_for: "Login"

      # Handle optional notification permission popup
      - if_present: "Allow Notifications"
        then:
          - tap: "Not Now"
          - wait: 500ms

      # Handle optional rate app dialog
      - if_present: "Rate our app"
        then:
          - tap: "Later"
          - wait: 500ms

      - tap: "Email"
      - type: "user@example.com"
      - tap: "Password"
      - type: "password123"
      - tap: "Login"

      - wait_for: "Home"
      - verify_screen: "Logged in successfully"
```

### Retry Flaky Operations

```yaml
tests:
  - name: Upload with retry
    steps:
      - tap: "Upload"

      - retry:
          attempts: 3
          delay: 2s
          steps:
            - tap: "Select File"
            - wait: 1s
            - tap: "photo.jpg"
            - tap: "Upload"
            - wait_for: "Upload complete"

      - verify_screen: "File uploaded successfully"
```

## Asking for Help

The plugin's skill activates when you ask questions about mobile testing:

```
How do I write a mobile UI test?
What YAML actions are available?
How do I verify a screen state?
How do I handle optional popups?
What's the best way to test login flows?
```

## Documentation

- [Actions Reference](skills/yaml-test-schema/references/actions.md) - All available actions
- [Assertions Reference](skills/yaml-test-schema/references/assertions.md) - Verification actions
- [Flow Control Reference](skills/yaml-test-schema/references/flow-control.md) - Conditionals, loops, retries

## Troubleshooting

### "No device found"

Ensure your device is connected:
```bash
# Android
adb devices

# iOS Simulator
xcrun simctl list devices | grep Booted
```

### "Element not found"

- Add `wait_for` before interacting with elements
- Check element text matches exactly (case-sensitive)
- Use `screenshot` to debug what's on screen

### "Test timeout"

- Increase timeout in test: `timeout: 120s`
- Add `wait` actions for slow operations
- Use `wait_for` instead of fixed `wait` durations

## Contributing

Contributions welcome! Please open an issue or submit a PR.

## License

MIT
