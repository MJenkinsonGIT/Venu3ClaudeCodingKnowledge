# Consolidated Data Field — Development Lessons
## Real-World Development Experience — February 2026

**Project:** ConsolidatedField for Garmin Venu 3
**SDK Version:** 8.4.1 / API Level 5.2
**Status:** Complete and functional

This document captures practical lessons learned while building a high-density dual-triangle data field combining activity steps, distance, speed, intensity minutes, time, and daily step progress into a single data field slot.

---

## Numbered Index

1. [The 2-App Limit Problem and Consolidation Strategy](#1-the-2-app-limit-problem-and-consolidation-strategy)
2. [Dual-Triangle Layout Design](#2-dual-triangle-layout-design)
3. [Position-Aware Rendering via getObscurityFlags()](#3-position-aware-rendering-via-getobscurityflags)
4. [Bezel-Aware Horizontal and Vertical Positioning](#4-bezel-aware-horizontal-and-vertical-positioning)
5. [Iterative Label and Unit Offset Tuning](#5-iterative-label-and-unit-offset-tuning)
6. [Shifting the Entire Layout as a Unit](#6-shifting-the-entire-layout-as-a-unit)
7. [The Steps/Distance Zero Bug — Wrong API](#7-the-stepsdistance-zero-bug--wrong-api)
8. [App Name vs. PRG Filename](#8-app-name-vs-prg-filename)
9. [Final Working Layout Values](#9-final-working-layout-values)
10. [Complete Compute Pattern for Steps and Distance](#10-complete-compute-pattern-for-steps-and-distance)

---

## 1. The 2-App Limit Problem and Consolidation Strategy

### Discovery

The Venu 3 enforces a hard limit of **2 concurrent Connect IQ app instances** running during an activity. This means no matter how many data fields are installed, only 2 can be active simultaneously — and each app instance occupies one of those 2 slots regardless of how many field slots on screen it occupies.

### Strategy

Design one data field to consolidate related metrics into a spatially coherent layout. Use the full height of the slot deliberately — a dual-triangle arrangement naturally organizes two groups of three values (apex + two base items) while the header row carries global context (time, daily steps).

A single 1-of-4 slot can show 6 or more distinct metrics, effectively doubling information density compared to using 2 single-value fields.

---

## 2. Dual-Triangle Layout Design

### Concept

Two invisible triangles share the slot:

```
         [  TIME  ]  [STEPS / GOAL]

   [Steps]                    [Tot]        ← Apexes

   [Dist] [Speed]       [Mod] [Vig]        ← Bases
     mi     mph
```

Left triangle: activity-session metrics (steps → distance, speed)
Right triangle: intensity minutes (total → moderate, vigorous)
Header: time of day and daily step progress

### Why Triangles Work

- The single apex item sits closer to center horizontally — safe from the bezel curve
- The two base items spread outward but can be tuned to stay within the safe area
- Visual grouping communicates hierarchy: apex = primary metric, base = its decomposition

---

## 3. Position-Aware Rendering via getObscurityFlags()

### The Problem

A data field placed in the top half of a 2-field layout has its header at the top and detail rows below. The same field in the bottom half needs to be mirrored vertically — otherwise detail rows are nearest the center line and the header is near the outer bezel.

### The Solution

`WatchUi.DataField` exposes `getObscurityFlags()` which returns a bitmask indicating which edges are obscured. The constants `OBSCURE_TOP` and `OBSCURE_BOTTOM` detect which slot you're in.

```monkeyc
var flags     = getObscurityFlags();
var inBotSlot = (flags & OBSCURE_BOTTOM) != 0;
var topLayout = !inBotSlot;
```

**Key insight:** Check for `OBSCURE_BOTTOM` rather than `OBSCURE_TOP`. The bottom slot reliably has `OBSCURE_BOTTOM` set.

### Mirroring the Layout

```monkeyc
if (topLayout) {
    yTime       = (h *  4) / 100;
    yDailySteps = (h * 19) / 100;
    yPeak       = (h * 42) / 100;
    yBase       = (h * 69) / 100;
    labelOff    = -22;   // labels sit ABOVE their value anchor
} else {
    yTime       = (h * 76) / 100;
    yDailySteps = (h * 61) / 100;
    yPeak       = (h * 40) / 100;
    yBase       = (h * 13) / 100;
    labelOff    = 36;    // labels sit BELOW their value anchor
}
```

`labelOff` being positive in the bottom layout flips the label from above-value to below-value, so the label always points away from the center of the screen.

---

## 4. Bezel-Aware Horizontal and Vertical Positioning

### The Circular Bezel Interaction

Horizontal and vertical position interact on a round display: a value pushed far to the left or right is closer to the curved edge, and must also be moved toward the vertical center to remain inside the visible circle.

### Base Column Positions

Initial attempts at 25%/75% clipped against the bezel. Pulling columns inward to 16%/38% (left) and 62%/84% (right) of total width provided safe margin with adequate separation:

```monkeyc
var xLeftPeak  = (w * 27) / 100;   // left apex
var xRightPeak = (w * 73) / 100;   // right apex
var xSpread    = (w * 11) / 100;   // half-spread from apex to each base column

var xLeftA  = xLeftPeak  - xSpread;  // dist  ≈ 16%
var xLeftB  = xLeftPeak  + xSpread;  // speed ≈ 38%
var xRightA = xRightPeak - xSpread;  // mod   ≈ 62%
var xRightB = xRightPeak + xSpread;  // vig   ≈ 84%
```

### Rule of Thumb

If you push columns outward horizontally, pull the base row toward vertical center to compensate. Pushing from 25%/75% to 15%/85% required pulling `_yBottom` from 80% to 75%.

---

## 5. Iterative Label and Unit Offset Tuning

### labelOff — Independent Values for Top and Bottom Slots

Top and bottom layouts need different `labelOff` values because in the bottom layout the label renders below the value and is closer to the adjacent row.

```monkeyc
labelOff = -22;   // Top layout: label 22px above value anchor
labelOff = 36;    // Bottom layout: label 36px below value anchor
```

### Unit Subscript Offset Evolution (`mi`, `mph`)

| Offset | Result |
|--------|--------|
| +18 | Units touching digits — visually merged |
| +24 | Still touching in larger font contexts |
| +32 | Almost there |
| +38 | Clean separation — final value |

### Key Lesson

Label and unit offsets are **pixel values, not percentages** — typographic spacing doesn't scale with slot height. All row/column positions should use percentage arithmetic; these typographic gaps remain fixed pixel values.

---

## 6. Shifting the Entire Layout as a Unit

### When to Do This

When unit subscripts (bottom of top slot) started clipping against the center line, adjusting only `yBase` would re-introduce overlap with adjacent elements. The correct fix was to shift ALL four Y positions for the top layout upward by the same amount.

### How to Do It Safely

All four top-layout Y positions use percentage arithmetic from `h`. To shift everything up by ~4%, subtract 4 from each percentage:

```
Before:  yTime=8%, yDailySteps=23%, yPeak=46%, yBase=73%
After:   yTime=4%, yDailySteps=19%, yPeak=42%, yBase=69%
```

Internal gaps (15%, 23%, 27%) remain identical. Only absolute position changes.

**Rule:** When clipping occurs at one edge and the opposite edge has free space, shift all Y positions uniformly rather than adjusting individual rows. This avoids cascading overlap problems.

---

## 7. The Steps/Distance Zero Bug — Wrong API

### Symptom

Steps, distance, and speed displayed 0 throughout an entire activity session, never updating regardless of actual movement.

### Root Cause

The original code derived activity steps from `info.elapsedDistance`:

```monkeyc
// WRONG — elapsedDistance is GPS-based, null/zero indoors
var currentActivitySteps = 0;
if (info.elapsedDistance != null) {
    currentActivitySteps = (info.elapsedDistance * STEPS_PER_MILE / 1609.34).toNumber();
}
```

`Activity.Info.elapsedDistance` is populated from GPS. For indoor activities (Cardio, Indoor Walk, etc.) GPS is off and this field is always null.

### Correct Approach: ActivityMonitor with Baseline

```monkeyc
// In onTimerStart():
var amInfo = ActivityMonitor.getInfo();
_baselineSteps  = (amInfo.steps    != null) ? amInfo.steps    : 0;
_baselineDistCm = (amInfo.distance != null) ? amInfo.distance : 0;

// In compute():
var amInfo        = ActivityMonitor.getInfo();
var currentSteps  = (amInfo.steps    != null) ? amInfo.steps    : _baselineSteps;
var currentDistCm = (amInfo.distance != null) ? amInfo.distance : _baselineDistCm;
var sessionSteps  = currentSteps  - _baselineSteps;
var sessionDistCm = currentDistCm - _baselineDistCm;
if (sessionSteps  < 0) { sessionSteps  = 0; }
if (sessionDistCm < 0) { sessionDistCm = 0; }
var distMiles = sessionDistCm.toFloat() / 160934.4f;  // cm per mile
```

### General Rule

- **Indoor** step/distance metrics → **always use `ActivityMonitor.getInfo()`** (accelerometer-based)
- **Never use `Activity.Info.elapsedDistance`** for indoor activities (GPS-based, null indoors)
- **Capture a baseline at `onTimerStart()`** to isolate current session from lifetime totals

---

## 8. App Name vs. PRG Filename

Renaming the `.prg` file before copying to the watch does NOT change the app's display name. The name shown in the watch UI comes exclusively from the string resource compiled into the binary:

```xml
<!-- resources/strings/strings.xml -->
<string id="AppName">Your Name Here</string>
```

Edit `strings.xml` and rebuild. The `.prg` filename is irrelevant.

Naming constraints: ampersand (`&`) is prohibited even as `&amp;`. Use "and". Letters, numbers, and spaces are safe.

---

## 9. Final Working Layout Values

Tested through ~10 simulator + real-device iterations. Venu 3 at half-screen slot height:

### Horizontal (same for both orientations)

```monkeyc
var xCenter    = w / 2;
var xLeftPeak  = (w * 27) / 100;
var xRightPeak = (w * 73) / 100;
var xSpread    = (w * 11) / 100;

var xLeftA  = xLeftPeak  - xSpread;   // dist  ≈ 16%
var xLeftB  = xLeftPeak  + xSpread;   // speed ≈ 38%
var xRightA = xRightPeak - xSpread;   // mod   ≈ 62%
var xRightB = xRightPeak + xSpread;   // vig   ≈ 84%
```

### Vertical — Top Slot

```monkeyc
yTime       = (h *  4) / 100;
yDailySteps = (h * 19) / 100;
yPeak       = (h * 42) / 100;
yBase       = (h * 69) / 100;
labelOff    = -22;    // label ABOVE value
// mi/mph drawn at yBase + 38
```

### Vertical — Bottom Slot

```monkeyc
yTime       = (h * 76) / 100;
yDailySteps = (h * 61) / 100;
yPeak       = (h * 40) / 100;
yBase       = (h * 13) / 100;
labelOff    = 36;     // label BELOW value
// mi/mph drawn at yBase + 38 (same)
```

---

## 10. Complete Compute Pattern for Steps and Distance

```monkeyc
// Member variables:
// _baselineSteps  as Number  (0 initially)
// _baselineDistCm as Number  (0 initially)

// onTimerStart():
var amInfo = ActivityMonitor.getInfo();
_baselineSteps  = (amInfo.steps    != null) ? amInfo.steps    : 0;
_baselineDistCm = (amInfo.distance != null) ? amInfo.distance : 0;

// compute():
var amInfo = ActivityMonitor.getInfo();

// Daily display (global, not session)
var dailySteps = (amInfo.steps    != null) ? amInfo.steps    : 0;
var stepGoal   = (amInfo.stepGoal != null) ? amInfo.stepGoal : 0;

// Session-only (baseline-subtracted)
var currentSteps  = (amInfo.steps    != null) ? amInfo.steps    : _baselineSteps;
var currentDistCm = (amInfo.distance != null) ? amInfo.distance : _baselineDistCm;
var sessionSteps  = currentSteps  - _baselineSteps;
var sessionDistCm = currentDistCm - _baselineDistCm;
if (sessionSteps  < 0) { sessionSteps  = 0; }
if (sessionDistCm < 0) { sessionDistCm = 0; }
var distMiles = sessionDistCm.toFloat() / 160934.4f;

// Rolling speed (only when timer is running)
if (info.timerState != null && info.timerState == Activity.TIMER_STATE_ON) {
    var nowMs = System.getTimer();
    _speedSamples.add({ :distCm => sessionDistCm, :time => nowMs });
    // trim window to desired duration...
    if (_speedSamples.size() >= 2) {
        var oldest  = _speedSamples[0];
        var newest  = _speedSamples[_speedSamples.size() - 1];
        var deltaCm = newest[:distCm] - oldest[:distCm];
        var deltaMs = newest[:time]   - oldest[:time];
        speedMph = (deltaCm.toFloat() / (deltaMs.toFloat() + 1.0f))
                   * (3600000.0f / 160934.4f);
        // +1.0f prevents div/0 (branchless; see monkeyc_analyzer_unreachable_statement_guide.md)
    }
}
```

---

## Summary of Key Lessons

1. **Venu 3 has a 2-app limit** — design for density; one well-designed field beats two simple ones
2. **Dual-triangle layouts** organize 6+ metrics using implicit spatial grouping
3. **`getObscurityFlags()`** enables position-aware rendering — check `OBSCURE_BOTTOM` for bottom slot
4. **Horizontal and vertical bezel clearance interact** — pushing columns outward requires pulling the row toward vertical center
5. **Label offsets are not symmetric** — top and bottom slots need different `labelOff` values
6. **Unit subscript offsets are fixed pixels** — typographic spacing doesn't scale with slot height
7. **Shift entire layouts as a unit** — move all Y positions by the same percentage when clipping appears at one edge
8. **`Activity.Info.elapsedDistance` is GPS-only** — use `ActivityMonitor` for indoor step/distance metrics
9. **Capture baselines at `onTimerStart()`** — `ActivityMonitor` returns lifetime totals; subtract baseline for session values
10. **App name comes from `strings.xml`**, not the `.prg` filename

---

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Core SDK concepts, full permission table, `Activity.Info` reference
- `garmin_development_addendum.md` — Developer keys, installation, simulator setup
- `connectiq_code_patterns.md` — Circular buffer, indoor step/distance baseline pattern
- `venu3_practical_lessons.md` — Layout guidelines, bezel constraints, screen configuration
- `steps_and_time_development_lessons.md` — ActivityMonitor permission rules, time formatting
- `monkeyc_analyzer_unreachable_statement_guide.md` — Branchless arithmetic to avoid div/0 warnings

---

*Last updated: February 2026 — SDK 8.4.1*
