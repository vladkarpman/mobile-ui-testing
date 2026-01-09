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

### 3. Load Recording State

Read the recording state file and validate:
- Test name exists
- App package is set
- Snapshots were captured

### 4. Stop Background Capture

Kill the ADB getevent process:
```bash
pkill -f "adb.*getevent"
```

Verify touch_log.txt was written:
```bash
# Check file exists and has content
ls -la tests/{test-name}/touch_log.txt
wc -l tests/{test-name}/touch_log.txt
```

### 5. Parse Touch Log

Read `tests/{test-name}/touch_log.txt` and extract gestures:

1. Parse using logic from `lib/touch-parser.md`
2. Load max coordinates from snapshots.json
3. Convert raw coordinates to screen coordinates:
   ```
   screen_x = (raw_x / max_x) * screen_width
   screen_y = (raw_y / max_y) * screen_height
   ```
4. Detect gesture types:
   - **Tap:** DOWN→UP < 200ms, movement < 50px
   - **Swipe:** DOWN→UP, movement >= 100px
   - **Long press:** DOWN→UP >= 500ms, movement < 50px

Output: Array of touch events with timestamps and screen coordinates

### 6. Correlate Touch Events with Element Snapshots

For each touch event:

1. Find snapshot with closest timestamp (within 200ms)
2. Search elements for one containing touch coordinates:
   - Check if (x, y) is within element bounds
   - Prefer clickable elements over containers
3. Record correlation:
   ```json
   {
     "touch": {"type": "tap", "x": 540, "y": 800, "timestamp": "..."},
     "element": {"text": "Login", "bounds": [500, 780, 580, 820]},
     "confidence": "high"
   }
   ```

If no element found at coordinates:
- confidence: "low"
- element: null

### 7. Detect Type Actions

Compare consecutive snapshots for text field changes:

1. Find EditText/TextField elements
2. If text content changed between snapshots
3. Record type action:
   ```json
   {
     "type": "type",
     "text": "user@email.com",
     "target": "Email field",
     "confidence": "high"
   }
   ```

### 8. Generate YAML Test

Build test.yaml from correlated actions:

```yaml
config:
  app: {app-package}

steps:
  # HIGH confidence - element found
  - tap: "Login"

  # HIGH confidence - text field changed
  - type:
      text: "user@email.com"
      into: "Email"

  # LOW confidence - no element at coords
  - tap: [540, 1200]

  # HIGH confidence - elements shifted
  - swipe: down
```

Prefer element text over coordinates when available.

Create the test folder structure:

```bash
# Create folder structure
mkdir -p tests/{test-name}/screenshots
mkdir -p tests/{test-name}/baselines
mkdir -p tests/{test-name}/reports
```

Generate the test file at `tests/{test-name}/test.yaml`:

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
      {generated steps with confidence comments}

      # --- Verification (add manually) ---
      # - verify_screen: "Expected final state"
```

### 9. Show Analysis Report

Output summary:

```
Analysis complete: {N} touch events → {M} actions
══════════════════════════════════════════════

Actions detected:
  1. tap "Login"           ← HIGH (element found)
  2. type "user@email.com" ← HIGH (text field changed)
  3. tap [540, 1200]       ← LOW (no element at coords)
  ...

Low confidence actions: {count}
Consider reviewing actions marked LOW.

Generated: tests/{test-name}/test.yaml
```

### 10. Smart Enhancements

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

### 11. Create Baseline Images (Optional)

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

### 12. Cleanup

1. Delete `.claude/recording-state.json`
2. Move all captured screenshots to `tests/{test-name}/screenshots/`
3. Keep `touch_log.txt` for debugging purposes
4. Generate recording summary report at `tests/{test-name}/reports/recording-summary.html`

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

  <h2>Analysis Results</h2>
  <p>Touch events: {N} | Actions generated: {M}</p>
  <p>High confidence: {high_count} | Low confidence: {low_count}</p>

  <h2>Timeline</h2>
  <ul>
    <li>[00:00] App launched</li>
    <li>[00:03] Tap: "Email" field (HIGH)</li>
    <li>[00:05] Type: "user@example.com" (HIGH)</li>
    <!-- List of all actions with timestamps and confidence -->
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

### 13. Report Success

```
Recording complete!
══════════════════════════════════════════════

Generated: tests/{test-name}/test.yaml

Folder structure:
  tests/{test-name}/
  ├── test.yaml          ← Run with /run-test
  ├── touch_log.txt      ← Raw touch data
  ├── snapshots.json     ← Element snapshots
  ├── screenshots/       ← {N} images captured
  ├── baselines/         ← {M} baseline images
  └── reports/           ← Recording summary

Analysis summary:
  Touch events: {N}
  Actions detected: {M}
  High confidence: {high_count}
  Low confidence: {low_count}

Steps:
  1. tap: "Email"              ← HIGH
  2. type: "user@example.com"  ← HIGH
  3. tap: "Password"           ← HIGH
  4. type: "********"          ← HIGH
  5. tap: "Login"              ← HIGH
  6. tap: [540, 1200]          ← LOW (no element)

══════════════════════════════════════════════

Next: Review test.yaml and run with /run-test tests/{test-name}
      Pay attention to LOW confidence actions - may need manual adjustment.
```

### 14. Show Generated File

Display the full generated YAML for user review:

```yaml
# {full generated content}
```

## Action Conversion Rules

| Detected | Generated YAML | Confidence |
|----------|----------------|------------|
| Tap on element with text | `- tap: "Element Text"` | HIGH |
| Tap on element without text | `- tap: [x, y]` with comment | LOW |
| Text input detected | `- type: { text: "value", into: "Field" }` | HIGH |
| Swipe up | `- swipe: up` | HIGH |
| Swipe down | `- swipe: down` | HIGH |
| Swipe left | `- swipe: left` | HIGH |
| Swipe right | `- swipe: right` | HIGH |
| Long press on element | `- long_press: "Element"` | HIGH |
| Long press at coords | `- long_press: [x, y]` | LOW |
| Back button | `- press: back` | HIGH |
| Time gap > 2s | `- wait: {gap}s` | N/A |

## Confidence Levels

- **HIGH:** Element found at touch coordinates, or text field change detected
- **LOW:** No element found at coordinates, using raw coordinate fallback

Low confidence actions should be reviewed manually - they may indicate:
- Touch on non-labeled elements
- Touch on dynamically loaded content
- Timing mismatch between touch and snapshot

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
- If touch_log.txt empty: warn user that no touches were captured
- If snapshot correlation fails: use coordinates with LOW confidence

## Usage

```
/stop-recording
```

No arguments needed - uses the active recording session.
