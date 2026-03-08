# Skin Temperature Widget — Development Lessons
## Real-World Discoveries — February 2026

**Project:** SkinTempWidget for Garmin Venu 3
**SDK Version:** 8.4.1 / API Level 5.2
**App Type:** Widget (64 KB memory limit)
**Status:** Complete and functional

This document captures practical lessons from building the skin temperature widget, with emphasis on the glance view styling process (many iterations) and touch/gesture input routing (non-obvious behavior on Venu 3).

---

## Numbered Index

1. [Glance View Architecture](#1-glance-view-architecture)
2. [Glance View Styling — Full Walkthrough](#2-glance-view-styling--full-walkthrough)
3. [AMOLED Brightness Calibration](#3-amoled-brightness-calibration)
4. [Pixel Sampling for UI Matching](#4-pixel-sampling-for-ui-matching)
5. [Gradient + Corner Masking Technique](#5-gradient--corner-masking-technique)
6. [Noise Texture via RLE Column Patterns](#6-noise-texture-via-rle-column-patterns)
7. [Launcher Icon — Transparent Trick](#7-launcher-icon--transparent-trick)
8. [Swipe and Touch Navigation on Venu 3](#8-swipe-and-touch-navigation-on-venu-3)
9. [onSwipe vs onNextPage / onPreviousPage](#9-onswipe-vs-onnextpage--onpreviouspage)
10. [Drag Events for Smooth Scrolling](#10-drag-events-for-smooth-scrolling)
11. [dc.setClip() for Graph Rendering](#11-dcsetclip-for-graph-rendering)
12. [App ID Conflicts Between Projects](#12-app-id-conflicts-between-projects)
13. [Timezone Bug — Gregorian.moment() Uses UTC](#13-timezone-bug--gregorianmoment-uses-utc)
14. [Widget Memory Limit (64 KB)](#14-widget-memory-limit-64-kb)
15. [fillPolygon Performance on Device](#15-fillpolygon-performance-on-device)
16. [Graph Layout — graphBottom and Bezel Clearance](#16-graph-layout--graphbottom-and-bezel-clearance)
17. [SensorHistory.getTemperatureHistory() — Period Parameter](#17-sensorhistorygettemperaturehistory--period-parameter)
18. [Page Indicator Dots (Custom Implementation)](#18-page-indicator-dots-custom-implementation)
19. [Application.Storage — No Permission Required](#19-applicationstorage--no-permission-required)
20. [Persistence Layer Architecture (TempStorage)](#20-persistence-layer-architecture-tempstorage)
21. [Glance Annotation — (:glance) Required on Both Classes](#21-glance-annotation--glance-required-on-both-classes)

---

## 1. Glance View Architecture

### How the dc Works in a Glance

`GlanceView` is passed a drawing context that covers only the glance slot area — not the full screen. The SDK documentation confirms that both `onLayout()` and `onUpdate()` receive this slot-bounded dc rather than a full-screen context.

On the Venu 3, the system renders the launcher icon (the circle with an app icon inside) **to the LEFT of your dc** — it is in its own reserved area. When your code draws from x=0, that x=0 is already past the icon. The dc represents only the content area.

---

## 2. Glance View Styling — Full Walkthrough

### Seven Iterations to Match System UI

This process took seven iterations before achieving a seamless result. Each attempt revealed a new requirement.

**Attempt 1: Solid gray card** — Visible but looked obviously different from system cards.

**Attempt 2: Simple gradient** — Better, but the card had visible left-side border mismatch.

**Attempt 3: First measured brightness (too dim)** — AMOLED is extremely high contrast; values that look visible on a laptop monitor are nearly invisible on the watch. See [AMOLED Brightness Calibration](#3-amoled-brightness-calibration).

**Attempt 4: Increased brightness + measured arc direction** — Gradient card visible. Fade formulas must be measured from where drawing begins, not from x=0. `ARC_COUNTER_CLOCKWISE` from 90→180° draws the top-left corner correctly.

**Attempt 5: Removed left-side border, matched system card via pixel sampling** — The system icon card background at the boundary with our dc measures ~RGB(90, 92, 89) at mid-height. Our card needs to START at those exact brightnesses. `fillPeak = 91`, `borderPeak = 97`, `fadeEnd = 68%` of dc width. Result: seamless extension of the system card with no visible join.

**Attempt 6: Pixel leak at bottom edge** — Gray fill leaked slightly below the bottom border (dc y=159 to y=163). Small black mask rect fixed it. Always verify with a screenshot that the fill doesn't escape the border.

**Attempt 7: Noise texture** — Native glances have a subtle per-column texture (different shades). A plain smooth gradient is visibly different at close range. See [Noise Texture via RLE Column Patterns](#6-noise-texture-via-rle-column-patterns).

---

## 3. AMOLED Brightness Calibration

The Venu 3's AMOLED display has extremely high contrast. Values that look visible on a laptop monitor may be nearly invisible on the watch.

**Empirically tested values:**

| RGB Brightness (0–255) | Appearance on Watch |
|------------------------|---------------------|
| 30–40 | Nearly black, barely visible |
| 55 | Very dark, hard to distinguish from black |
| 91 | Dark gray — visible as background fill |
| 97 | Medium-dark gray — clear as border line |
| 155 | Light gray — similar to `COLOR_LT_GRAY` |
| 200+ | Bright, stands out prominently |

**Constructing gray colors numerically:**
```monkeyc
var brightness = 91;
var col = (brightness << 16) | (brightness << 8) | brightness;
dc.setColor(col, Graphics.COLOR_TRANSPARENT);
```

This is the only way to use arbitrary gray values not covered by built-in `COLOR_*` constants.

---

## 4. Pixel Sampling for UI Matching

When your widget needs to visually join with the system UI (as in the glance card continuation), pixel sampling from screenshots is the most reliable way to match colors.

### Method
1. Take a screenshot of the watch showing your widget alongside native ones
2. Open the image in an editor or script that reports RGB values
3. Sample at multiple x positions along the boundary row and the top border row
4. Average and round to the nearest value computable with integer math

### Sampled Values
```
Card background (mid-height, at left edge of dc):
  x=148: RGB(89, 91, 88)   → fillPeak = 91
  x=152: RGB(93, 95, 92)
  x=156: RGB(83, 85, 82)

Top border (y=28, top of card):
  x=141: RGB(97, 97, 95)   → borderPeak = 97
  x=149: RGB(88, 88, 88)
```

The ~3px variation is the noise texture. Average values near the boundary gives the correct base brightness.

---

## 5. Gradient + Corner Masking Technique

The only reliable way to draw a gradient fill with clean rounded left corners — without any off-screen buffer — is the **draw-then-mask** approach:

**Step 1:** Draw full-width gradient strips (2px-wide `fillRectangle` strips at full dc height, fading from `fillPeak` at x=0 to black at `fadeEnd`). No rounded corners yet.

**Step 2:** Mask corners with black `fillCircle` arcs or `fillRectangle` blocks to create the rounded corner illusion.

**Step 3:** Draw border lines (horizontal top/bottom) with their own fade starting at `borderPeak`.

**Step 4:** Check for pixel leaks at edges; add black mask rectangles as needed.

---

## 6. Noise Texture via RLE Column Patterns

Native glances have a subtle scrambled texture — each pixel column has a slightly different shade. To reproduce this without per-pixel drawing (which is too slow):

Define an array of brightness offsets per column group (RLE-encoded) and apply each offset when drawing gradient strips:

```monkeyc
// Example: different brightness offsets for groups of columns
var noisePattern = [0, -3, 2, -1, 4, -2, 1, -4, 3, 0] as Array<Number>;
var noiseIdx = (x / 2) % noisePattern.size();
var adjusted = fillPeak + noisePattern[noiseIdx];
```

This gives a visually similar texture to the system glances without per-pixel overhead.

---

## 7. Launcher Icon — Transparent Trick

To make the launcher icon area (the circle shown in the glance list and app launcher) use a transparent background so the watch's system UI shows through:

- Create a 1-bit transparent PNG for the launcher icon area
- The system renders its own background behind your icon

This avoids designing an icon background that needs to match the system's gradient or theme.

---

## 8. Swipe and Touch Navigation on Venu 3

**Confirmed on device by video recordings of actual gestures:**

| Physical Gesture | Callback Fired |
|-----------------|----------------|
| **Swipe UP** | `onNextPage()` |
| **Swipe DOWN** | `onPreviousPage()` |
| **Swipe LEFT** | `onNextPage()` |
| **Swipe RIGHT** | `onPreviousPage()` |

This is counterintuitive and the opposite of what the callback names suggest. See [onSwipe vs onNextPage / onPreviousPage](#9-onswipe-vs-onnextpage--onpreviouspage) for why `onSwipe` should be avoided.

**IMPORTANT NOTE:** An earlier version of this document (before CustomHRV development) incorrectly documented swipe direction behavior, because the skin temp widget only cycled between two views and the direction was ambiguous. The mapping above is confirmed correct by video evidence from the CustomHRV app.

---

## 9. onSwipe vs onNextPage / onPreviousPage

### Why Not onSwipe?

`onSwipe` is fired by the touchscreen driver and is unreliable on Venu 3 — it may not fire consistently for all gesture speeds and directions. `onNextPage` and `onPreviousPage` are the reliable handlers.

Never use `onSwipe` for primary navigation on Venu 3.

### Practical Rule

When wiring up a two-action screen (increase/decrease, action-A/action-B):
- Put the action that responds to **Swipe UP** in `onNextPage()`
- Put the action that responds to **Swipe DOWN** in `onPreviousPage()`

---

## 10. Drag Events for Smooth Scrolling

For continuous scrolling (e.g. panning a graph), use `onDrag()` rather than swipe events:

```monkeyc
public function onDrag(dragEvent as WatchUi.DragEvent) as Boolean {
    var coords    = dragEvent.getCoordinates();  // Current position [x, y]
    var deltaX    = coords[0] - _prevX;
    _prevX        = coords[0];
    _scrollOffset += deltaX;
    WatchUi.requestUpdate();
    return true;
}
```

**`DragEvent.getCoordinates()` returns position, not delta.** Track previous X/Y yourself to compute movement.

---

## 11. dc.setClip() for Graph Rendering

Use `dc.setClip()` to restrict rendering to a rectangle, preventing graph bars from bleeding into header/footer areas:

```monkeyc
// Restrict to graph area only
dc.setClip(leftPad, topPad, graphW, graphH);
// ... draw graph bars ...
dc.clearClip();  // Remove restriction
// ... draw header/footer freely ...
```

---

## 12. App ID Conflicts Between Projects

Every project must have a **unique app ID** in `manifest.xml`. Copying a project template without changing the ID causes installs to silently overwrite each other on the watch.

**Changing an app ID** makes the watch treat it as a brand-new app — the old version stays installed and must be removed via Garmin Express manually.

Generate a new UUID for each new project.

---

## 13. Timezone Bug — Gregorian.moment() Uses UTC

`Gregorian.moment()` uses UTC, not local time. Never pass local time fields from `Gregorian.info()` to it.

**Wrong:**
```monkeyc
// This produces a UTC moment, not a local midnight moment
var localInfo   = Gregorian.info(Time.now(), Time.FORMAT_SHORT);
var localMidnight = Gregorian.moment({:year => localInfo.year, :month => localInfo.month, :day => localInfo.day});
```

**Correct approach for local midnight:**
```monkeyc
// Use System.getClockTime() and subtract elapsed seconds since midnight
var clock     = System.getClockTime();
var secSinceMidnight = clock.hour * 3600 + clock.min * 60 + clock.sec;
var localMidnight    = Time.now().subtract(new Time.Duration(secSinceMidnight));
```

---

## 14. Widget Memory Limit (64 KB)

The Venu 3 widget/glance memory limit is **65,536 bytes (64 KB)** — much smaller than the 256 KB data field limit.

- Use `const` for large lookup tables (they go in read-only flash, not RAM)
- Avoid creating large objects in `onUpdate()` (runs every refresh cycle)
- Pre-allocate arrays at initialization rather than growing them dynamically
- Profile memory usage with the simulator's memory tool

---

## 15. fillPolygon Performance on Device

`dc.fillPolygon()` causes **watchdog timeout** on the Venu 3 when called in high-frequency loops (e.g., drawing every bar of a graph as a polygon).

**Fix:** Replace with `dc.fillRectangle()` for any filled graph areas. The visual result is identical for vertical bars and much faster.

For isolated triangle indicators (arrows, etc.), `fillPolygon()` is acceptable — the performance issue only manifests with many calls per frame.

---

## 16. Graph Layout — graphBottom and Bezel Clearance

### The Round Bezel Problem

The Venu 3 screen is 454×454 but edges are clipped by the circular bezel. At y ≈ 380+, the visible width narrows quickly. X-axis labels placed too low will be partially or fully clipped.

### Tested Safe Layout Constants

```
topPad    = 50   → graphTop    at y=50   (header clear of top bezel)
bottomPad = 104  → graphBottom at y=350  (clear of bezel at full width)
leftPad   = 12
rightPad  = 44   (when showing Y axis)

graphW    = 454 - 12 - 44 = 398 pixels
graphH    = 350 - 50      = 300 pixels
```

X-axis labels drawn at `graphBottom + 9`, tick marks at `graphBottom` to `graphBottom + 5`.

**General rule:** Content below y=382 (bottom 72px) risks clipping. At y=350 (bottom 104px), safe for most content.

---

## 17. SensorHistory.getTemperatureHistory() — Period Parameter

For `getTemperatureHistory()` (and `getHeartRateHistory()`), `:period` is the **number of samples**, not a time duration:

```monkeyc
// Get 300 temperature samples (not 300 seconds!)
var iter = SensorHistory.getTemperatureHistory({
    :period => 300,
    :order  => SensorHistory.ORDER_OLDEST_FIRST
});
```

300 samples at ~30-minute intervals ≈ 150 hours (just over 6 days).

**Skin temperature sampling:** approximately every 30 minutes during sleep; less frequent during waking hours. Plan for 48 samples/day but handle gaps gracefully — some time buckets may have null values.

---

## 18. Page Indicator Dots (Custom Implementation)

`WatchUi.ViewLoop` (API 3.4+) provides a built-in page indicator but takes ownership of left/right swipe navigation — conflicting with horizontal graph scrolling. A custom indicator drawn in `onUpdate()` gives full control:

```monkeyc
// Draw N dots; highlight the active one
var dotR     = 4;
var dotGap   = 12;
var totalW   = (N * dotR * 2) + ((N - 1) * (dotGap - dotR * 2));
var startX   = (dc.getWidth() - totalW) / 2;
var dotY     = dc.getHeight() - 20;

for (var i = 0; i < N; i++) {
    var cx = startX + i * dotGap + dotR;
    if (i == _currentPage) {
        dc.setColor(Graphics.COLOR_WHITE, Graphics.COLOR_TRANSPARENT);
        dc.fillCircle(cx, dotY, dotR);
    } else {
        dc.setColor(Graphics.COLOR_DK_GRAY, Graphics.COLOR_TRANSPARENT);
        dc.fillCircle(cx, dotY, dotR - 1);
    }
}
```

---

## 19. Application.Storage — No Permission Required

`Application.Storage` requires **no manifest permission** and is available to all app types (widget, watch-app, data field, glance).

**Do NOT add** `<iq:uses-permission id="Storage"/>` — "Storage" is not a valid permission ID and will cause a build error.

Valid permission IDs: Ant, Background, BluetoothLowEnergy, Communications, ComplicationProvider, ComplicationSubscriber, DataFieldAlert, Fit, FitRecording, PersistedContent, Positioning, Sensor, SensorHistory, SensorLogging, UserProfile.

---

## 20. Persistence Layer Architecture (TempStorage)

### Design Overview

The `TempStorage` module provides a 7-day rolling window of daily average skin temperatures, stored in `Application.Storage` (flash, survives power cycles).

### Storage Keys

```
"st_days"  : Array<Number> — 7 epoch-day integers (days since 2000-01-01 UTC)
"st_temps" : Array<Float>  — 7 daily averages in Celsius; -999.0 = no data
```

**Why epoch-days since 2000, not Unix timestamps?** Smaller integers, more readable in debug output, and compatible with Android Mobile SDK's `Communications.transmit()` for future BLE sync.

**Why always store Celsius?** Unit-agnostic storage — if the user changes °F/°C preference, stored historical data remains correct. Convert to display units only at render time.

### Schema Migration Awareness

When adding new fields to stored records, old records won't have those keys. Always treat new fields as nullable when reading:

```monkeyc
var timeStr = r.get("time");
var dtLabel = (timeStr instanceof String)
    ? (dateStr as String) + " " + (timeStr as String)
    : (dateStr as String);   // Gracefully handles records without "time"
```

---

## Quick Reference — Key Lessons

1. Glance dc starts to the right of the launcher icon — x=0 is already past it
2. AMOLED brightness requires empirical calibration; laptop-visible values may be invisible on watch
3. Pixel-sample screenshots to match system UI colors precisely
4. Swipe UP = `onNextPage()`, Swipe DOWN = `onPreviousPage()` — confirmed on device
5. Never use `onSwipe` for primary navigation on Venu 3 — unreliable
6. `DragEvent.getCoordinates()` returns position, not delta — track previous position yourself
7. Every project needs a unique app ID — copying without changing causes silent overwrites
8. `Gregorian.moment()` uses UTC — use `System.getClockTime()` subtraction for local midnight
9. `fillPolygon` causes watchdog timeout in high-frequency loops — use `fillRectangle` for graph fills
10. `dc.setClip()` / `dc.clearClip()` restrict rendering to a rectangle — use for graph areas
11. Widget memory limit is 64 KB — use `const` for large tables, avoid allocating in `onUpdate()`
12. Changing an app ID creates a new separate app; old version stays installed until Garmin Express
13. `Application.Storage` requires no manifest permission; "Storage" is not a valid permission ID
14. Temperature history `:period` = sample count, not seconds (same as HR history)
15. `(:glance)` goes on the `getGlanceView()` function and the `GlanceView` class — never on the AppBase class. Guard expensive `onStart()` work with `isGlanceModeEnabled` because on non-live-update devices `onStart()` runs every 30 seconds in the background glance refresh cycle.

---

## 21. Glance Annotations — Complete Guide

### How the Glance Binary Is Built

The Connect IQ compiler builds three separate binaries from the same source tree:

| Process | What is included |
|---|---|
| Full app | Everything — all code, no restrictions |
| Background | AppBase (always) + `(:background)` annotated code |
| Glance | AppBase (always) + `(:glance)` annotated code + `(:background)` annotated code |

**AppBase is unconditionally compiled into every process.** It does not need and must not carry a `(:glance)` annotation.

### The Correct Annotation Pattern

Annotate the `getGlanceView()` **function** inside AppBase, and the `GlanceView` subclass. Do not annotate the AppBase class itself, and do not annotate any other AppBase methods.

```monkeyc
// SkinTempApp.mc — annotate the function, not the class
class SkinTempApp extends Application.AppBase {

    public function initialize() { AppBase.initialize(); }

    public function onStart(state as Dictionary?) as Void {
        // Guard expensive work — see onStart() section below
        var s = System.getDeviceSettings();
        if (!((s has :isGlanceModeEnabled) && s.isGlanceModeEnabled)) {
            TempStorage.update();
        }
    }

    (:glance)
    public function getGlanceView() {
        return [new SkinTempGlanceView()];
    }

    public function getInitialView() { ... }
}

// SkinTempGlanceView.mc — annotate the class
(:glance)
class SkinTempGlanceView extends WatchUi.GlanceView { ... }
```

This matches the standard used across the Connect IQ developer community and in the official Garmin developer blog. The function-level annotation explicitly marks `getGlanceView()` as the glance entry point, which silences all compiler warnings cleanly.

Any helper class that the GlanceView references should also carry `(:glance)` so it is explicitly included in the glance binary.

### Why Annotating the AppBase Class Crashes the Glance

When the AppBase *class* (not a function) carries `(:glance)`, the compiler treats the entire class like any other annotated symbol and compiles it into the glance binary — but then excludes all unannotated dependencies. AppBase's method bodies reference `TempStorage`, `DailyGraphView`, `GraphDelegate`, etc., none of which are annotated. The glance process loads, cannot resolve those references, and crashes silently. The system falls back to the default placeholder showing the app name.

Annotating only the *function* avoids this entirely. The function is scoped to glance; the class body is not affected.

### onStart() Is Called in the Glance Process Too

This is the most commonly missed consequence of the glance lifecycle, and a source of real production crashes.

On devices without live glance updates, the firmware runs the **entire app lifecycle every 30 seconds** as a background refresh: `onStart()` → `getGlanceView()` → `onLayout()` → `onShow()` → `onUpdate()` → `onHide()` → `onStop()`. This means any expensive work in `onStart()` runs silently in the background, consuming glance-process memory (capped at 32 KB), draining battery, and potentially crashing if memory is exceeded.

**The fix is to gate heavyweight `onStart()` work with `isGlanceModeEnabled`:**

```monkeyc
public function onStart(state as Dictionary?) as Void {
    var settings = System.getDeviceSettings();
    var inGlanceMode = (settings has :isGlanceModeEnabled) && settings.isGlanceModeEnabled;
    if (!inGlanceMode) {
        TempStorage.update();  // Only runs when the full widget is actually opened
    }
}
```

`isGlanceModeEnabled` is `true` when the device uses the glance carousel model (CIQ 3.1+ with glances enabled, and all CIQ 4+ devices). On older devices that have no glance support, the field does not exist — the `has` check handles this cleanly.

Note: `getInitialView()` is only ever called for the full app, never in the glance process, so it is safe to put expensive startup work there instead if `isGlanceModeEnabled` feels overly defensive.

### The 32 KB Glance Memory Limit in Practice

The glance limit is 32 KB — the same as the background process limit, and half the 64 KB widget limit. This is tighter than it appears:
- AppBase is always loaded, consuming part of that budget whether the glance uses it or not
- Background-annotated code is also included in the glance binary
- On some device variants (e.g. fēnix 6X vs fēnix 6), the larger screen canvas means the same code uses measurably more memory just from rendering

Published apps have shipped, worked on test devices, and then crashed on slightly different hardware variants with an OutOfMemory error because the peak glance memory was too close to the ceiling. The simulator's **View Memory** screen (visible at the bottom of the sim window in glance mode) shows peak memory and should be checked before release.

### Warning Reference

| Annotation state | Compiler warnings | Glance renders? |
|---|---|---|
| No `(:glance)` anywhere | "Entire application will be loaded as glance" | Yes, but wasteful |
| `(:glance)` on GlanceView class only | "Implicitly added '$.AppName'" | Yes, but entry point inferred |
| `(:glance)` on AppBase **class** | None | **No — silent crash** |
| `(:glance)` on `getGlanceView()` function + GlanceView class | None | **Yes — correct, standard pattern** |

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Core API reference, sensor permissions, drawing primitives
- `hrv_logger_development_lessons.md` — Sensor lifecycle pattern that informs widget design
- `customhrv_development_lessons.md` — `fillPolygon` usage and arrow-drawing patterns
- `venu3_practical_lessons.md` — Bezel clearance, data screen configuration
- `garmin_development_addendum.md` — App ID management, installation workflow
- `monkeyc_analyzer_unreachable_statement_guide.md` — Static analyzer issues common in widget code

---

*Last updated: March 2026 — SDK 8.4.1*
