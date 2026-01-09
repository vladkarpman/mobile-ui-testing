---
name: record-test
description: Start recording user actions to generate a YAML test
argument-hint: <test-name>
allowed-tools:
  - Read
  - Write
  - Glob
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

Create a recording state file at `.claude/recording-state.json`:

```json
{
  "testName": "{test-name}",
  "appPackage": "{detected-package}",
  "device": "{device-id}",
  "startTime": "{ISO timestamp}",
  "status": "recording",
  "actions": [],
  "screenshots": []
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

Now interact with your app. I'm watching for:
  • Taps and clicks
  • Text input
  • Swipes and scrolls
  • Screen changes

When you're done, say "stop recording" or use:
  /stop-recording

Tip: Pause briefly between actions so I can capture each one.

══════════════════════════════════════════════
```

### 7. Continuous Monitoring

While recording is active, periodically (every 1-2 seconds):

1. **Take screenshot**: `mobile_take_screenshot`
2. **Get current elements**: `mobile_list_elements_on_screen`
3. **Detect changes** by comparing to previous state:
   - New elements appeared → possible navigation
   - Elements disappeared → possible navigation
   - Text field content changed → user typed something
   - Different screen layout → swipe or navigation occurred

4. **Infer actions** from changes:
   - If a button disappeared and new screen appeared → tap on that button
   - If text field has new content → type action
   - If elements shifted position → swipe action

5. **Record inferred action** with:
   - Action type (tap, type, swipe)
   - Target element (text or coordinates)
   - Screenshot reference
   - Timestamp

### 8. Handle User Messages

While recording, if user provides context:
- "I just tapped the login button" → confirm and record tap action
- "I entered my email" → record type action with placeholder
- "I scrolled down" → record swipe action

### 9. Recording State Updates

After each detected action, update `.claude/recording-state.json`:

```json
{
  "actions": [
    {
      "type": "tap",
      "target": "Login",
      "coordinates": [540, 580],
      "timestamp": "2025-01-09T16:30:05Z",
      "screenshot": "recording_001.png"
    },
    {
      "type": "type",
      "text": "[user input]",
      "timestamp": "2025-01-09T16:30:08Z",
      "screenshot": "recording_002.png"
    }
  ]
}
```

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
