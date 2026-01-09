---
name: run-test
description: Execute a YAML mobile UI test file on a connected device
argument-hint: <test-path> [--report]
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - TodoWrite
  - mcp__mobile-mcp__mobile_list_available_devices
  - mcp__mobile-mcp__mobile_list_apps
  - mcp__mobile-mcp__mobile_launch_app
  - mcp__mobile-mcp__mobile_terminate_app
  - mcp__mobile-mcp__mobile_get_screen_size
  - mcp__mobile-mcp__mobile_click_on_screen_at_coordinates
  - mcp__mobile-mcp__mobile_double_tap_on_screen
  - mcp__mobile-mcp__mobile_long_press_on_screen_at_coordinates
  - mcp__mobile-mcp__mobile_list_elements_on_screen
  - mcp__mobile-mcp__mobile_press_button
  - mcp__mobile-mcp__mobile_open_url
  - mcp__mobile-mcp__mobile_swipe_on_screen
  - mcp__mobile-mcp__mobile_type_keys
  - mcp__mobile-mcp__mobile_take_screenshot
  - mcp__mobile-mcp__mobile_save_screenshot
  - mcp__mobile-mcp__mobile_set_orientation
  - mcp__mobile-mcp__mobile_get_orientation
---

# Run YAML Mobile UI Test

Execute the YAML test specified in the argument using mobile-mcp tools.

## Arguments

- `<test-path>` - Path to test folder or YAML file (required)
  - **Folder format**: `tests/login-flow/` (contains `test.yaml`)
  - **File format**: `tests/login.test.yaml` (legacy)
- `--report` - Generate HTML/JSON report (optional)

## Output Format

### Step-by-Step Output

For each step, output detailed progress:

```
Running: {test.name}
────────────────────────────────────────

  [1/8] wait_for "Login"
        ✓ Found in 1.2s

  [2/8] tap "Email"
        ✓ Tapped at (540, 320)

  [3/8] type "user@example.com"
        ✓ Typed 16 characters

  [4/8] tap "Password"
        ✓ Tapped at (540, 420)

  [5/8] type "password123"
        ✓ Typed 11 characters

  [6/8] tap "Login"
        ✓ Tapped at (540, 580)

  [7/8] wait 2s
        ✓ Waited 2 seconds

  [8/8] verify_screen "Home screen"
        ✓ MATCH (confidence: 94%)

────────────────────────────────────────
✓ PASSED  (8/8 steps in 6.4s)
```

### On Failure

```
  [4/8] tap "Login Button"
        ✗ FAILED: Element not found

        Screenshot: tests/reports/screenshots/failure_004.png

        Elements on screen:
        - "Email" at (540, 320)
        - "Password" at (540, 420)
        - "Sign In" at (540, 580)
        - "Forgot Password" at (540, 650)

        Hint: Did you mean "Sign In"?

────────────────────────────────────────
✗ FAILED  (3/8 steps, failed at step 4)
```

### Test Summary

After all tests complete:

```
═══════════════════════════════════════
           TEST RESULTS
═══════════════════════════════════════

  ✓ User login flow                    6.4s
  ✓ Navigate to settings               3.2s
  ✗ Checkout process                   8.1s
    └─ Failed: Element "Pay Now" not found

───────────────────────────────────────
  Total:    3 tests
  Passed:   2
  Failed:   1
  Duration: 17.7s
═══════════════════════════════════════
```

## Execution Process

### 1. Parse Arguments

Check if `--report` flag is present. Parse test path.

### 2. Detect Test Format

Check if argument is a folder or file:

**If folder** (e.g., `tests/login-flow/`):
- Test file: `{folder}/test.yaml`
- Baselines: `{folder}/baselines/`
- Reports go to: `{folder}/reports/`

**If file** (e.g., `tests/login.test.yaml`):
- Legacy flat file format
- No baselines available
- Reports go to: `tests/reports/`

### 3. Read and Parse Test File

Read the YAML test file. Parse:
- `config` section: app package, device (optional)
- `setup` section: pre-test actions
- `teardown` section: post-test actions
- `tests` section: array of test cases

### 4. Device Selection

If `config.device` is specified, use that device.
Otherwise, call `mobile_list_available_devices` and:
- If one device: use it automatically
- If multiple: ask user to select
- If none: report error and stop

### 5. Initialize Report Data (if --report)

Create data structure to collect:
- Test results (pass/fail, duration)
- Step details (action, result, timing)
- Screenshots at key points
- Failure information

### 6. Execute Setup

Run all actions in `setup` section before any tests.
Report each step with the detailed format above.

### 7. Execute Each Test

For each test in `tests` array:
1. Report: "Running: {test.name}" with separator line
2. Track start time
3. Execute each step in `steps` array:
   - Report step number, action, and parameters
   - Execute the action
   - Report result (success with details, or failure with diagnostics)
   - On failure: capture screenshot, gather element list, stop test
4. Track end time
5. Report: PASSED or FAILED with step count and duration

### 8. Baseline Comparison (if available)

After each step that has a corresponding baseline:

1. Take screenshot of current state
2. Load baseline image from `baselines/step_{N}_*.png`
3. Compare images:
   - If similar (>90% match): PASS
   - If different: WARN with visual diff

Report baseline mismatches:
```
Step 3: tap "Login"
  ✓ Action completed
  ⚠ Visual mismatch with baseline
    Expected: baselines/step_03_dashboard.png
    Actual: reports/run_step_03.png
    Diff: 15% pixels different
```

