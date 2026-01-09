# Recording Flow Improvements Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement guided step-by-step recording with burst capture and organized test folder structure.

**Architecture:** Replace continuous polling with explicit user-Claude handshake. User says "done" after each action, Claude burst-captures 7 screenshots over 2s, analyzes for state changes, then prompts "Ready for next action".

**Tech Stack:** Claude Code plugin commands (markdown), mobile-mcp tools, YAML test format

---

## Task 1: Update record-test.md with Guided Flow

**Files:**
- Modify: `commands/record-test.md`

**Step 1: Replace the continuous monitoring section with guided flow**

Open `commands/record-test.md` and replace sections 6-9 (lines 66-142) with the new guided flow:

```markdown
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

### 7. Create Test Folder Structure

Before starting, create the folder structure:

```
tests/{test-name}/
├── screenshots/
├── baselines/
└── reports/
```

Use Bash to create directories:
```bash
mkdir -p tests/{test-name}/screenshots
mkdir -p tests/{test-name}/baselines
mkdir -p tests/{test-name}/reports
```

### 8. Guided Recording Loop

Wait for user to say "done" (or variations: "ok", "next", "ready", "finished").

When user signals done:

**Step 8.1: Burst Capture**

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

**Step 8.2: Analyze Screenshots**

Compare consecutive screenshots:
1. Diff element lists - what appeared/disappeared?
2. Identify distinct states (group similar screenshots)
3. Detect: tap (element gone), type (text changed), swipe (elements shifted), navigation (new screen)

**Step 8.3: Report to User**

```
Captured 7 screenshots, detected {N} states:
  1. {state_1_description}
  2. {state_2_description}
  ...

Inferred action: {action_type} "{target}" → {result}

Ready for next action.
```

**Step 8.4: Update Recording State**

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

**Step 8.5: Loop**

Repeat from step 8 until user says "stop recording" or "/stop-recording".

### 9. Handle User Messages During Recording

- "done" / "ok" / "next" / "finished" → trigger burst capture
- "stop" / "stop recording" / "/stop-recording" → end recording
- "undo" / "back" → remove last recorded action
- Any other message → treat as context hint, store with next action
```

**Step 2: Update the recording state schema (section 4)**

Replace the recording state JSON (around line 47) with:

```markdown
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
```

**Step 3: Commit changes**

```bash
git add commands/record-test.md
git commit -m "feat(record): implement guided step-by-step recording flow"
```

---

## Task 2: Update stop-recording.md for Folder Structure

**Files:**
- Modify: `commands/stop-recording.md`

**Step 1: Update YAML generation path**

Replace section 4 (around line 65) to use folder structure:

```markdown
### 4. Generate YAML Test File

Create `tests/{test-name}/test.yaml`:

```yaml
# {Test Name}
# Recorded on: {date}
# Duration: {recording duration}
# Actions: {action count}

config:
  app: {app-package}

setup:
  - terminate_app
  - launch_app
  - wait: 3s

teardown:
  - terminate_app

tests:
  - name: {test-name}
    description: Recorded test - review and add assertions
    timeout: {duration + 30}s
    steps:
      {generated steps from actions}
```
```

**Step 2: Add baseline creation option**

Add new section after section 5:

```markdown
### 6. Create Baselines (Optional)

Ask user: "Would you like to save key screenshots as baselines for future comparison?"

If yes:
1. For each unique final state in the recording, copy screenshot to `baselines/`
2. Name as `step_{N}_{description}.png`
3. These will be used by `/run-test` for visual validation

Example baselines:
```
tests/{test-name}/baselines/
├── step_01_home_screen.png
├── step_02_login_form.png
└── step_03_dashboard.png
```
```

**Step 3: Update cleanup section**

```markdown
### 7. Cleanup

1. Delete `.claude/recording-state.json`
2. Keep all screenshots in `tests/{test-name}/screenshots/`
3. Generate `tests/{test-name}/reports/recording-summary.html` with:
   - Timeline of actions
   - Thumbnail screenshots
   - Generated YAML preview
```

**Step 4: Update success report**

```markdown
### 8. Report Success

```
Recording complete!
══════════════════════════════════════════════

Generated: tests/{test-name}/test.yaml

Folder structure:
  tests/{test-name}/
  ├── test.yaml          ← Run with /run-test
  ├── screenshots/       ← {N} images captured
  ├── baselines/         ← {M} baseline images
  └── reports/           ← Recording summary

Actions recorded: {count}
Duration: {time}

