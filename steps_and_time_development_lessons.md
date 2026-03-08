# Steps and Time Data Field — Development Lessons
## Real-World Development Experience — February 17, 2026

**Project:** Steps and Time Data Field for Garmin Venu 3
**SDK Version:** 8.4.1 / API Level 5.2
**Status:** Complete and deployed successfully

This document captures practical lessons learned while building a custom data field to display daily step count and current time — topics not covered in the official SDK documentation.

---

## Numbered Index

1. [Project Overview](#1-project-overview)
2. [Automatic Full-Screen View Discovery](#2-automatic-full-screen-view-discovery)
3. [XML Special Character Handling](#3-xml-special-character-handling)
4. [Permission Requirements — ActivityMonitor Needs None](#4-permission-requirements--activitymonitor-needs-none)
5. [Font Sizing and Layout](#5-font-sizing-and-layout)
6. [Color Consistency with Native Fields](#6-color-consistency-with-native-fields)
7. [SimpleDataField vs DataField](#7-simpledatafield-vs-datafield)
8. [Activity Compatibility](#8-activity-compatibility)
9. [Time Formatting Best Practices](#9-time-formatting-best-practices)
10. [Complete Working Code](#10-complete-working-code)

---

## 1. Project Overview

### Goal
Display two values in a single data field slot:
- Current daily step count (from ActivityMonitor)
- Current time of day (auto-detects 12hr/24hr format from watch settings)

### Why This Project Was Needed
- Garmin's built-in "Time of Day" field had a bug showing sunrise time instead of current time when added to bike activities
- Users wanted both steps and time visible during cardio/bike workouts
- Needed a cleaner, more minimal display than default multi-field layouts

### Final Result
- Works in all activities (Cardio, Bike, Custom activities)
- Auto-detects 12/24 hour format from watch settings
- Minimal, clean design matching Garmin's default field aesthetics
- Displays in both small field slot AND automatic full-screen view

---

## 2. Automatic Full-Screen View Discovery

### The Unexpected Behavior

When a user adds a **custom DataField** (one that extends `WatchUi.DataField` with custom `onUpdate()` rendering) to an activity, Garmin **automatically creates TWO views**:

1. **Small field slot** — the field appears where the user placed it
2. **Full-screen page** — a dedicated screen showing just that field at full size

**This is NOT programmed behavior — it's Garmin's automatic feature.**

### Why This Happens

| Base Class | Result | Use Case |
|------------|--------|----------|
| `WatchUi.SimpleDataField` | Field slot only | Single value, Garmin handles formatting |
| `WatchUi.DataField` | Field slot + Full-screen page | Custom UI via `onUpdate()` |

```monkeyc
// This triggers BOTH field slot AND full-screen view automatically
class MyField extends WatchUi.DataField {
    public function onUpdate(dc as Dc) as Void {
        dc.drawText(...);
    }
}
```

### User Experience
When the user swipes to the auto-created full-screen page during an activity, the same `onUpdate()` code runs but `dc.getWidth()` and `dc.getHeight()` return the full 454×454 screen dimensions. Layout must handle both sizes gracefully — use percentage-based positioning.

---

## 3. XML Special Character Handling

### The Ampersand Problem

Garmin's compiler explicitly prohibits ampersands in app names — even when properly escaped as `&amp;`. This is a Garmin-specific restriction.

```xml
<!-- FAILS — XML parse error -->
<string id="AppName">Steps & Time</string>

<!-- FAILS — Garmin rejects even properly escaped entity -->
<string id="AppName">Steps &amp; Time</string>

<!-- WORKS -->
<string id="AppName">Steps and Time</string>
```

Safe characters: letters, numbers, spaces. Use spelled-out "and" instead of "&".

---

## 4. Permission Requirements — ActivityMonitor Needs None

### The Mistake

```xml
<!-- WRONG — causes build error: "Invalid permission provided: ActivityMonitor" -->
<iq:permissions>
    <iq:uses-permission id="ActivityMonitor"/>
</iq:permissions>
```

### The Fix

```xml
<!-- CORRECT — ActivityMonitor needs no permission declaration -->
<iq:permissions>
    <!-- Empty, or include only the permissions you actually need -->
</iq:permissions>
```

### Why
`ActivityMonitor` is part of the core Toybox API — always available without permission.

**APIs that do NOT need permissions:** ActivityMonitor, System, Graphics, WatchUi, Lang, Math, Application.Storage

**APIs that DO need permissions:** Sensor, SensorHistory, Positioning, Communications, UserProfile, FitRecording

See `garmin_connectiq_knowledge_base.md` § 7 for the complete permission reference table.

---

## 5. Font Sizing and Layout

### The Initial Problem
First version used large fonts that caused text overlap, especially in the compressed 1-of-4 slot view. Layout looked fine in the simulator at full screen but failed in real use.

### Working Layout

```monkeyc
public function onUpdate(dc as Dc) as Void {
    var width   = dc.getWidth();
    var height  = dc.getHeight();
    var xCenter = width / 2;

    // Two-row layout at 1/3 and 2/3 height
    var ySteps = height / 3;
    var yTime  = (2 * height) / 3;

    // Steps label above steps value
    dc.drawText(xCenter, ySteps - 20, Graphics.FONT_XTINY, "Steps", Graphics.TEXT_JUSTIFY_CENTER);
    dc.drawText(xCenter, ySteps + 2,  Graphics.FONT_SMALL,  _steps.format("%d"), Graphics.TEXT_JUSTIFY_CENTER);

    // Time value (no label needed — format is self-explanatory)
    dc.drawText(xCenter, yTime, Graphics.FONT_SMALL, _timeString, Graphics.TEXT_JUSTIFY_CENTER);
}
```

Key rule: always test with two-digit values, not just single digits or the "--" placeholder.

---

## 6. Color Consistency with Native Fields

### Adapt to User Theme

```monkeyc
var bgColor = getBackgroundColor();  // Returns user's theme color (Black or White)
var fgColor = Graphics.COLOR_WHITE;
if (bgColor == Graphics.COLOR_WHITE) {
    fgColor = Graphics.COLOR_BLACK;
}
dc.setColor(fgColor, bgColor);
dc.clear();
dc.setColor(fgColor, Graphics.COLOR_TRANSPARENT);
// Draw text with fgColor
```

Using the same colors as native fields reduces cognitive friction. Users expect visual consistency.

---

## 7. SimpleDataField vs DataField

| Feature | SimpleDataField | DataField |
|---------|----------------|-----------|
| Rendering | Garmin handles | Custom `onUpdate()` |
| Full-Screen | No automatic full screen | Yes — automatic dual view |
| Multiple Values | No — single value only | Yes |
| Complexity | Minimal | More code, full control |

```monkeyc
// SimpleDataField — one value only
class StepsOnly extends WatchUi.SimpleDataField {
    public function initialize() {
        SimpleDataField.initialize();
        label = "Steps";
    }
    public function compute(info as Info) as Numeric or String or Null {
        var activityInfo = ActivityMonitor.getInfo();
        return activityInfo.steps;
    }
}
```

Use `DataField` when displaying multiple values or custom layouts. Use `SimpleDataField` for single values when Garmin's auto-formatting is acceptable.

---

## 8. Activity Compatibility

Data fields work in **all activities** automatically — no `fit-contrib-activities` is needed or allowed in the manifest.

If a field doesn't appear in a specific activity after installation, try restarting the watch. This is an installation timing issue, not a compatibility problem.

| Behavior | Sideloaded (.prg copy) | Store Installed |
|----------|------------------------|-----------------|
| "Add to activities" prompt | No | Yes |
| Auto-creates dedicated screen | No | Yes |
| Visible in phone app Data Screens | No | Yes |
| Available to add to existing screens | Yes | Yes |
| On-watch configuration | Yes | Yes |

---

## 9. Time Formatting Best Practices

### Auto-Detect 12hr vs 24hr

```monkeyc
var clockTime      = System.getClockTime();
var hour           = clockTime.hour;    // 0-23
var minute         = clockTime.min;     // 0-59
var deviceSettings = System.getDeviceSettings();

if (!deviceSettings.is24Hour) {
    // 12-hour format
    var amPm = "AM";
    if (hour >= 12) {
        amPm = "PM";
        if (hour > 12) { hour = hour - 12; }
    }
    if (hour == 0) { hour = 12; }
    _timeString = hour.format("%d") + ":" + minute.format("%02d") + " " + amPm;
} else {
    // 24-hour format
    _timeString = hour.format("%02d") + ":" + minute.format("%02d");
}
```

### Edge Cases
- Midnight (0:00) → 12:00 AM in 12hr, 00:00 in 24hr
- Noon (12:00) → 12:00 PM in 12hr, 12:00 in 24hr
- 1 PM (13:00) → 1:00 PM in 12hr, 13:00 in 24hr

### Format Specifiers
```monkeyc
"%d"    // Integer, no leading zero (1, 2, 12)
"%02d"  // 2 digits with leading zero (01, 02, 12)
```

---

## 10. Complete Working Code

### StepsTimeView.mc

```monkeyc
import Toybox.Activity;
import Toybox.ActivityMonitor;
import Toybox.Graphics;
import Toybox.Lang;
import Toybox.System;
import Toybox.WatchUi;

class StepsTimeView extends WatchUi.DataField {
    private var _steps      as Number;
    private var _timeString as String;

    public function initialize() {
        DataField.initialize();
        _steps      = 0;
        _timeString = "--:--";
    }

    public function onLayout(dc as Dc) as Void { }

    public function compute(info as Activity.Info) as Void {
        // Get steps from ActivityMonitor (no permission needed, works indoors)
        var amInfo = ActivityMonitor.getInfo();
        _steps = (amInfo.steps != null) ? amInfo.steps : 0;

        // Get time and format per device settings
        var clockTime = System.getClockTime();
        var hour      = clockTime.hour;
        var minute    = clockTime.min;
        var settings  = System.getDeviceSettings();

        if (!settings.is24Hour) {
            var amPm = "AM";
            if (hour >= 12) {
                amPm = "PM";
                if (hour > 12) { hour = hour - 12; }
            }
            if (hour == 0) { hour = 12; }
            _timeString = hour.format("%d") + ":" + minute.format("%02d") + " " + amPm;
        } else {
            _timeString = hour.format("%02d") + ":" + minute.format("%02d");
        }
    }

    public function onUpdate(dc as Dc) as Void {
        var bgColor = getBackgroundColor();
        var fgColor = (bgColor == Graphics.COLOR_WHITE) ? Graphics.COLOR_BLACK : Graphics.COLOR_WHITE;

        dc.setColor(fgColor, bgColor);
        dc.clear();
        dc.setColor(fgColor, Graphics.COLOR_TRANSPARENT);

        var width   = dc.getWidth();
        var height  = dc.getHeight();
        var xCenter = width / 2;
        var ySteps  = height / 3;
        var yTime   = (2 * height) / 3;

        dc.drawText(xCenter, ySteps - 20, Graphics.FONT_XTINY, "Steps", Graphics.TEXT_JUSTIFY_CENTER);
        dc.drawText(xCenter, ySteps + 2,  Graphics.FONT_SMALL,  _steps.format("%d"), Graphics.TEXT_JUSTIFY_CENTER);
        dc.drawText(xCenter, yTime,        Graphics.FONT_SMALL,  _timeString, Graphics.TEXT_JUSTIFY_CENTER);
    }
}
```

### StepsTimeApp.mc

```monkeyc
import Toybox.Application;
import Toybox.Lang;
import Toybox.WatchUi;

class StepsTimeApp extends Application.AppBase {
    public function initialize() { AppBase.initialize(); }

    public function getInitialView() as [Views] or [Views, InputDelegates] {
        return [new StepsTimeView()];
    }
}

function getApp() as StepsTimeApp {
    return Application.getApp() as StepsTimeApp;
}
```

---

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Full permission table, `SimpleDataField` vs `DataField`, manifest configuration
- `garmin_development_addendum.md` — Installation workflow, `drawables.xml` location, array type safety
- `venu3_practical_lessons.md` — Layout guidelines for 1-of-4 slot, percentage-based positioning
- `consolidated_field_development_lessons.md` — More complex multi-metric field building on the same foundations

---

*Last updated: February 2026 — SDK 8.4.1*
