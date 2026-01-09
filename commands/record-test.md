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
  "actionCount": 0,
  "actions": []
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

GUIDED RECORDING MODE
─────────────────────
1. Do an action on your device (tap, type, swipe)
2. Say "done" when finished
3. I'll capture and analyze
4. Wait for "Ready" before next action

Do your first action, then say "done".
══════════════════════════════════════════════
```

### 7. Guided Recording Loop

Wait for user to say "done" (or variations: "ok", "next", "ready", "finished").

When user signals done:

**Step 7.1: Burst Capture**

Take 7 screenshots rapidly over 2 seconds:

```
for i in 1 to 7:
  - mobile_take_screenshot → save to screenshots/{seq}_{timestamp}.png
  - mobile_list_elements_on_screen → store elements
  - wait 300ms
```

Save each screenshot to `tests/{test-name}/screenshots/` with naming:
- `{action_number}_{shot_number}_{timestamp}.png`
- Example: `001_01_20260109_204512.png`

**Step 7.2: Analyze Screenshots**

Compare consecutive screenshots:
1. Diff element lists - what appeared/disappeared?
2. Identify distinct states (group similar screenshots)
3. Detect: tap (element gone), type (text changed), swipe (elements shifted), navigation (new screen)

**Step 7.3: Report to User**

```
Captured 7 screenshots, detected {N} states:
  1. {state_1_description}
  2. {state_2_description}
  ...

Inferred action: {action_type} "{target}" → {result}

Ready for next action.
```

**Step 7.4: Update Recording State**

Add to `.claude/recording-state.json`:
```json
{
  "actions": [
    {
      "actionNumber": 1,
      "type": "tap",
      "target": "Login",
      "states": ["home", "loading", "dashboard"],
      "screenshots": ["001_01.png", "001_02.png", ...],
      "timestamp": "2026-01-09T20:45:12Z"
    }
  ]
}
```

**Step 7.5: Loop**

Repeat from Step 7.1 until user says "stop recording" or "/stop-recording".

### 8. Handle User Messages During Recording

- "done" / "ok" / "next" / "finished" → trigger burst capture
- "stop" / "stop recording" / "/stop-recording" → end recording
- "undo" / "back" → remove last recorded action
- Any other message → treat as context hint, store with next action

## Action Detection Heuristics

### Tap Detection
- Element that was visible is now gone or changed state
- New screen appeared after element interaction
- Button/clickable element was in focus area

### Type Detection
- Text field content changed
- Keyboard was visible
- Input field has new value

### Swipe Detection
- Multiple elements shifted in same direction
- Scroll position changed
- List items changed

### Navigation Detection
- Significant screen layout change
- New elements appeared that weren't visible before
- Different screen title or header

## Error Handling

- If device disconnects: pause recording, notify user
- If app crashes: record the crash, offer to stop or continue
- If unable to detect actions: ask user to describe what they did

## Notes

- Recording continues until `/stop-recording` is called
- User can provide hints during recording to improve accuracy
- Screenshots are stored temporarily for YAML generation
