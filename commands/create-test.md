---
name: create-test
description: Create a new YAML test file with proper structure
argument-hint: <test-name>
allowed-tools:
  - Write
  - Read
  - Glob
  - AskUserQuestion
---

# Create YAML Mobile UI Test

Generate a new YAML test file with proper structure based on the test name argument.

## Process

### 1. Determine Test Name

Use the argument as the test name. If no argument provided, ask:
"What would you like to name this test? (e.g., login, onboarding, checkout)"

### 2. Detect App Package

Try to detect the app package:
1. Look for existing test files in `tests/*.test.yaml`
2. Check for `config.app` in any existing test
3. Look for Android manifest or build.gradle for package name

If not found, ask:
"What is the app package name? (e.g., com.example.app)"

### 3. Ask About Test Purpose

Ask: "What does this test verify?"
Options:
- Navigation flow
- Form input/submission
- Feature functionality
- Error handling
- Other (describe)

### 4. Generate Test File

Create file at `tests/{test-name}.test.yaml`:

```yaml
# {Test Name} Tests
# Generated test file - customize as needed

config:
  app: {detected-or-provided-package}

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: {Test purpose from user input}
    description: Add description here
    timeout: 60s
    tags: []
    steps:
      # Add your test steps here
      # Examples:
      # - tap: "Button text"
      # - type: "Input text"
      # - swipe: up
      # - wait: 1s
      # - verify_screen: "Expected state"
      - wait: 1s
      - screenshot: "initial_state"
```

### 5. Report Success

Report:
```
Created test file: tests/{test-name}.test.yaml

Next steps:
1. Edit the file to add your test steps
2. Run with: /run-test tests/{test-name}.test.yaml

Quick reference:
- tap: "Button"     - Tap element by text
- type: "text"      - Type into focused field
- swipe: up         - Swipe in direction
- wait: 2s          - Wait for duration
- verify_screen: "" - Verify screen state
- screenshot: "x"   - Capture screenshot
```

## Template Variations

### Navigation Test Template
```yaml
tests:
  - name: Navigate to {destination}
    steps:
      - wait_for: "Home"
      - tap: "{destination}"
      - wait: 2s
      - verify_screen: "{destination} screen displayed"
      - screenshot: "{destination}_screen"
```

### Form Test Template
```yaml
tests:
  - name: Submit {form} form
    steps:
      - wait_for: "{form field}"
      - tap: "{field 1}"
      - type: "{value 1}"
      - tap: "{field 2}"
      - type: "{value 2}"
      - tap: "Submit"
      - wait_for: "Success"
      - verify_screen: "Form submitted successfully"
```

### Feature Test Template
```yaml
tests:
  - name: {Feature} works correctly
    steps:
      - wait_for: "{Feature button}"
      - tap: "{Feature button}"
      - wait: 2s
      - verify_screen: "{Feature} activated"
      - screenshot: "{feature}_active"
```

## Directory Structure

Always create tests in `tests/` directory:
```
tests/
├── onboarding.test.yaml
├── login.test.yaml
├── {new-test}.test.yaml
└── ...
```

Create the `tests/` directory if it doesn't exist.

## Usage Examples

```
/create-test login
/create-test checkout-flow
/create-test profile-settings
```
