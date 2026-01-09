---
name: stop-recording
description: Stop recording and generate YAML test from captured actions
allowed-tools:
  - Read
  - Write
  - Bash
  - mcp__mobile-mcp__mobile_take_screenshot
  - mcp__mobile-mcp__mobile_list_elements_on_screen
---

# Stop Recording - Generate YAML Test

Stop the active recording session and generate a YAML test file from captured actions.

## Process

### 1. Check Recording State

Read `.claude/recording-state.json` to get:
- Test name
- App package
- Recorded actions
- Screenshots

If no active recording:
```
No active recording found.

Start a new recording with:
  /record-test <test-name>
```

### 2. Capture Final State

1. Take final screenshot
2. Get final elements list
3. Add to recording state

### 3. Process Recorded Actions

For each recorded action, convert to YAML step:

**Tap actions:**
```yaml
- tap: "{element text}"  # Prefer text over coordinates
```

**Type actions:**
```yaml
- tap: "{input field}"   # Focus the field first
- type: "{entered text}"
```

**Swipe actions:**
```yaml
- swipe: {direction}     # up, down, left, right
```

**Wait actions (inferred from timing gaps):**
```yaml
- wait: 1s               # Add between actions with >1s gap
```

### 4. Generate YAML Test File

Create the test folder structure and generate `tests/{test-name}/test.yaml`:

```bash
# Create folder structure
mkdir -p tests/{test-name}/screenshots
mkdir -p tests/{test-name}/baselines
mkdir -p tests/{test-name}/reports
```

Generate the test file:

```yaml
# {Test Name} (Recorded)
# Recorded on: {date}
# Duration: {recording duration}

config:
  app: {app-package}

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: {test-name} (recorded)
    description: Recorded test - review and customize
    timeout: {duration + buffer}s
    steps:
      # --- Recorded Actions ---
      {generated steps}

      # --- Verification (add manually) ---
      # - verify_screen: "Expected final state"
```

### 5. Smart Enhancements

Apply these improvements to recorded actions:

**Convert coordinates to element text:**
- If tap was at coordinates, find element at that position
- Use element text instead of coordinates when possible
- Fall back to coordinates if no text available

**Add wait_for before critical actions:**
```yaml
- wait_for: "Login"    # Added: ensure element exists
- tap: "Login"
```

**Group related actions:**
```yaml
# Login form
- tap: "Email"
- type: "user@example.com"
- tap: "Password"
- type: "password123"
```

**Add comments for clarity:**
```yaml
# Navigate to settings
- tap: "Menu"
- tap: "Settings"

# Enable dark mode
- tap: "Dark Mode"
- wait: 500ms
```

### 6. Create Baseline Images (Optional)

After generating the test, ask the user if they want to save key screenshots as baselines:

```
Would you like to save screenshots as baseline images for visual validation?

Screenshots captured during recording:
  1. screenshot_001.png - Initial state (after app launch)
  2. screenshot_003.png - After login form filled
  3. screenshot_005.png - Final state (dashboard)

Options:
  [A] Save ALL screenshots as baselines
  [S] SELECT specific screenshots to save
  [N] No baselines needed

Enter choice (A/S/N):
```

**If user selects specific screenshots:**
- Copy selected screenshots to `tests/{test-name}/baselines/`
- Rename with format: `step_{N}_{description}.png` (e.g., `step_01_home.png`, `step_02_login.png`, `step_03_dashboard.png`)
- Add corresponding `verify_screenshot` steps to the generated YAML

**Example baseline verification step:**
```yaml
- verify_screenshot: "baselines/step_03_dashboard.png"
  threshold: 0.95  # 95% similarity required
```

### 7. Cleanup

1. Delete `.claude/recording-state.json`
2. Move all captured screenshots to `tests/{test-name}/screenshots/`
3. Generate recording summary report at `tests/{test-name}/reports/recording-summary.html`

**Recording summary HTML structure:**
```html
<!DOCTYPE html>
<html>
<head>
  <title>{test-name} Recording Summary</title>
  <style>/* Basic styling for readability */</style>
</head>
<body>
  <h1>{test-name} Recording Summary</h1>
  <p>Recorded on: {date} | Duration: {duration}</p>

  <h2>Timeline</h2>
  <ul>
    <li>[00:00] App launched</li>
    <li>[00:03] Tap: "Email" field</li>
    <li>[00:05] Type: "user@example.com"</li>
    <!-- List of all actions with timestamps -->
  </ul>

  <h2>Screenshots</h2>
  <div class="gallery">
    <figure>
      <img src="../screenshots/screenshot_001.png" alt="Step 1">
      <figcaption>Step 1: Initial state</figcaption>
    </figure>
    <figure>
      <img src="../screenshots/screenshot_002.png" alt="Step 2">
      <figcaption>Step 2: After login</figcaption>
    </figure>
    <!-- Gallery of all captured screenshots with relative paths -->
  </div>

  <h2>Generated YAML</h2>
  <pre><code>
# {test-name} (Recorded)
config:
  app: {app-package}

tests:
  - name: {test-name}
    steps:
      - tap: "Email"
      - type: "user@example.com"
      # Full generated test.yaml content
  </code></pre>
</body>
</html>
```

### 8. Report Success

```
Recording complete!
══════════════════════════════════════════════

Generated: tests/{test-name}/test.yaml

Folder structure:
  tests/{test-name}/
  ├── test.yaml          ← Run with /run-test
  ├── screenshots/       ← {N} images captured
  ├── baselines/         ← {M} baseline images
  └── reports/           ← Recording summary

Actions recorded: {count}
Duration: {time}

Steps:
  1. tap: "Email"
  2. type: "user@example.com"
  3. tap: "Password"
  4. type: "********"
  5. tap: "Login"
  6. wait: 2s

══════════════════════════════════════════════

Next: Review test.yaml and run with /run-test tests/{test-name}
```

### 9. Show Generated File

Display the full generated YAML for user review:

```yaml
# {full generated content}
```

## Action Conversion Rules

| Detected | Generated YAML |
|----------|----------------|
| Tap on element with text | `- tap: "Element Text"` |
| Tap on element without text | `- tap: [x, y]` with comment |
| Text input | `- tap: "Field"` then `- type: "value"` |
| Swipe up | `- swipe: up` |
| Swipe down | `- swipe: down` |
| Swipe left | `- swipe: left` |
| Swipe right | `- swipe: right` |
| Long press | `- long_press: "Element"` |
| Back button | `- press: back` |
| Time gap > 2s | `- wait: {gap}s` |

## Placeholder Handling

For sensitive data detected during recording:
- Email addresses → `"user@example.com"`
- Passwords → `"password123"` with comment `# TODO: use test credentials`
- Phone numbers → `"+1234567890"`
- Credit cards → `"4111111111111111"` (test card)

## Error Handling

- If recording state is corrupted: offer to discard or attempt recovery
- If no actions recorded: suggest user may need to interact more slowly
- If screenshots missing: generate YAML without screenshot references

## Usage

```
/stop-recording
```

No arguments needed - uses the active recording session.
