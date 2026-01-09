---
name: run-test
description: Execute a YAML mobile UI test file on a connected device
argument-hint: <test-file.yaml>
allowed-tools:
  - Read
  - Bash
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

Execute the YAML test file specified in the argument using mobile-mcp tools.

## Execution Process

### 1. Read and Parse Test File

Read the YAML test file provided as argument. Parse:
- `config` section: app package, device (optional)
- `setup` section: pre-test actions
- `teardown` section: post-test actions
- `tests` section: array of test cases

### 2. Device Selection

If `config.device` is specified, use that device.
Otherwise, call `mobile_list_available_devices` and:
- If one device: use it
- If multiple: ask user to select
- If none: report error and stop

### 3. Execute Setup

Run all actions in `setup` section before any tests.

### 4. Execute Each Test

For each test in `tests` array:
1. Report: "Running: {test.name}"
2. Execute each step in `steps` array
3. Track pass/fail status
4. On failure: capture screenshot, report error, skip remaining steps
5. Report: "PASSED" or "FAILED: {reason}"

### 5. Execute Teardown

Run all actions in `teardown` section after all tests (even if tests failed).

### 6. Report Summary

```
Test Results
============
✓ Test name 1
✓ Test name 2
✗ Test name 3 - Failed: Element "X" not found

Passed: 2/3
Failed: 1/3
```

## Action Mapping

Map YAML actions to mobile-mcp tools:

| YAML Action | Execution |
|-------------|-----------|
| `tap: "X"` | `mobile_list_elements_on_screen` → find element by text → `mobile_click_on_screen_at_coordinates` |
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
2. Search for element where text or content-description matches
3. Use the element's center coordinates for interaction
4. If not found, wait briefly and retry (up to 3 times)
5. If still not found, fail with "Element 'X' not found on screen"

## Error Handling

- On step failure: capture screenshot, log error, mark test as failed
- Continue to next test after failure (don't abort entire suite)
- Always run teardown even after failures
- Report clear error messages with context

## Usage Examples

```
/run-test tests/mcp/onboarding.test.yaml
/run-test tests/mcp/home-navigation.test.yaml
```