**Note**: Baseline comparison is only available for folder-format tests that have a `baselines/` directory.

### 9. Execute Teardown

Run all actions in `teardown` section after all tests (even if tests failed).

### 10. Report Summary

Show the summary table with all test results.

### 11. Generate Report (if --report)

If `--report` flag was provided:

**For folder-format tests** (`tests/login-flow/`):
1. Create `{test-folder}/reports/` directory if needed
2. Generate timestamp: `YYYY-MM-DD_HH-MM-SS`
3. Save artifacts:
   ```
   {test-folder}/reports/
   ├── {timestamp}_run.html           ← Visual report
   ├── {timestamp}_run.json           ← Machine-readable
   └── {timestamp}_screenshots/       ← Screenshots from this run
       ├── step_01_login_screen.png
       ├── step_02_after_tap.png
       └── ...
   ```
4. Report: "Report saved: {test-folder}/reports/{timestamp}_run.html"

**For file-format tests** (legacy `tests/login.test.yaml`):
1. Create `tests/reports/` directory if needed
2. Generate timestamp: `YYYY-MM-DD_HH-MM-SS`
3. Save screenshots to `tests/reports/screenshots/`
4. Generate JSON report: `tests/reports/{timestamp}_{test-name}.json`
5. Generate HTML report: `tests/reports/{timestamp}_{test-name}.html`
6. Report: "Report saved: tests/reports/{filename}.html"

## Action Mapping

Map YAML actions to mobile-mcp tools:

| YAML Action | Execution |
|-------------|-----------|
| `tap: "X"` | `mobile_list_elements_on_screen` → find element → `mobile_click_on_screen_at_coordinates` |
| `tap: [x, y]` | `mobile_click_on_screen_at_coordinates` directly |
| `double_tap: "X"` | Find element → `mobile_double_tap_on_screen` |
| `long_press: "X"` | Find element → `mobile_long_press_on_screen_at_coordinates` |
| `type: "X"` | `mobile_type_keys` with text |
| `type: {text: "X", submit: true}` | `mobile_type_keys` with submit=true |
| `swipe: up/down/left/right` | `mobile_swipe_on_screen` with direction |
| `swipe: {direction: up, distance: N}` | `mobile_swipe_on_screen` with distance |
| `press: back/home/...` | `mobile_press_button` with button name |
| `wait: 2s` | Pause execution for duration |
| `launch_app` | `mobile_launch_app` with config.app |
| `launch_app: "pkg"` | `mobile_launch_app` with specified package |
| `terminate_app` | `mobile_terminate_app` with config.app |
| `set_orientation: X` | `mobile_set_orientation` |
| `open_url: "X"` | `mobile_open_url` |
| `screenshot: "name"` | `mobile_take_screenshot` and note the name |
| `save_screenshot: "/path"` | `mobile_save_screenshot` to file |

## Verification Actions

| YAML Action | Execution |
|-------------|-----------|
| `verify_screen: "X"` | `mobile_take_screenshot` → AI analysis against expectation |
| `verify_contains: "X"` | `mobile_list_elements_on_screen` → check element exists |
| `verify_no_element: "X"` | `mobile_list_elements_on_screen` → check element absent |

## Flow Control Actions

| YAML Action | Execution |
|-------------|-----------|
| `wait_for: "X"` | Poll `mobile_list_elements_on_screen` until element found (default 10s timeout) |
| `wait_for: {element: "X", timeout: 30s}` | Poll with custom timeout |
| `if_present: "X"` | Check `mobile_list_elements_on_screen`, execute `then` if found |
| `retry: {attempts: N, steps: [...]}` | Retry steps up to N times on failure |
| `repeat: {times: N, steps: [...]}` | Execute steps N times |

## Finding Elements

When an action references an element by text (e.g., `tap: "Continue"`):

1. Call `mobile_list_elements_on_screen` to get all elements
2. Search for element where text or content-description matches (case-insensitive)
3. If exact match not found, try partial match
4. Use the element's center coordinates for interaction
5. If not found after 3 retries with 500ms delay, fail with helpful message
6. Include list of available elements in failure message

## Error Handling

- On step failure: capture screenshot, log error, mark test as failed
- Continue to next test after failure (don't abort entire suite)
- Always run teardown even after failures
- Report clear error messages with:
  - What was expected
  - What was found (or not found)
  - Screenshot path
  - Available elements on screen
  - Suggestions (e.g., "Did you mean 'Sign In'?")

## Report Format

### JSON Report Structure

```json
{
  "testFile": "tests/login.test.yaml",
  "timestamp": "2025-01-09T16:30:00Z",
  "duration": 17.7,
  "summary": {
    "total": 3,
    "passed": 2,
    "failed": 1
  },
  "tests": [
    {
      "name": "User login flow",
      "status": "passed",
      "duration": 6.4,
      "steps": [
        {
          "action": "wait_for",
          "params": "Login",
          "status": "passed",
          "duration": 1.2,
          "details": "Found in 1.2s"
        }
      ]
    }
  ]
}
```

### HTML Report

Generate a styled HTML report with:
- Summary header with pass/fail counts
- Expandable test sections
- Inline screenshots (base64 encoded)
- Color-coded status (green=pass, red=fail)
- Timing information
- Failure details with element dumps

## Usage Examples

**Folder format (recommended):**
```
/run-test tests/login-flow/
/run-test tests/onboarding/ --report
/run-test tests/checkout/ --report
```

**File format (legacy):**
```
/run-test tests/login.test.yaml
/run-test tests/onboarding.test.yaml --report
```
