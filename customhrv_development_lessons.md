# CustomHRV — Development Lessons
## Real-World Discoveries — February 2026

**Project:** CustomHRV Watch App for Garmin Venu 3
**SDK Version:** 8.4.1 / API Level 5.2
**App Type:** Watch App (768 KB memory limit)
**Status:** Complete and functional

This document captures practical lessons from building the CustomHRV app — a multi-view watch app for recording and analyzing HRV (heart rate variability) readings over time.

---

## Numbered Index

1. [onSelect() Does Not Fire for Row Taps on Touchscreen](#1-onselect-does-not-fire-for-row-taps-on-touchscreen)
2. [Swipe Direction vs Callback Name — Confirmed on Device](#2-swipe-direction-vs-callback-name--confirmed-on-device)
3. [Adding Time to Storage Records — Schema Migration Awareness](#3-adding-time-to-storage-records--schema-migration-awareness)
4. [HRVStorage.addReading() Signature Change](#4-hrvstorageaddreading-signature-change)
5. [Multi-Flag Data Quality System](#5-multi-flag-data-quality-system)
6. [DetailView Delete Confirmation Pattern](#6-detailview-delete-confirmation-pattern)
7. [Display Index vs Storage Index](#7-display-index-vs-storage-index)
8. [Cast Expressions Are Not lvalues](#8-cast-expressions-are-not-lvalues)
9. [Bezel Safe Zone — Full-Screen Watch App Layout](#9-bezel-safe-zone--full-screen-watch-app-layout)
10. [Unicode Arrow Characters Don't Render — Use fillPolygon Instead](#10-unicode-arrow-characters-dont-render--use-fillpolygon-instead)
11. [App List Placement — Activities vs Apps](#11-app-list-placement--activities-vs-apps)

---

## 1. onSelect() Does Not Fire for Row Taps on Touchscreen

### The Problem

The initial HistoryView used `onSelect()` in the BehaviorDelegate to handle taps on list rows. On the Venu 3 (touchscreen), tapping anywhere on the screen fires `onTap()` rather than `onSelect()`. `onSelect()` fires only when the dedicated SELECT button (if present) is physically pressed.

### Fix

Override `onTap(evt as WatchUi.ClickEvent)` in the delegate and use coordinates to determine which row was tapped:

```monkeyc
public function onTap(clickEvent as WatchUi.ClickEvent) as Boolean {
    var coords = clickEvent.getCoordinates();   // [x, y]
    var tapY   = coords[1];
    var rowIdx = _view.rowForTapY(tapY);
    if (rowIdx >= 0) {
        _view.openDetail(rowIdx);
        return true;
    }
    return false;
}
```

The view exposes a helper that maps a Y pixel to a display row index:

```monkeyc
public function rowForTapY(tapY as Number) as Number {
    if (tapY < CONTENT_Y || tapY >= FOOTER_Y) { return -1; }
    var relY = tapY - CONTENT_Y;
    var row  = relY / ROW_H;
    var displayIdx = _scrollPos + row;
    if (displayIdx >= _readings.size()) { return -1; }
    return displayIdx;
}
```

### Key Point

`WatchUi.ClickEvent.getCoordinates()` returns `[x, y]` as a two-element array. Available on all touchscreen devices including Venu 3.

See also: `garmin_connectiq_knowledge_base.md` §6 — Input and Menus.

---

## 2. Swipe Direction vs Callback Name — Confirmed on Device

**Verified by video recordings of actual gestures on the Venu 3 against live code:**

| Physical Gesture | Callback Fired |
|-----------------|----------------|
| **Swipe UP** | `onNextPage()` |
| **Swipe DOWN** | `onPreviousPage()` |

This is counterintuitive and the opposite of what the callback names suggest. It was also the opposite of what was previously documented in the skin temp widget lessons — that earlier documentation was wrong and has been corrected here.

### Confirmation Evidence (CustomHRV App)

| Gesture (video) | Behavior | Handler in code |
|----------------|----------|-----------------|
| Swipe DOWN → History | DashboardDelegate pushes HistoryView | `onPreviousPage()` |
| Swipe DOWN → Include/Exclude | DetailDelegate calls `toggleExclusion()` | `onPreviousPage()` |
| Swipe UP → Delete | DetailDelegate calls `requestDelete()` | `onNextPage()` |
| Swipe UP → Increase value | SettingAdjustDelegate calls `increment()` | `onNextPage()` |
| Swipe DOWN → Decrease value | SettingAdjustDelegate calls `decrement()` | `onPreviousPage()` |

### Why the Skin Temp Doc Got It Wrong

The skin temp widget only had two views to cycle between. Swiping up and swiping down both toggled between the same two screens, so it was impossible to observe which direction fired which callback. The incorrect assumption was written down and propagated.

### Practical Rule

When wiring up a two-action screen (e.g. increase/decrease, action-A/action-B):
- Put the action that should respond to **swipe UP** in `onNextPage()`
- Put the action that should respond to **swipe DOWN** in `onPreviousPage()`

---

## 3. Adding Time to Storage Records — Schema Migration Awareness

The initial storage schema only stored "date" (YYYY-MM-DD). When "time" (HH:MM) was added, old records in storage will not have a "time" key.

All code that reads "time" from a reading must treat it as nullable:

```monkeyc
var timeStr = r.get("time");
var dtLabel = (timeStr instanceof String)
    ? (dateStr as String) + " " + (timeStr as String)
    : (dateStr as String);
```

This pattern gracefully handles both old records (no time) and new ones.

`HRVStorage.currentTimeString()` uses `System.getClockTime()` to get the time at the moment a reading is saved, giving 24hr "HH:MM" format.

---

## 4. HRVStorage.addReading() Signature Change

When the "time" field was added, the method signature changed:

```monkeyc
// Old signature
static function addReading(rmssd as Number, qualityPct as Number) as Void

// New signature
static function addReading(rmssd as Number, qualityPct as Number, timeStr as String) as Void
```

The time string is captured at the end of the measurement (in `finishMeasurement()`) before navigating to ResultView, then passed through ResultView to the storage call in `saveReading()`.

**General principle:** When adding fields to a storage record schema, update all callers in one pass and add nullable handling to all readers. Missing either step causes runtime errors or silently lost data.

---

## 5. Multi-Flag Data Quality System

Three types of data quality flags are computed for each reading when HistoryView loads:

1. **`:qualityBad`** — the `quality` field (% valid callbacks) indicates too many empty callbacks. Threshold is the `maxEmptyPct` setting (default 40%).

2. **`:duplicateDate`** — more than one non-excluded reading shares the same "date" value. Excluded readings don't count toward the duplicate. Flag appears on ALL duplicates on that date, not just the "extra" ones.

3. **`:timeOutlier`** — the reading's time is more than `TIME_OFFSET_THRESHOLD_MINS` (currently 180 = 3 hours) away from the median time of all non-excluded readings. Uses clock-wrap-aware comparison (e.g. 23:50 and 00:10 are only 20 minutes apart).

Flags are computed in a single pass over the display array in `HRVStorage.computeFlags()`. Excluded readings are not flagged (they're already out of the baseline).

Visual indicator: any flagged non-excluded reading shows its date/time in red in the history list. The ">" chevron on the right always shows to indicate a detail view is available.

---

## 6. DetailView Delete Confirmation Pattern

Destructive actions (delete) on a small watch screen need confirmation but also need to be fast. Pattern used:

- First `onNextPage()` call → set `_confirmDel = true`, repaint footer to show "tap=CONFIRM DELETE  back=cancel"
- Second `onNextPage()` call → execute `HRVStorage.deleteReading(origIdx)` and pop view
- `onBack()` → clear `_confirmDel`, reload history, pop view

This avoids a separate confirmation dialog screen while still preventing accidental deletion. The two-tap requirement adds friction without adding a full extra screen.

---

## 7. Display Index vs Storage Index

Readings are stored oldest-first in `Application.Storage`. HistoryView reverses this for display (newest-first). This means:

```
displayIndex = 0          → most recent reading
storageIndex = n - 1      → most recent reading (where n = total count)

displayIndex i            → storageIndex (n - 1 - i)
```

When passing to `HRVStorage.toggleExclusion()` or `HRVStorage.deleteReading()`, always convert:
```monkeyc
var origIdx = _readings.size() - 1 - displayIdx;
```

The `computeFlags()` method operates on the display-order array (reversed), so flag array indices match display indices directly — no conversion needed when drawing rows.

---

## 8. Cast Expressions Are Not lvalues

### The Error

```
no viable alternative at input '(flags[i] as Dictionary)[:qualityBad] ='
```

### Root Cause

Monkey C's type system treats a cast expression as read-only — it cannot appear on the left side of an assignment. This applies to any cast, not just Dictionary element writes.

### Broken Pattern

```monkeyc
// ILLEGAL — cast result is not assignable
(flags[i] as Dictionary)[:qualityBad] = true;
(myArray[j] as MyClass).someField = value;
```

### Fix — Extract to Local Variable First

```monkeyc
// CORRECT — local variable is a proper reference
var entry = flags[i] as Dictionary;
entry[:qualityBad] = true;

var obj = myArray[j] as MyClass;
obj.someField = value;
```

The local variable holds a reference to the same object, so mutations go through correctly. No copy is made — Monkey C uses reference semantics for Dictionaries and objects.

This pattern also appears in `garmin_connectiq_knowledge_base.md` §11.

---

## 9. Bezel Safe Zone — Full-Screen Watch App Layout

On the Venu 3 (454×454 AMOLED, circular), the circular bezel cuts off content near the edges. The CustomHRV app uses a consistent layout for all full-screen views:

- Header bar: `y = 42`, height = 28 (center `y = 56`)
- Content zone: `y = 70` to `y = 355`
- Footer bar: `y = 355`, height = 24 (center `y = 367`)
- Sub-footer text: `y = 393`

This keeps all interactive/informative content within approximately ±180px of center, which is comfortably inside the circular display area. Content below `y = 410` risks clipping by the bezel.

**Note:** This layout is for full-screen views (Watch Apps, Widgets). Data field half-slots use different layout math — see `venu3_practical_lessons.md` §2 and `consolidated_field_development_lessons.md` §3-4.

---

## 10. Unicode Arrow Characters Don't Render — Use fillPolygon Instead

### The Problem

Unicode arrow characters (`\u25b2` ▲ and `\u25bc` ▼) embedded in text strings render as `?` boxes on the Venu 3. The built-in Garmin fonts do not include these glyphs. Attempts to fix this via XML string resources and `WatchUi.loadResource()` also failed — the font simply doesn't have the glyph, regardless of how the character is delivered to `dc.drawText()`.

### The Fix

Draw arrows as filled triangles using `dc.fillPolygon()`. This is device-independent, scales to any size, and never depends on font glyph availability.

```monkeyc
function drawUpArrow(dc as Graphics.Dc, cx as Number, cy as Number,
                     size as Number, color as Number) as Void {
    dc.setColor(color, Graphics.COLOR_TRANSPARENT);
    var half = size / 2;
    var vOff = (size * 0.45).toNumber();
    dc.fillPolygon([
        [cx, cy - vOff],          // tip (top)
        [cx - half, cy + vOff],   // bottom-left
        [cx + half, cy + vOff]    // bottom-right
    ]);
}

function drawDownArrow(dc as Graphics.Dc, cx as Number, cy as Number,
                       size as Number, color as Number) as Void {
    dc.setColor(color, Graphics.COLOR_TRANSPARENT);
    var half = size / 2;
    var vOff = (size * 0.45).toNumber();
    dc.fillPolygon([
        [cx, cy + vOff],          // tip (bottom)
        [cx - half, cy - vOff],   // top-left
        [cx + half, cy - vOff]    // top-right
    ]);
}
```

### Inline Arrow Hints

To render footer text like "▲/▼=Scroll  Tap=Details" with actual drawn arrows mixed into the text, use `dc.getTextDimensions()` to measure each text segment's width, then lay out arrows and text left-to-right from a computed start position:

```monkeyc
// Measure total width of all segments
var totalW = arrowSize + slashW + arrowSize + suffixW;
var startX = cx - totalW / 2;
// Draw: arrow, text, arrow, text from left to right
```

The CustomHRV project implements this in `ArrowUtils.mc` as a shared module with helper functions: `drawUpDownHint()`, `drawArrowLabel()`, and `drawUpDownPair()`. A good arrow size for `FONT_XTINY` footer text is 10 pixels.

### Key Points

- `fillPolygon()` takes `Array<Point2D>` where `Point2D` is `[Numeric, Numeric]`
- Inline array literals `[[x1,y1], [x2,y2], [x3,y3]]` work directly as the argument
- The 0.45 multiplier on size gives a visually balanced equilateral-ish triangle
- Always call `dc.setColor()` before `fillPolygon()` — it uses the foreground color
- **Performance note:** `fillPolygon` can cause watchdog timeout if called in tight loops at high frequency. In data fields updated every second, it is fine. See `skin_temp_widget_development_lessons.md` §15 for details.
- This approach works on every CIQ device, not just Venu 3

---

## 11. App List Placement — Activities vs Apps

On the Venu 3 the watch UI has three distinct places CIQ apps appear:

| Section | What appears there |
|---------|-------------------|
| **Glances** | Apps that implement `getGlanceView()`, shown in the carousel (swipe up from watch face) |
| **Apps** | `type="widget"` apps; also `type="watch-app"` apps without ActivityRecording |
| **Activities** | `type="watch-app"` apps that use `ActivityRecording` with the `Fit` permission |

### Confirmed by Testing

Three builds of a minimal HelloWorldTest app were used to isolate the variable:

| Manifest type | Permissions | ActivityRecording | Appears in |
|--------------|-------------|-------------------|------------|
| `watch-app` | none | no | **Apps** |
| `watch-app` | `Fit` | yes (create/start/discard session) | **Activities** |
| `widget` | `SensorHistory` | no | **Apps** + **Glances** |

CustomHRV (`watch-app` + `Fit` + full `ActivityRecording` usage) also appears in Activities, consistent with the second row.

### What We Have Not Isolated

The HelloWorldTest jumped from no permissions + no ActivityRecording to `Fit` + ActivityRecording in one step. It is not yet confirmed whether the `Fit` permission alone is sufficient to trigger Activities placement, or whether the actual use of `ActivityRecording.createSession()` is required (or both together). A future test adding only the `Fit` permission with no ActivityRecording code would answer this.

### Practical Notes

- Tapping a `widget`-type app from the **Apps** list skips the glance view entirely and launches straight into the full app.
- A `widget`-type app only appears in **Glances** if it implements `getGlanceView()`. Without that, it appears in Apps only.
- The Activities section groups CIQ watch-apps alongside native Garmin activity profiles (Running, Biking, etc.). This is a front-end UI distinction only — the underlying CIQ app type is the same `watch-app` in both the Apps and Activities cases.

### Permission Note — `Fit` vs `FitRecording`

The main knowledge base permission table (`garmin_connectiq_knowledge_base.md` §7) lists `FitRecording` as the permission for `ActivityRecording`. However, both CustomHRV and HelloWorldTest use `<iq:uses-permission id="Fit"/>` and work correctly. Both `Fit` and `FitRecording` are valid permission IDs. The exact difference between them has not been tested — use `Fit` as it is confirmed working for `ActivityRecording` sessions.

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Core SDK concepts, input handling, storage APIs
- `garmin_development_addendum.md` — Developer keys, installation, simulator setup
- `hrv_logger_development_lessons.md` — Beat-to-beat intervals, ActivityRecording session requirement
- `monkeyc_analyzer_unreachable_statement_guide.md` — Static analyzer warnings
- `venu3_practical_lessons.md` — Data screen configuration on Venu 3
- `skin_temp_widget_development_lessons.md` — Widget architecture, swipe navigation, `fillPolygon` performance

---

*Last updated: March 2026 — SDK 8.4.1*