Steps:
  1. tap: "Email"
  2. type: "user@example.com"
  3. tap: "Password"
  4. type: "********"
  5. tap: "Login"
  6. wait_for: "Dashboard"

══════════════════════════════════════════════

Next: Review test.yaml and run with /run-test tests/{test-name}
```
```

**Step 5: Commit changes**

```bash
git add commands/stop-recording.md
git commit -m "feat(stop-recording): update for folder structure and baselines"
```

---

## Task 3: Update run-test.md for Folder Detection

**Files:**
- Modify: `commands/run-test.md`

**Step 1: Add folder detection logic**

Add new section after argument parsing:

```markdown
### 2. Detect Test Format

Check if argument is a folder or file:

**If folder** (e.g., `tests/login-flow`):
- Test file: `{folder}/test.yaml`
- Baselines: `{folder}/baselines/`
- Reports go to: `{folder}/reports/`

**If file** (e.g., `tests/login.test.yaml`):
- Legacy flat file format
- No baselines available
- Reports go to: `tests/reports/`

```python
if path.is_dir():
    test_file = path / "test.yaml"
    baselines_dir = path / "baselines"
    reports_dir = path / "reports"
else:
    test_file = path
    baselines_dir = None
    reports_dir = path.parent / "reports"
```
```

**Step 2: Add baseline comparison**

Add new section for visual validation:

```markdown
### Baseline Comparison (if available)

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

Visual comparison uses screenshot dimensions and key element positions, not pixel-perfect matching.
```

**Step 3: Update report generation**

```markdown
### Report Generation

Save run artifacts to `{test-folder}/reports/`:

```
reports/
├── {timestamp}_run.html      ← Visual report
├── {timestamp}_run.json      ← Machine-readable
└── {timestamp}_screenshots/  ← Screenshots from this run
    ├── step_01.png
    ├── step_02.png
    └── ...
```

HTML report includes:
- Pass/fail status per step
- Side-by-side baseline vs actual (if baselines exist)
- Timing information
- Error details for failures
```

**Step 4: Commit changes**

```bash
git add commands/run-test.md
git commit -m "feat(run-test): add folder detection and baseline comparison"
```

---

## Task 4: Update README with New Structure

**Files:**
- Modify: `README.md`

**Step 1: Add test folder structure documentation**

Add new section after "Test File Structure":

```markdown
## Test Folder Structure (v2.0+)

New recordings create an organized folder structure:

```
tests/
└── login-flow/
    ├── test.yaml              # Test definition
    ├── screenshots/           # Captured during recording
    ├── baselines/             # Golden images for validation
    └── reports/               # Run results and history
```

### Recording with Guided Mode

The improved recording flow guides you step-by-step:

```
/record-test login-flow

> Recording started. Do your first action, then say "done".

*tap Login button*
> done

> Captured 3 states: Home → Loading → Login Form
> Ready for next action.

*type email*
> done

> Captured: type in Email field
> Ready for next action.

...

> stop

> Recording complete! Generated tests/login-flow/test.yaml
```

### Baseline Visual Validation

After recording, save key screenshots as baselines:

```
tests/login-flow/baselines/
├── step_01_home.png
├── step_02_login_form.png
└── step_03_dashboard.png
```

Future test runs compare against baselines and warn on visual differences.
```

**Step 2: Commit changes**

```bash
git add README.md
git commit -m "docs: add test folder structure and guided recording docs"
```

---

## Task 5: Test the New Recording Flow

**Step 1: Start a test recording**

```bash
# In Claude Code with the plugin loaded
/record-test test-guided-flow
```

**Step 2: Verify folder creation**

Check that `tests/test-guided-flow/` was created with subdirectories.

**Step 3: Perform guided recording**

1. Do an action on device
2. Say "done"
3. Verify burst capture happens (7 screenshots)
4. Verify "Ready for next action" prompt
5. Repeat 2-3 times

**Step 4: Stop recording**

Say "stop recording" or `/stop-recording`

**Step 5: Verify output**

- `tests/test-guided-flow/test.yaml` exists
- `tests/test-guided-flow/screenshots/` has captured images
- Recording summary displayed

**Step 6: Run the generated test**

```bash
/run-test tests/test-guided-flow
```

**Step 7: Commit if all works**

```bash
git add -A
git commit -m "test: verify new recording flow works"
git push
```

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | Guided recording flow | `commands/record-test.md` |
| 2 | Folder structure output | `commands/stop-recording.md` |
| 3 | Folder detection + baselines | `commands/run-test.md` |
| 4 | Documentation | `README.md` |
| 5 | End-to-end testing | Manual verification |

**Total estimated time:** 30-45 minutes
