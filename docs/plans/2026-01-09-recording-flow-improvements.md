# Recording Flow Improvements Design

**Date:** 2026-01-09
**Status:** Approved

## Problem

The current recording flow has two issues:
1. **Misses intermediate states** - Loading screens, dialogs, and transitions are skipped because polling is too slow
2. **Timing mismatch** - User acts while Claude is still processing previous screenshot, causing missed actions

## Solution

Replace continuous polling with a **guided step-by-step flow** using **burst capture**.

## Recording Flow

```
┌─────────────────────────────────────────────────────────┐
│  /record-test login-flow                                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Claude: "Recording started. Do your first action,  │
│              then say 'done'"                           │
│                                                         │
│  2. User: *taps Login button on device*                │
│     User: "done"                                        │
│                                                         │
│  3. Claude: *takes 5-10 rapid screenshots*             │
│             *analyzes for changes*                      │
│             "Captured: tap 'Login' → loading → home"    │
│             "Ready for next action"                     │
│                                                         │
│  4. User: *types email*                                │
│     User: "done"                                        │
│                                                         │
│  5. Claude: *burst capture*                            │
│             "Captured: type in Email field"             │
│             "Ready for next action"                     │
│                                                         │
│  ... continues until /stop-recording ...                │
└─────────────────────────────────────────────────────────┘
```

### Key Principles

- **Explicit handshake** - User says "done", Claude captures, then says "Ready"
- **No race conditions** - Clear synchronization between user and Claude
- **Catches transitions** - Burst capture gets loading states and dialogs

## Burst Capture Mechanism

When user says "done", execute rapid screenshot sequence:

```
Burst Capture (2-3 seconds total)
─────────────────────────────────
  t=0ms     → screenshot_001.png + elements
  t=300ms   → screenshot_002.png + elements
  t=600ms   → screenshot_003.png + elements
  t=900ms   → screenshot_004.png + elements
  t=1200ms  → screenshot_005.png + elements
  t=1500ms  → screenshot_006.png + elements
  t=2000ms  → screenshot_007.png + elements (final)
```

### Analysis Phase

1. Compare consecutive screenshots for changes
2. Identify distinct states (e.g., "button pressed" → "loading spinner" → "new screen")
3. Map each state to a YAML action
4. Discard duplicate/similar screenshots, keep unique states

### Output to User

```
Captured 7 screenshots, detected 3 states:
  1. Home screen (before action)
  2. Loading spinner (300ms)
  3. Profile screen (final)

Inferred action: tap "Profile" → wait → verify "Profile screen"

Ready for next action.
```

## Test Folder Structure

Replace flat YAML files with organized folders:

```
tests/
└── login-flow/
    ├── test.yaml                    # Test definition
    │
    ├── screenshots/                 # All captured during recording
    │   ├── 001_home.png
    │   ├── 002_loading.png
    │   ├── 003_profile.png
    │   └── ...
    │
    ├── baselines/                   # Golden images for validation
    │   ├── step_01_home.png         # Expected state after step 1
    │   ├── step_02_profile.png      # Expected state after step 2
    │   └── ...
    │
    └── reports/                     # Run results
        ├── 2026-01-09_20-45.html    # HTML report with screenshots
        └── 2026-01-09_20-45.json    # Machine-readable results
```

### Backwards Compatibility

- Existing `tests/*.test.yaml` files continue to work
- New recordings create folder structure automatically
- `/run-test tests/login-flow` detects folder and uses `test.yaml` inside

### Baseline Workflow

1. First successful run → screenshots saved to `baselines/`
2. Future runs → compare against baselines
3. Mismatch → test fails with visual diff

## Command Changes

### `/record-test {name}`

1. Create folder: `tests/{name}/`
2. Create `screenshots/` subfolder
3. Launch app, capture initial state
4. Enter guided loop:
   - Prompt: "Do your action, say 'done' when ready"
   - Wait for user "done"
   - Burst capture (7 shots over 2s)
   - Save to `screenshots/001_*.png`, `002_*.png`, ...
   - Analyze and report what was captured
   - Prompt: "Ready for next action"
   - Repeat until `/stop-recording`

### `/stop-recording`

1. Generate `tests/{name}/test.yaml` from captured actions
2. Optionally create `baselines/` from key screenshots
3. Generate summary report in `reports/`
4. Display generated YAML + action summary

### `/run-test tests/{name}`

1. Detect folder vs flat file
2. If folder: read `test.yaml`, use `baselines/` for comparison
3. Save run screenshots to `reports/`
4. Generate HTML report with pass/fail per step
5. Visual diff if baseline mismatch

## Implementation Tasks

1. Update `commands/record-test.md` with guided flow
2. Update `commands/stop-recording.md` for folder structure
3. Update `commands/run-test.md` for folder detection and baselines
4. Create folder structure utilities
5. Implement burst capture logic
6. Implement baseline comparison
7. Update README with new structure

## Not In Scope (YAGNI)

- Background agents for capture (simpler synchronous approach works)
- Video recording (screenshots sufficient)
- Cloud storage for screenshots (local only)
- Automatic baseline updates (manual review required)
