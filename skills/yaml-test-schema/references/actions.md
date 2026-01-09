# YAML Test Actions Reference

Complete reference for all basic actions in YAML mobile tests.

## Tap Actions

### Tap by Description
```yaml
- tap: "Button text"
```
Finds element by text/description and taps it.

### Tap by Coordinates
```yaml
- tap: [100, 200]
```
Taps at specific x, y coordinates (pixels from top-left).

### Double Tap
```yaml
- double_tap: "Element"
- double_tap: [100, 200]
```

### Long Press
```yaml
# Default duration (500ms)
- long_press: "Element"

# Custom duration
- long_press:
    element: "Element"
    duration: 2000  # milliseconds
```

## Type Actions

### Basic Type
```yaml
- type: "Hello world"
```
Types text into currently focused field.

### Type with Submit
```yaml
- type:
    text: "Search query"
    submit: true  # Press enter after typing
```

## Swipe Actions

### Simple Swipe
```yaml
- swipe: up      # up, down, left, right
- swipe: down
- swipe: left
- swipe: right
```

### Swipe with Distance
```yaml
- swipe:
    direction: up
    distance: 800  # pixels
```

## Button Press

```yaml
- press: back        # Android back button
- press: home        # Home button
- press: volume_up   # Volume up
- press: volume_down # Volume down
- press: enter       # Enter/return key
```

## Wait Actions

```yaml
- wait: 2s      # 2 seconds
- wait: 500ms   # 500 milliseconds
- wait: 1m      # 1 minute
```

## App Control

### Launch App
```yaml
# Launch configured app (from config.app)
- launch_app

# Launch specific app
- launch_app: "com.other.app"
```

### Terminate App
```yaml
# Stop configured app
- terminate_app

# Stop specific app
- terminate_app: "com.other.app"
```

### Set Orientation
```yaml
- set_orientation: portrait
- set_orientation: landscape
```

### Open URL
```yaml
- open_url: "https://example.com"
```

## Screenshot Actions

### Capture Screenshot
```yaml
- screenshot: "descriptive_name"
```
Takes screenshot and labels it for reference.

### Save to File
```yaml
- save_screenshot: "/path/to/file.png"
```

## Mobile-MCP Tool Mapping

| YAML Action | mobile-mcp Tool |
|-------------|-----------------|
| `tap: "X"` | `mobile_list_elements_on_screen` + `mobile_click_on_screen_at_coordinates` |
| `tap: [x,y]` | `mobile_click_on_screen_at_coordinates` |
| `double_tap` | `mobile_double_tap_on_screen` |
| `long_press` | `mobile_long_press_on_screen_at_coordinates` |
| `type` | `mobile_type_keys` |
| `swipe` | `mobile_swipe_on_screen` |
| `press` | `mobile_press_button` |
| `launch_app` | `mobile_launch_app` |
| `terminate_app` | `mobile_terminate_app` |
| `set_orientation` | `mobile_set_orientation` |
| `open_url` | `mobile_open_url` |
| `screenshot` | `mobile_take_screenshot` |
| `save_screenshot` | `mobile_save_screenshot` |
