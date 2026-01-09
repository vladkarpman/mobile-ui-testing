---
name: generate-test
description: Generate a YAML test from natural language description
argument-hint: "<description>"
allowed-tools:
  - Write
  - Read
  - Glob
  - AskUserQuestion
---

# Generate YAML Test from Natural Language

Convert a natural language description into a complete YAML test file.

## Process

### 1. Parse the Description

Extract from the user's description:
- **Actions**: What the user does (tap, type, swipe, navigate)
- **Verifications**: What should be checked (screen state, elements)
- **Flow**: The sequence of steps

### 2. Detect App Package

Look for existing tests to find the app package:
1. Search `tests/*.test.yaml` for `config.app`
2. Check Android `app/build.gradle*` for applicationId
3. If not found, ask: "What is your app package name?"

### 3. Generate Test Name

Convert description to kebab-case name:
- "user logs in with email" → `user-logs-in-with-email`
- "checkout flow with credit card" → `checkout-flow-with-credit-card`

### 4. Generate YAML

Create a complete test file following this structure:

```yaml
# {Generated Test Name}
# Generated from: "{original description}"

config:
  app: {detected-package}

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: {Descriptive test name}
    description: {Original description}
    timeout: 60s
    steps:
      # Generated steps based on description
      - wait_for: "{first expected element}"
      - tap: "{element to interact with}"
      - type: "{text to enter}"
      - wait: 1s
      - verify_screen: "{expected outcome}"
      - screenshot: "{step_name}"
```

### 5. Step Generation Rules

**For navigation actions:**
- "go to X" / "navigate to X" / "open X" → `- tap: "X"`
- "scroll down" / "scroll up" → `- swipe: down` / `- swipe: up`

**For input actions:**
- "enter X in Y" / "type X" → `- tap: "Y"` then `- type: "X"`
- "fill in the form" → multiple tap + type pairs

**For verification:**
- "see X" / "verify X" / "check X" → `- verify_screen: "X"`
- "X appears" / "shows X" → `- wait_for: "X"` then `- verify_contains: "X"`

**For waiting:**
- "wait for X" → `- wait_for: "X"`
- Add `- wait: 1s` between major actions

**For conditional flows:**
- "if X appears, tap Y" → `- if_present: "X"` with `then: [- tap: "Y"]`

### 6. Save the File

Write to `tests/{generated-name}.test.yaml`

### 7. Report Success

```
Generated test: tests/{name}.test.yaml

Test: {test name}
Steps: {count} actions

Preview:
{show first 5 steps}

Run with: /run-test tests/{name}.test.yaml
```

## Examples

**Input:** "user logs in with email and password then sees home screen"

**Output:**
```yaml
config:
  app: com.example.app

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: User login flow
    description: user logs in with email and password then sees home screen
    timeout: 60s
    steps:
      - wait_for: "Login"
      - tap: "Email"
      - type: "user@example.com"
      - tap: "Password"
      - type: "password123"
      - tap: "Login"
      - wait: 2s
      - verify_screen: "Home screen after successful login"
      - screenshot: "login_success"
```

**Input:** "scroll through feed and like the first post"

**Output:**
```yaml
tests:
  - name: Like first post in feed
    description: scroll through feed and like the first post
    timeout: 60s
    steps:
      - wait_for: "Feed"
      - swipe: up
      - wait: 1s
      - tap: "Like"
      - wait: 500ms
      - verify_screen: "Post liked indicator visible"
      - screenshot: "post_liked"
```

## Tips for Users

Encourage descriptive prompts:
- Good: "user opens settings, enables dark mode, and returns to home"
- Less good: "test settings"

If description is too vague, ask for clarification:
- "What specific actions should be tested?"
- "What should happen at the end?"
