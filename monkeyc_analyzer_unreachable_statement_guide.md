# Monkey C Static Analyzer — "Statement is not reachable" Warning
## Root Cause Analysis and Permanent Solutions

**Date:** February 18, 2026
**SDK Version:** 8.4.1
**Device:** Venu 3
**Project:** StepDistanceField (initial discovery)
**Status:** Resolved

This document captures hard-won knowledge about the Monkey C compiler's static flow analyzer, which produces "Statement is not reachable" warnings that are difficult to diagnose because the flagged line shifts with every code change, creating a whack-a-mole effect.

---

## Numbered Index

1. [The Core Problem](#1-the-core-problem)
2. [How the Analyzer Works](#2-how-the-analyzer-works)
3. [Patterns That Trigger the Warning](#3-patterns-that-trigger-the-warning)
4. [Solutions That Work](#4-solutions-that-work)
5. [Solutions That Do NOT Work](#5-solutions-that-do-not-work)
6. [Diagnostic Technique](#6-diagnostic-technique)
7. [Activity.Info Timer State Reference](#7-activityinfo-timer-state-reference)
8. [Non-Nullable API Methods](#8-non-nullable-api-methods)
9. [Checklist — Preventing Future Warnings](#9-checklist--preventing-future-warnings)
10. [Summary](#10-summary)

---

## 1. The Core Problem

The Monkey C compiler includes an aggressive interprocedural static analyzer that traces member variable values forward from `initialize()`. When it can **prove** that one branch of a conditional is impossible based on those traced values, it flags the dead branch as "Statement is not reachable."

The warning is benign (build succeeds) but achieving a zero-warning build requires understanding how the analyzer thinks.

---

## 2. How the Analyzer Works

### What It Does

1. Starts at `initialize()` and records the concrete values assigned to every member variable
2. Propagates those known values into every function reachable from the initialization path
3. Evaluates conditionals against the known values
4. Flags any branch it can prove will never execute

### What It Does NOT Do

- Does NOT account for runtime state changes from lifecycle callbacks (`onTimerStart`, `compute`, etc.)
- Does NOT treat the call graph dynamically — it doesn't know that `compute()` runs repeatedly and modifies members before other functions read them
- Does NOT see through opaque external values (function parameters, nullable API returns)

### Example Trace

```monkeyc
private var _running as Boolean;

function initialize() {
    _running = false;          // Analyzer records: _running = false
}

function compute(info as Activity.Info) as Void {
    _running = true;           // Analyzer ignores this runtime change
    doWork();
}

function doWork() as Void {
    if (_running) {            // Analyzer: _running is false (from init)
        // This code...        // FLAGGED: "Statement is not reachable"
    }
}
```

---

## 3. Patterns That Trigger the Warning

### Pattern 1: Boolean Member as Branch Condition

```monkeyc
private var _flag as Boolean;
function initialize() { _flag = false; }
// ...
if (_flag) { /* flagged */ }   // Analyzer proves: always false
```

### Pattern 2: Number Member Compared Against Init Value

```monkeyc
private var _count as Number;
function initialize() { _count = 0; }
// ...
if (_count > 0) { /* flagged */ }   // Analyzer proves: 0 > 0 is false
```

### Pattern 3: Non-Nullable API Return with Null Guard

```monkeyc
var amInfo = ActivityMonitor.getInfo();  // Returns Info (non-nullable)
if (amInfo == null) {
    return;   // FLAGGED: return type says non-nullable, so null is impossible
}
```

### Pattern 4: System.getTimer() Subtraction Guard

```monkeyc
var deltaMs = System.getTimer() - someTimestamp;  // getTimer() returns positive ms
if (deltaMs <= 0) {
    return 0.0f;  // FLAGGED if someTimestamp was 0 from init
}
```

### Pattern 5: Arithmetic on Init-Traced Values

```monkeyc
private var _bufFull as Number;   // 0 or 1
function initialize() { _bufFull = 0; }

var count = _bufFull * (SIZE - idx) + idx;  // Analyzer: 0 * anything + idx = idx
if (count > idx) { /* flagged */ }
```

Even replacing Boolean with Number flags doesn't help — the analyzer evaluates arithmetic on known constants.

---

## 4. Solutions That Work

### Solution 1: Use Function Parameters Instead of Members (Best)

The analyzer **cannot** resolve the runtime value of function parameters or nullable API returns.

```monkeyc
// BEFORE (triggers warning):
private var _timerRunning as Boolean;
function initialize() { _timerRunning = false; }
function compute(info as Activity.Info) as Void {
    if (_timerRunning) { /* flagged */ }
}

// AFTER (clean build):
function compute(info as Activity.Info) as Void {
    // info.timerState is nullable (Activity.TimerState or Null) — opaque to analyzer
    if (info.timerState != null && info.timerState == Activity.TIMER_STATE_ON) {
        // safe — analyzer can't prove this branch dead
    }
}
```

**Key insight:** `Activity.Info` fields like `timerState` are declared nullable (`TimerState or Null`), so the analyzer can't determine their value statically. This makes them safe branch conditions.

### Solution 2: Branchless Arithmetic (For Divide-by-Zero Guards)

```monkeyc
// BEFORE (triggers warning — analyzer proves deltaMs > 0):
if (deltaMs <= 0) {
    return 0.0f;   // FLAGGED
}
return value / deltaMs;

// AFTER (clean build):
return value / (deltaMs.toFloat() + 1.0f);  // +1ms prevents div/0, ~0.007% error
```

**When appropriate:** The +1ms error is negligible for display-only values like speed in mph. Don't use for precision-critical calculations.

### Solution 3: Remove Provably-Unnecessary Null Guards

If the SDK docs declare a return type as non-nullable, don't null-check it.

```monkeyc
// BEFORE (triggers warning):
var info = ActivityMonitor.getInfo();  // Returns Info (non-nullable)
if (info == null) { return; }         // FLAGGED

// AFTER (clean build):
var info = ActivityMonitor.getInfo();  // Use directly
var steps = info.steps;               // .steps IS nullable — this null check is fine
```

How to check: look at the SDK HTML docs. If return type says `as Info` (no `or Null`), it's non-nullable. If it says `as Info or Null`, a null guard is appropriate.

---

## 5. Solutions That Do NOT Work

| Approach | Why It Fails |
|----------|-------------|
| Replace Boolean with Number (0/1) flag | Analyzer evaluates arithmetic on known constants |
| Use ternary instead of if/else | Analyzer resolves ternaries the same way |
| Move the check to a separate function | Analyzer does interprocedural analysis |
| Initialize member in `onTimerStart` instead | Analyzer still traces from `initialize()` first |
| Pre-seed buffer to avoid needing a flag | Removes one warning but exposes the next underneath |
| Add a loop with conditional break | Analyzer proves loop condition is always false |

---

## 6. Diagnostic Technique

The warning only gives a line number. When fixing one warning reveals another, use this process:

1. **Read the exact line indicated** — it's the first statement inside a branch the analyzer proved dead
2. **Look at the enclosing `if` condition** — what variable is being tested?
3. **Trace that variable back to `initialize()`** — what value was it assigned?
4. **Ask:** Can the analyzer prove this branch is impossible given the init value?
5. **Check SDK docs** for the return type of any API call in the condition — is it nullable?

---

## 7. Activity.Info Timer State Reference

Use `info.timerState` instead of Boolean member variables to track timer state:

```monkeyc
// Available in compute(info as Activity.Info):
info.timerState  // Activity.TimerState or Null

// Constants:
Activity.TIMER_STATE_OFF      // 0 — no active recording
Activity.TIMER_STATE_STOPPED  // 1 — recording active, timer stopped
Activity.TIMER_STATE_PAUSED   // 2 — recording active, timer paused (e.g. auto-pause)
Activity.TIMER_STATE_ON       // 3 — recording active, timer running

// Pattern for "only when timer is actively running":
if (info.timerState != null && info.timerState == Activity.TIMER_STATE_ON) {
    // Timer is running
}
```

---

## 8. Non-Nullable API Methods

These methods return non-nullable types — do NOT null-guard the return value:

| Method | Return Type |
|--------|------------|
| `ActivityMonitor.getInfo()` | `ActivityMonitor.Info` |
| `System.getTimer()` | `Number` (positive ms since boot) |
| `System.getClockTime()` | `ClockTime` |
| `System.getDeviceSettings()` | `DeviceSettings` |

Always check the SDK HTML docs at `doc/Toybox/` for authoritative type signatures.

---

## 9. Checklist — Preventing Future Warnings

Before building, audit your code:

- [ ] No Boolean members used as `if` conditions — use parameter-based checks instead
- [ ] No Number members compared against their init value in branches
- [ ] No null guards on non-nullable API returns — check SDK docs for return types
- [ ] No divide-by-zero guards where the analyzer can prove the divisor is positive — use branchless arithmetic
- [ ] No defensive early-returns that the analyzer can prove unreachable

---

## 10. Summary

The Monkey C analyzer is an aggressive constant-propagation engine that traces from `initialize()` without modeling runtime lifecycle.

**The universal fix pattern:** Never branch on values the analyzer can resolve from initialization. Branch on values from external sources (parameters, nullable API returns) that the analyzer treats as opaque.

The warning shifts location each time a fix is applied because fixing one branch exposes the next one — this is normal. Work through them systematically from the top of the warning list.

---

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Non-nullable API list, `Activity.Info` timer state constants
- `consolidated_field_development_lessons.md` — Original project where these warnings were discovered (indoor step tracking)
- `timerhr_field_development_lessons.md` — HR zone null-guard pattern using local variable to avoid analyzer warning
- `connectiq_code_patterns.md` — Code patterns that avoid triggering warnings

---

*Last updated: February 2026 — SDK 8.4.1*
