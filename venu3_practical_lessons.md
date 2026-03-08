# Venu 3 Practical Development Lessons
## Discoveries from Real-World Testing — February 2026

This document captures practical knowledge discovered during development and testing that is not covered in the official SDK documentation.

---

## Numbered Index

1. [Data Screen Configuration on Venu 3](#1-data-screen-configuration-on-venu-3)
2. [Layout Guidelines for 1-of-4 Data Field Displays](#2-layout-guidelines-for-1-of-4-data-field-displays)
3. [Intensity Minutes Field — Baseline Capture Timing](#3-intensity-minutes-field--baseline-capture-timing)

---

## 1. Data Screen Configuration on Venu 3

### The Phone App vs. On-Watch Configuration Split

There are **two separate systems** for managing activity data screens on the Venu 3, and they do not fully see each other:

**Garmin Connect Phone App:**
- Only shows/manages screens created through Garmin's official Connect IQ install flow
- Cannot see or manage screens that were created directly on the watch

**On-Watch Configuration:**
- Can create, edit, and delete data screens independently of the phone app
- Screens created here are NOT visible in the phone app's Data Screens list
- This is the only way to add additional data screens if the phone app doesn't expose the option

### How to Access On-Watch Data Screen Configuration

**Critical: This menu is only accessible during an active activity session.**

You CANNOT configure data screens from the watch's main Settings menu, the watch face long-press menu, or the Garmin Connect phone app (for on-watch-created screens).

**Correct procedure:**
1. Start the activity (e.g. Cardio) — press the Start button to begin recording
2. Once the activity is running, **long-press the lower button** on the watch
3. This opens the activity settings menu
4. Navigate to Data Screens
5. Add, remove, or edit screens and fields here
6. Changes take effect immediately

### The "Screen Full" Error

When installing a new Connect IQ data field, Garmin may prompt "Add to activities?" If you accept and get an error that an activity is "full," the activity has reached its maximum number of data screens. Remove an existing screen via the on-watch method before another can be added automatically.

### Why Screens Created by the Install Flow Persist After Field Removal

When the Connect IQ install flow creates a dedicated screen for a data field, that screen is managed independently from the field assignment. If you later remove the field from that screen or reassign it, the **screen itself remains** — visible and swipeable during activities but showing as empty/default content.

To fully remove it, explicitly delete the screen through the on-watch Data Screens configuration menu during an active activity.

### Sideloaded Apps vs. Store-Installed Apps

| Behavior | Sideloaded (.prg copy) | Store Installed |
|----------|------------------------|-----------------|
| "Add to activities" prompt | No | Yes |
| Auto-creates dedicated screen | No | Yes (if activity not full) |
| Visible in phone app Data Screens | No | Yes |
| Available to add to existing screens | Yes | Yes |
| On-watch configuration | Yes | Yes |

**Practical implication for development:** During development (sideloading), you will always need to manually add your data field to a screen using the on-watch configuration. The auto-screen-creation behavior users experience from a store install cannot be tested via sideloading.

---

## 2. Layout Guidelines for 1-of-4 Data Field Displays

### The Problem

A custom DataField's `onLayout()` receives different dimensions depending on display context:
- **Full screen / 1-field layout:** full usable watch face dimensions
- **1-of-4 layout:** roughly 1/4 of the screen area as a small rectangular slot

The same percentage-based positions that look fine at full size can cause overlapping text, clipped content, and bezel collisions at the smaller size. The 1-of-4 case is further complicated by the **circular bezel**: slots near the edges of the round screen have their corners clipped, meaning content near outer edges both horizontally and vertically risks being hidden.

### Tested Working Layout Values (IntensityMinutes Field)

These values were arrived at through iterative simulator + real-device testing and represent a good starting point for a 3-tier vertical layout (label / large value / two sub-values):

```monkeyc
// Horizontal
_xCenter = width / 2;              // 50% — center anchor
_xLeft   = (width  *  3) / 20;    // 15% — left sub-value column
_xRight  = (width  * 17) / 20;    // 85% — right sub-value column

// Vertical
_yLabel  = height / 6;             // ~17% — top label
_yTotal  = (height * 11) / 20;    // 55%  — large center value
_yBottom = (height *  3) /  4;    // 75%  — bottom sub-values anchor

// Sub-value label offset (drawn above _yBottom)
bottomLabelY = _yBottom - 28;     // 28px above the value anchor
```

### Key Lessons

**1. Start with the 1-of-4 view as your constraint, not full screen.**
Full screen will always look fine; it's the compressed view that breaks first. Use the simulator's multi-field layout (Simulation → Data Fields) to test early.

**2. Percentage-based positions are essential — never use fixed pixel offsets for primary layout.**
`dc.getWidth()` and `dc.getHeight()` return the actual slot dimensions at runtime, so percentage arithmetic automatically scales between full-screen and compressed contexts.

**3. The circular bezel clips corners — horizontal and vertical position interact.**
Pushing sub-value columns outward horizontally (e.g. to 15%/85%) gains horizontal space but puts those elements closer to the curved edge of the bezel. Compensate by also moving them *upward* vertically. Moving columns from 25%/75% to 15%/85% required pulling `_yBottom` from 80% to 75% to stay inside the visible area.

**4. Large number fonts eat vertical space fast.**
`FONT_NUMBER_MEDIUM` is tall enough that a single digit can overlap a label placed at the "obvious" position above it. Place the label higher than you think necessary (around 17% from top) and the value lower (around 55%).

**5. Two-digit numbers are the real test case.**
Single-digit "0" values look fine in almost any layout. Always test with two-digit values.

**6. The label-to-value gap for sub-rows needs ~28px minimum.**
For a `FONT_XTINY` label sitting above a `FONT_SMALL` value, a vertical offset of 28px between the label anchor and value anchor provided clean separation in the 1-of-4 context.

### Simulator Workflow for Layout Tuning

1. Start a simulated activity (Simulation → Activity Data)
2. Open Data Fields menu → select a 4-field layout
3. Assign your field to one of the slots
4. Iterate on `onLayout()` position values, rebuild, and check
5. Pay particular attention to the **bottom-left and bottom-right slots** — these sit nearest the curved bezel edge and are where clipping is most likely
6. Once the 4-field layout looks clean, verify the full-screen view still looks acceptable

---

## 3. Intensity Minutes Field — Baseline Capture Timing Note

The IntensityMinutesField captures its baseline from `ActivityMonitor.activeMinutesWeek` at `onTimerStart()`. This means:

- If the timer is already running when the field loads (e.g. the watch was restarted mid-activity), `onTimerStart()` fires immediately on load per SDK documentation. The baseline may not reflect the true start of the activity in this edge case.
- For normal use (field present before activity start), behavior is correct.

---

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — `getObscurityFlags()`, font constants, percentage-based positioning
- `garmin_development_addendum.md` — Sideloading workflow, on-watch configuration process
- `consolidated_field_development_lessons.md` — Detailed bezel-safe positioning values, dual-slot layout examples
- `timerhr_field_development_lessons.md` — Dual x-column strategy, negative y-coordinate technique

---

*Last updated: February 2026 — SDK 8.4.1*
