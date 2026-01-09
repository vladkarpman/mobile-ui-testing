---
name: record-test
description: Start recording user actions to generate a YAML test
argument-hint: <test-name>
allowed-tools:
  - Read
  - Write
  - Glob
  - Bash
  - AskUserQuestion
  - mcp__mobile-mcp__mobile_list_available_devices
  - mcp__mobile-mcp__mobile_list_apps
  - mcp__mobile-mcp__mobile_launch_app
  - mcp__mobile-mcp__mobile_get_screen_size
  - mcp__mobile-mcp__mobile_list_elements_on_screen
  - mcp__mobile-mcp__mobile_take_screenshot
---

# Record Test - Start Recording User Actions

Start recording mode to capture user interactions with the app and generate a YAML test file.

## Process

### 1. Parse Test Name

Get the test name from the argument. If not provided, ask:
"What would you like to name this test recording?"

### 2. Detect App Package

Look for existing tests to find the app package:
1. Search `tests/*.test.yaml` for `config.app`
2. Check Android `app/build.gradle*` for applicationId
3. If not found, ask: "What is your app package name?"

### 3. Device Selection

Call `mobile_list_available_devices`:
- If one device: use it automatically
- If multiple: ask user to select
- If none: report error and stop

### 4. Initialize Recording State

Create recording state file and folder structure:

```bash
mkdir -p tests/{test-name}/screenshots
mkdir -p tests/{test-name}/baselines
mkdir -p tests/{test-name}/reports
```

Create `.claude/recording-state.json`:

```json
{
  "testName": "{test-name}",
  "testFolder": "tests/{test-name}",
  "appPackage": "{detected-package}",
  "device": "{device-id}",
  "startTime": "{ISO timestamp}",
  "status": "recording",
  "snapshotCount": 0,
  "screenshotCount": 0
}
```

### 5. Capture Initial State

1. Take initial screenshot: `mobile_take_screenshot`
2. Get initial elements: `mobile_list_elements_on_screen`
3. Store as baseline for detecting changes

### 6. Start Recording Message

Output:
```
Recording started: {test-name}
══════════════════════════════════════════════

Device: {device-name}
App: {app-package}
Saving to: tests/{test-name}/

POST-PROCESSING MODE
────────────────────
Capturing: Touch events + Element snapshots (200ms)
Analysis: After you say "stop"

Interact with your app now...

Commands:
  "stop" - End recording and analyze
  "pause" - Pause capturing
  "resume" - Resume capturing

══════════════════════════════════════════════
```

### 7. Start Background Data Capture (Android)

Start ADB getevent in background to capture all touch events:

**Command:**
```bash
adb -s {device} shell getevent -lt > tests/{test-name}/touch_log.txt 2>/dev/null &
```

Store the process info in recording state for cleanup.

Initialize `tests/{test-name}/snapshots.json`:
```json
{
  "startTime": "{ISO timestamp}",
  "device": "{device-id}",
  "screenSize": {"width": 1080, "height": 2400},
  "maxCoords": {"x": 1079, "y": 2339},
  "snapshots": []
}
```

Get max coordinates for conversion:
```bash
adb -s {device} shell getevent -p | grep -A 5 "ABS_MT_POSITION"
```

### 8. Continuous Element Snapshots

Poll `mobile_list_elements_on_screen` every 200ms:

1. Get current elements
2. Append to snapshots.json with timestamp
3. If elements changed significantly from previous:
   - Take screenshot: `screenshots/snapshot_{N}.png`
   - Update snapshot entry with screenshot reference

### 9. Recording State Schema

Update `.claude/recording-state.json`:
```json
{
  "testName": "{test-name}",
  "testFolder": "tests/{test-name}",
  "appPackage": "{detected-package}",
  "device": "{device-id}",
  "startTime": "{ISO timestamp}",
  "status": "recording",
  "snapshotCount": 0,
  "screenshotCount": 0
}
```

Note: `actions[]` removed - actions reconstructed during analysis.

### 10. Handle User Commands

While recording, handle:

| Command | Action |
|---------|--------|
| "stop" | End recording, trigger analysis phase |
| "pause" | Temporarily pause snapshot polling |
| "resume" | Resume snapshot polling |

### 11. Recording Output

After recording stops, test folder contains:
```
tests/{test-name}/
├── touch_log.txt      # Raw ADB getevent output
├── snapshots.json     # Element states every 200ms
├── screenshots/       # Visual captures on changes
└── (test.yaml created by analysis)
```

## Error Handling

- If device disconnects: pause recording, notify user
- If app crashes: record the crash, offer to stop or continue
- If unable to detect actions: ask user to describe what they did

## Notes

- Recording continues until `/stop-recording` is called
- User can provide hints during recording to improve accuracy
- Screenshots are stored temporarily for YAML generation
