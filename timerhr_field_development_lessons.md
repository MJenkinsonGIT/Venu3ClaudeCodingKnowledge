# TimerHRCalories Field — Development Lessons
## Real-World Discoveries — February 24, 2026

**Project:** TimerHRCalories Data Field for Garmin Venu 3
**SDK Version:** 8.4.1 / API Level 5.2
**Status:** Complete and functional

This document captures new lessons discovered during TimerHRCalories development that are not covered in existing knowledge base files.

---

## Numbered Index

1. [HR Zone Coloring via UserProfile API](#1-hr-zone-coloring-via-userprofile-api)
2. [Negative Y-Coordinates Strip Font Ascent Padding](#2-negative-y-coordinates-strip-font-ascent-padding)
3. [FONT_NUMBER_MILD for Tight Vertical Spaces](#3-font_number_mild-for-tight-vertical-spaces)
4. [Partial Edit Failure — "Undefined Symbol" Build Error](#4-partial-edit-failure--undefined-symbol-build-error)
5. [Dual Independent X-Column Strategy](#5-dual-independent-x-column-strategy)
6. [DST-Proof "Today" Boundary for Activity History](#6-dst-proof-today-boundary-for-activity-history)

---

## 1. HR Zone Coloring via UserProfile API

### The API

`UserProfile.getHeartRateZones(sport)` returns a **6-element `Array<Number>`** of BPM threshold values:

```
[0] = min zone 1  (lower bound of zone 1)
[1] = max zone 1  (= lower bound of zone 2)
[2] = max zone 2
[3] = max zone 3
[4] = max zone 4
[5] = max zone 5  (upper bound of zone 5)
```

Zone membership logic:

```monkeyc
if (hr < zones[0])  → below all zones (default color)
if (hr <= zones[1]) → zone 1
if (hr <= zones[2]) → zone 2
if (hr <= zones[3]) → zone 3
if (hr <= zones[4]) → zone 4
else                → zone 5
```

### Sport-Specific Zones

```monkeyc
_hrZones = UserProfile.getHeartRateZones(UserProfile.getCurrentSport());
```

Call each `compute()` cycle so zones update if sport changes. Falls back to `HR_ZONE_SPORT_GENERIC` when the sport has no dedicated zones.

### Permission Required

`UserProfile` must be declared in `manifest.xml` — without it the build succeeds but the API throws at runtime:

```xml
<iq:permissions>
    <iq:uses-permission id="UserProfile"/>
</iq:permissions>
```

This differs from `ActivityMonitor` which requires no permission.

### Garmin Connect Zone Color Scheme

| Zone | Color Constant | Description |
|------|---------------|-------------|
| 1 | `Graphics.COLOR_LT_GRAY` | Warm Up |
| 2 | `Graphics.COLOR_BLUE` | Easy |
| 3 | `Graphics.COLOR_GREEN` | Aerobic |
| 4 | `Graphics.COLOR_ORANGE` | Threshold |
| 5 | `Graphics.COLOR_RED` | Maximum |

### Complete Working Helper Function

```monkeyc
// _hrZones as Array<Number> or Null -- loaded in compute()
private function getZoneColor(hr as Number or Null, defaultColor as Number) as Number {
    if (hr == null) { return defaultColor; }
    var zones = _hrZones;  // Store in local var — avoids static analyzer warning
    if (zones == null) { return defaultColor; }
    if (hr < zones[0])  { return defaultColor; }
    if (hr <= zones[1]) { return Graphics.COLOR_LT_GRAY; }  // zone 1
    if (hr <= zones[2]) { return Graphics.COLOR_BLUE; }     // zone 2
    if (hr <= zones[3]) { return Graphics.COLOR_GREEN; }    // zone 3
    if (hr <= zones[4]) { return Graphics.COLOR_ORANGE; }   // zone 4
    return Graphics.COLOR_RED;                               // zone 5
}
```

**Note:** Storing `_hrZones` in a local variable before the null check is required to avoid the Monkey C static analyzer warning. Direct use of the member variable in the conditional chain would trigger "Statement is not reachable" warnings because the analyzer traces the null-initialized value from `initialize()`. See `monkeyc_analyzer_unreachable_statement_guide.md`.

### Simulator Limitation

The simulator may not have real HR zone data loaded. Zone coloring will appear correct on-device once the watch has the user's actual profile. Test zone color logic on the real device during an activity.

---

## 2. Negative Y-Coordinates Strip Font Ascent Padding

### The Problem

When a text element is positioned at `y = 0` (the very top of a slot), the visible glyph does not appear at the top edge — there is a noticeable gap of blank space above the rendered characters. This gap is the font's internal **ascent padding** (space reserved above the tallest glyph in the font).

For `FONT_NUMBER_MILD` this padding is approximately 15–20px. For `FONT_NUMBER_MEDIUM` it is larger (~20–25px).

### The Fix: Go Negative

Setting `y` to a small negative value pushes the anchor point above the slot boundary, causing the ascent padding to be clipped while the actual glyph body remains visible:

```monkeyc
// Strips ~20px of ascent padding from a NUMBER_MILD font
_yHRVal = (h * -9) / 100;   // approximately -20px when h = 227px
```

The visible digits still render cleanly — only the blank space above them is lost.

### Key Lesson

**`y = 0` is NOT the same as "text touching the top edge."** The text anchor includes font ascent space that renders above the glyphs. If a `NUMBER_*` font value appears to float with empty space above it at `y = 0`, use negative y to pull it up.

**This is intentional, not a hack.** Garmin's `drawText` treats the y coordinate as the top of the font's bounding box, not the top of the visible glyph.

### Practical Values (FONT_NUMBER_MILD, slot h ≈ 227px)

| y% | y pixels | Effect |
|----|----------|--------|
| 0% | 0px | ~18px blank gap above digits |
| -5% | -11px | ~7px blank gap above digits |
| -9% | -20px | Digits nearly flush with slot top |
| -12% | -27px | Digits clipped at top by ~7px |

---

## 3. FONT_NUMBER_MILD for Tight Vertical Spaces

### The Font Size Hierarchy

```
FONT_NUMBER_MILD     ← smallest number font (~45px glyph height)
FONT_NUMBER_MEDIUM   ← medium (~60px glyph height)
FONT_NUMBER_HOT      ← large
FONT_NUMBER_THAI_HOT ← largest
```

**Important:** The "--" placeholder is significantly shorter than real digits. Always verify layout with real numbers active during an actual activity, not just with "--" showing.

### When to Use FONT_NUMBER_MILD

Use `FONT_NUMBER_MILD` instead of `FONT_NUMBER_MEDIUM` when:
- The slot is a compressed half-screen view (≈227px tall)
- A large HR/number value must share vertical space with a label above or below it
- `FONT_NUMBER_MEDIUM` causes overlap with adjacent elements even after repositioning

In the TimerHRCalories bottom slot, switching from `FONT_NUMBER_MEDIUM` to `FONT_NUMBER_MILD` for the Heart Rate display was the key fix that allowed the "Heart Rate" label to sit below the number without overlapping.

### Approximate Heights (empirical, Venu 3)

| Font | Approximate glyph height |
|------|--------------------------|
| `FONT_XTINY` | ~13px |
| `FONT_TINY` | ~16px |
| `FONT_SMALL` | ~22px |
| `FONT_MEDIUM` | ~28px |
| `FONT_LARGE` | ~36px |
| `FONT_NUMBER_MILD` | ~45px |
| `FONT_NUMBER_MEDIUM` | ~60px |

These are measured from real device screenshots — the SDK does not publish glyph heights.

---

## 4. Partial Edit Failure — "Undefined Symbol" Build Error

### What Happened

When adding `getZoneColor()` to TimerHRCalories, multiple code edits were applied in sequence. The edit that added the function calls in `onUpdate()` succeeded, but earlier edits that added the import, member variable, and function definition failed silently.

The result — the compiler saw calls to `:getZoneColor` in `onUpdate()` but no function definition existed:

```
ERROR: Undefined symbol ':getZoneColor' detected.
```

### Diagnostic

When you see "Undefined symbol ':functionName' detected" for a private function you just added:

1. Search the file for the function **definition** (not just calls to it)
2. If absent, the edit that was supposed to add it failed
3. Re-read the relevant section of the file to see current state
4. Re-apply the missing edits individually

### Prevention

When adding a feature that requires changes in multiple places (import, member, function, usage), verify each edit applied before moving to the next. After any edit failure or uncertainty, **read the file** before proceeding.

---

## 5. Dual Independent X-Column Strategy

### Context

Some layouts benefit from having side columns at different horizontal positions for different rows. In TimerHRCalories:

- **Outer columns (Act Cals / Avg HR)** at `x = 27% / 73%` — positioned at the wider mid-section where the bezel allows more horizontal spread
- **Inner columns (Total Cal / Max HR)** at `x = 18% / 82%` — pushed further outward to use extra horizontal width near the display's equator, while their vertical position stays near center where the safe zone is wider

### Implementation

```monkeyc
_xLeft       = (w * 27) / 100;   // outer column pair
_xRight      = (w * 73) / 100;
_xInnerLeft  = (w * 18) / 100;   // inner column pair (wider spread)
_xInnerRight = (w * 82) / 100;
```

### Bezel Safety Check for x = 18% / 82%

At x = 18% of 454px = 82px from edge = 145px from watch center.
For that column to be safe: `√(227² − y_from_centre²) > 145`
Which means `y_from_centre < 175px`, or within the middle ~77% of the slot height.

Keep inner column content within the central 77% of each slot (avoid extreme top and bottom edges when using very wide x positions).

---

## Summary of Key Lessons

1. **`UserProfile.getHeartRateZones()`** returns a 6-element array; use `getCurrentSport()` for activity-appropriate thresholds automatically. Requires `UserProfile` permission in manifest.

2. **`y = 0` leaves font ascent padding visible.** Use small negative y values (e.g. `-9%`) to push `NUMBER_*` fonts flush against the slot top edge.

3. **`FONT_NUMBER_MILD`** (~45px) is meaningfully shorter than `FONT_NUMBER_MEDIUM` (~60px) — use it when a large number must fit in a compressed half-slot.

4. **Partial edit failures produce "Undefined symbol" errors** — if a function call exists but no definition, the edit adding the definition failed. Read the file and re-apply.

5. **Dual x-column pairs** enable using the full circular screen width by placing different rows at different horizontal spreads while keeping each within bezel geometry limits.

---

---

## 6. DST-Proof "Today" Boundary for Activity History

### Context

The TimerHRCalories field tracks **total recorded activity time today** using `UserProfile.getUserActivityHistory()`. Each `UserActivity` entry has a `startTime` (`Time.Moment`) stored as UTC. To filter for "today only", a boundary representing local midnight must be computed and compared against each activity's UTC timestamp.

This sounds simple but has multiple pitfalls, all confirmed by real-device testing.

### What Does NOT Work

#### `Time.today()`
Despite appearing correct most days, `Time.today()` fails on the night of DST spring-forward. A late-evening activity (e.g. 11:29pm CST = 05:29 UTC next day) sits after the new CDT midnight boundary (05:00 UTC) and is incorrectly included as "today".

#### `Gregorian.moment()` with local date components
The SDK documentation states: **"Each option value is assumed to be in the UTC time zone."** Passing local date/time components to `Gregorian.moment()` produces a UTC-midnight boundary, which is wrong — it includes even more previous-day activities than `Time.today()`.

#### `Gregorian.info(Time.now()) + subtract seconds-since-midnight`
This correctly computes local midnight on normal days but has the same DST spring-forward failure as `Time.today()` — the computed boundary is in the new timezone, while the activity was recorded in the old one.

#### `Gregorian.localMoment(location, activity.startTime)`
The correct API for this problem in theory — it maps a UTC timestamp to the local calendar date using historical DST rules. However it requires a `Position.Location`, and for indoor activities (strength, treadmill, etc.) `Activity.Info.currentLocation` is always null. Cannot be relied upon.

#### `ClockTime.dst`
**This field is always 0 on real Garmin devices — a longstanding firmware bug confirmed in multiple Garmin developer forum threads.** It works correctly in the simulator only. Garmin themselves have said they intend to deprecate it. Do not use `ClockTime.dst` for any logic on real hardware.

### What Works: Application.Storage Minimum Offset

`System.getClockTime().timeZoneOffset` correctly reflects the current combined UTC offset in seconds — including DST when active (e.g. CDT = -18000, CST = -21600). This value alone is insufficient because it changes with DST.

The key insight: **standard time is always more negative (or less positive) than DST time** — clocks spring forward, meaning the offset increases by 3600 when DST activates. So the minimum `timeZoneOffset` observed across all runs is always the standard-time (base) offset.

By persisting this minimum in `Application.Storage`, it is available from the first compute cycle on every future run, regardless of whether DST is currently active:

```monkeyc
private function sumTodayActivitySecs() as Number {
    var currentOffset = System.getClockTime().timeZoneOffset;

    // Persist the minimum (most-negative = standard-time) offset seen across runs.
    var storage = Application.Storage;
    var stored = storage.getValue("baseTzOffset");
    var baseOffset;
    if (stored == null || currentOffset < stored) {
        storage.setValue("baseTzOffset", currentOffset);
        baseOffset = currentOffset;
    } else {
        baseOffset = stored as Number;
    }

    // Convert UTC epoch values to local day numbers by shifting by base offset.
    // Integer division by 86400 gives the day bucket; equal buckets = same local day.
    var todayDay = (Time.now().value() + baseOffset) / 86400;

    var total = 0;
    var iter  = UserProfile.getUserActivityHistory();
    var entry = iter.next();
    while (entry != null) {
        var st  = entry.startTime;
        var dur = entry.duration;
        if (st != null && dur != null) {
            var actDay = (st.value() + baseOffset) / 86400;
            if (actDay == todayDay) {
                total = total + dur.value();
            }
        }
        entry = iter.next();
    }
    return total;
}
```

**Why this correctly handles DST spring-forward:**
- 11:29pm CST activity → UTC 05:29 → `(05:29_epoch + (-21600)) / 86400` = yesterday's day number ✓
- Current time CDT → UTC 20:00 → `(20:00_epoch + (-21600)) / 86400` = today's day number ✓
- Different day numbers → activity excluded ✓

### Bootstrap Caveat

If the app is first installed **during DST** (summer), `Application.Storage` has no value yet. On the first run, `stored == null` so `baseOffset` is set to the current DST offset (e.g. -18000 CDT). The spring-forward bug will persist until the following winter when the standard-time offset (-21600) is observed and stored.

**For personal/regional use:** seed `Application.Storage` with the known standard-time offset in `initialize()` on first install, then remove the seed before public release:

```monkeyc
// In initialize() — seed for US Central Standard Time.
// Remove before public release; other timezones self-calibrate after first winter.
if (Application.Storage.getValue("baseTzOffset") == null) {
    Application.Storage.setValue("baseTzOffset", -21600);
}
```

**For public release:** omit the seed. New users installing during DST will experience the spring-forward bug once, during their first year, then the stored minimum self-corrects permanently.

### Requires `Application` Import

```monkeyc
import Toybox.Application;
```

`Application.Storage` requires no manifest permission — `"Storage"` is not a valid permission ID and adding it causes a build error. The storage is available without any permission declaration.

### Active Minutes — Full Architecture

The complete implementation combines the history iterator with the live timer to avoid double-counting:

- `getUserActivityHistory()` returns only **completed and saved** activities — the current in-progress activity is not included
- `Activity.Info.timerTime` provides the current activity's elapsed milliseconds
- Sum = `(_historicalActiveSecs + currentSecs) / 60`

The history sum is expensive (iterates all saved activities), so it is refreshed every 60 `compute()` cycles (~60 seconds) using a counter. The current timer value is read every cycle from `info.timerTime`.

```monkeyc
_historyCounter = _historyCounter + 1;
if (_historyCounter >= 60) {
    _historyCounter = 0;
    _historicalActiveSecs = sumTodayActivitySecs();
}
var timerVal = _timerMs;
var currentSecs = (timerVal != null) ? (timerVal / 1000).toNumber() : 0;
_dailyActMins = (_historicalActiveSecs + currentSecs) / 60;
```

Initialise `_historyCounter = 60` so the first `compute()` call triggers an immediate history refresh rather than waiting 60 cycles.

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — `UserProfile` API, `Activity.Info` fields, full permission table
- `monkeyc_analyzer_unreachable_statement_guide.md` — Local variable pattern for HR zone null-guard; negative y explanation in §5 Drawings
- `consolidated_field_development_lessons.md` — `getObscurityFlags()`, dual-slot layout, bezel-aware positioning
- `venu3_practical_lessons.md` — Bezel geometry, layout guidelines

---

*Last updated: March 2026 — SDK 8.4.1*
