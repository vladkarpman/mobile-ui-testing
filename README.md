# Mobile UI Testing Plugin

A Claude Code plugin for writing and running YAML-based mobile UI tests.

## Features

- **YAML Test Schema**: Knowledge of the complete test format
- **`/run-test`**: Execute YAML test files on real devices
- **`/create-test`**: Scaffold new test files with proper structure

## Prerequisites

- Claude Code CLI
- mobile-mcp configured and connected to a device
- Android device (via adb) or iOS simulator

## Installation

```bash
claude --plugin-dir /path/to/mobile-ui-testing
```

Or add to your project's `.claude/plugins/` directory.

## Usage

### Run a Test

```
/run-test tests/mcp/onboarding.test.yaml
```

### Create a New Test

```
/create-test login
```

### Ask About Test Syntax

```
How do I write a mobile UI test?
What actions are available in YAML tests?
How do I verify a screen matches an expectation?
```

## Test File Format

```yaml
config:
  app: com.your.app.package

setup:
  - terminate_app
  - launch_app
  - wait: 3s

tests:
  - name: User can login
    steps:
      - tap: "Email"
      - type: "user@example.com"
      - tap: "Password"
      - type: "secret123"
      - tap: "Login"
      - verify_screen: "Home screen after login"
```

## Documentation

- [YAML Schema Reference](skills/yaml-test-schema/references/actions.md)
- [Assertions](skills/yaml-test-schema/references/assertions.md)
- [Flow Control](skills/yaml-test-schema/references/flow-control.md)

## License

MIT
