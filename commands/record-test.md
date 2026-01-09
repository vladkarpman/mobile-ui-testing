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

DUAL-DETECTION MODE
───────────────────
Channel 1: Touch events (< 10ms latency)
Channel 2: Element monitoring (500ms polling)

I'm watching for your actions in real-time...

Commands:
  "stop" - End recording
  "undo" - Remove last action
  "done" - Force capture (if I missed something)

══════════════════════════════════════════════
```

### 7. Start Touch Event Stream (Android)

For Android devices, start ADB getevent to capture touch events:

**Command:**
```bash
adb -s {device} shell getevent -lt 2>/dev/null
```

**Parse touch events to detect:**
- **Tap:** BTN_TOUCH DOWN → UP within 200ms, < 50px movement
- **Swipe:** BTN_TOUCH DOWN, movement > 100px, UP
- **Long press:** BTN_TOUCH DOWN held > 500ms

**On touch detected:**
1. Record touch coordinates (x, y) and timestamp
2. Immediately capture screenshot
3. Get elements and find element at (x, y)
4. Record action: `{type, target, coordinates, source: "touch_event"}`
5. Notify user: "→ [N] tap at (x, y) on '{element}'"

**Coordinate conversion:**
Raw getevent coordinates need conversion to screen coordinates.
See `lib/touch-parser.md` for details.

### 8. Element Change Monitoring (Backup Channel)

Poll `mobile_list_elements_on_screen` every 500ms as backup:

**Purpose:**
- Confirms touch events had visible effect
- Catches non-touch changes (system dialogs, keyboards)
- Provides element context for coordinate-only touches

**On element change detected:**
1. Check if recent touch event correlates
2. If yes: mark touch event as "confirmed"
3. If no touch: likely system event, record with low confidence

### 9. Merge Detection Channels

Correlate touch events with element changes for high-confidence recording:

| Touch Event | Element Change | Action |
|-------------|----------------|--------|
| tap at (x,y) | element at (x,y) gone | tap "{element}" (HIGH confidence) |
| tap at (x,y) | no change | tap [x, y] (LOW confidence, ask user) |
| no touch | element changed | ignore (system event) |
| swipe detected | elements shifted | swipe {direction} (HIGH confidence) |

**Deduplication:**
- If same action detected by both channels within 500ms → merge into one
- Prefer element name over coordinates when available

### 10. Action Inference

When change detected, infer action type by analyzing element diff:

| Change Pattern | Inferred Action | Confidence |
|----------------|-----------------|------------|
| Clickable element gone + new screen | tap on element | high |
| Text field value changed | type in field | high |
| Multiple elements shifted same direction | swipe | medium |
| Element disappeared, same screen | tap (dismiss) | medium |
| New modal/dialog appeared | system event | low |

**Inference Logic:**

```
def infer_action(before, after):
  disappeared = elements_in(before) - elements_in(after)
  appeared = elements_in(after) - elements_in(before)

  # Check for tap
  for elem in disappeared:
    if elem.is_clickable and len(appeared) > 2:
      return Action(type="tap", target=elem.text, confidence="high")

  # Check for type
  for elem in after:
    if elem.type == "EditText":
      before_elem = find_matching(before, elem)
      if before_elem and elem.text != before_elem.text:
        return Action(type="type", target=elem.text, confidence="high")

  # Check for swipe
  shift = calculate_element_shift(before, after)
  if shift.magnitude > 100:
    return Action(type="swipe", direction=shift.direction, confidence="medium")

  return None  # No significant action detected
```

### 11. Handle User Commands

While monitoring, handle these commands:

| Command | Action |
|---------|--------|
| "stop" | End recording, proceed to stop-recording |
| "undo" | Remove last detected action |
| "done" | Force a capture (if auto-detection missed something) |
| Any other text | Store as context for next action |

On "undo":
```
→ Removed: tap "Login"
→ Actions: {count - 1} recorded
```

On user text (context):
```
→ Context noted: "{user_text}"
→ Will attach to next detected action
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
