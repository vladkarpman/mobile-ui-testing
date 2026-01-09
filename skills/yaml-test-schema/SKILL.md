# YAML Mobile Test Schema

This skill should be used when the user wants to "write mobile UI tests", "create YAML tests", "run mobile tests", "test my app", asks about "YAML test syntax", "test actions", "verify_screen", "tap", "swipe", or any mobile UI testing related queries. It provides complete knowledge of the YAML-based mobile testing format used with mobile-mcp.

## Overview

YAML mobile tests are structured files that Claude Code interprets and executes via mobile-mcp tools. No compilation required - just write YAML and run.

## Test File Structure

```yaml
config:
  app: com.your.app.package    # Required: Android package or iOS bundle ID
  device: emulator-5554         # Optional: auto-detect if omitted

setup:                          # Optional: runs once before all tests
  - terminate_app
  - launch_app
  - wait: 3s

teardown:                       # Optional: runs once after all tests
  - terminate_app

tests:
  - name: Test name             # Required
    description: What it tests  # Optional
    timeout: 60s                # Optional: default 60s
    tags: [smoke, critical]     # Optional: for filtering
    steps:
      - tap: "Button"
      - verify_screen: "Expected state"
```

## Quick Action Reference

### Basic Actions
| Action | Example |
|--------|---------|
| Tap element | `- tap: "Continue"` |
| Tap coordinates | `- tap: [100, 200]` |
| Type text | `- type: "hello"` |
| Swipe | `- swipe: up` |
| Press button | `- press: back` |
| Wait | `- wait: 2s` |

### App Control
| Action | Example |
|--------|---------|
| Launch app | `- launch_app` |
| Stop app | `- terminate_app` |
| Set orientation | `- set_orientation: landscape` |

### Verification
| Action | Example |
|--------|---------|
| Verify screen | `- verify_screen: "Home with tabs"` |
| Check elements | `- verify_contains: ["A", "B"]` |
| Check absence | `- verify_no_element: "Error"` |

### Flow Control
| Action | Example |
|--------|---------|
| Wait for element | `- wait_for: "Continue"` |
| Conditional | `- if_present: "Skip"` + `then: [...]` |
| Retry | `- retry: {attempts: 3, steps: [...]}` |
| Repeat | `- repeat: {times: 5, steps: [...]}` |

## Executing Tests

When executing YAML tests, map each action to mobile-mcp tools:

| YAML Action | mobile-mcp Tools |
|-------------|------------------|
| `tap: "X"` | `mobile_list_elements_on_screen` → find element → `mobile_click_on_screen_at_coordinates` |
| `tap: [x, y]` | `mobile_click_on_screen_at_coordinates` directly |
| `type: "X"` | `mobile_type_keys` |
| `swipe: up` | `mobile_swipe_on_screen` with direction |
| `press: back` | `mobile_press_button` |
| `wait: 2s` | Pause execution |
| `launch_app` | `mobile_launch_app` with config.app |
| `terminate_app` | `mobile_terminate_app` with config.app |
| `verify_screen: "X"` | `mobile_take_screenshot` → analyze image against expectation |
| `wait_for: "X"` | Poll `mobile_list_elements_on_screen` until element found |
| `if_present: "X"` | Check `mobile_list_elements_on_screen`, execute `then` if found |
| `screenshot: "name"` | `mobile_take_screenshot` and note the name |

## Detailed References

For complete documentation of each action type:

- **Basic Actions**: See `references/actions.md`
- **Assertions**: See `references/assertions.md`
- **Flow Control**: See `references/flow-control.md`

## Best Practices

1. **Add waits after actions** - UI needs time to respond
2. **Use `if_present` for optional elements** - Popups may or may not appear
3. **Set appropriate timeouts** - AI generation needs minutes, not seconds
4. **Capture screenshots** - Document test execution
5. **Use descriptive verifications** - Be specific about expected screen state
