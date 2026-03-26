# StimTracker — Development Lessons

**Project:** StimTracker
**SDK Version:** 8.4.1 / API Level 5.2
**App type:** Watch App (768 KB) with Glance
**Status:** Active development — simulator-confirmed, partial device testing

This file captures lessons learned during the development of StimTracker — a caffeine-tracking watch app with a decay model, history log, profile management, and settings. Many entries are general Monkey C patterns discovered in this project that apply across all watch app development.

---

## Numbered Index

| # | Section |
|---|---------|
| 1 | Delegate/view disconnect — always pass the view to the delegate |
| 2 | Local variable type annotations cause build error |
| 3 | `onMenu()` fires on long-press of the back button |
| 4 | Circle-clipped horizontal bar technique |
| 5 | Confirmed layout constants for secondary screens (FONT_XTINY) |
| 6 | Word-wrap helper for long profile names |
| 7 | `deleteProfile()` does not affect log history |
| 8 | Settings delegate: swipes scroll, back button exits |
| 9 | Profile delete flow: correct pop count after nested confirmation |
| 10 | Glance decay loop must read yesterday's log |
| 11 | Hitbox bug: never store a live-mutating list in the delegate |
| 12 | Ghost-view bug: delegate must receive the displayed view |
| 13 | "Hold to edit" hint label pattern |
| 14 | HH:MM swipe picker template |
| 15 | Number input widget template (±buttons + centre tap → TextPicker) |
| 16 | Hitbox alignment for numeric input widgets |
| 17 | ValueEditView title word-wrap |
| 18 | Compiler warning patterns to avoid |
| 19 | FONT_NUMBER_MEDIUM rendering offset (y is top of bounding box, not visual top) |
| 20 | Sort order — array-position reordering pattern |
| 21 | Application.Storage performance — pre-load events for batch calculations |
| 22 | Bitmap resource caching — never call loadResource in onUpdate |
| 23 | Complication live updates — FaceIt does not subscribe to change callbacks |

---

## 1. Delegate/View Disconnect — Always Pass the View to the Delegate

**Problem:** When `pushView(new MyView(), new MyDelegate(), ...)` is called with the delegate creating its own private view internally, `onUpdate()` fires on the pushed view but all delegate methods that call view functions operate on the delegate's private copy. Visual updates never appear; swipe-triggered scrolls and data changes silently affect an invisible ghost object.

**Fix:** Always create the view first, pass it to both `pushView()` and the delegate constructor.

```monkeyc
// CORRECT
var myView = new MyView(data);
WatchUi.pushView(myView, new MyDelegate(myView, data), WatchUi.SLIDE_UP);

// BROKEN — delegate's internal new MyView() is a ghost
WatchUi.pushView(new MyView(data), new MyDelegate(data), WatchUi.SLIDE_UP);
```

This pattern must be used for every view/delegate pair in the app. Affected in StimTracker: HistoryDelegate, LogStimulantDelegate, EditStimulantDelegate, PreviewDelegate, SettingsDelegate, ValueEditDelegate, BackdateDelegate, ProfileEditDelegate. All fixed by passing the view as the first constructor argument.

---

## 2. Local Variable Type Annotations Cause Build Error

`var line1 as String;` inside a function body is rejected by the Venu 3 compiler:
> ERROR: Invalid explicit typing of a local variable. Local variable types are inferred.

Monkey C infers local variable types — only member variables (class-level) can have explicit `as Type` annotations. Remove the annotation from local declarations:

```monkeyc
// BROKEN
var line1 as String;
var line2 as String;

// CORRECT
var line1;
var line2;
```

---

## 3. `onMenu()` Fires on Long-Press of the Back Button

`BehaviorDelegate.onMenu()` is triggered by a long-press of the physical back/menu button on Venu 3. This can be used as a "hold back = secondary action" pattern with a labelled hint bar at the bottom of the screen. The short back press still fires `onBack()` / `onPreviousPage()` as normal. Both can coexist in the same delegate.

```monkeyc
function onBack() as Boolean {
    WatchUi.popView(WatchUi.SLIDE_RIGHT); // short press
    return true;
}
function onMenu() as Boolean {
    WatchUi.pushView(new SecondaryView(), ...); // long press
    return true;
}
```

Used in StimTracker on the Preview screen: short back = cancel; long back = push ProfileEditView.

---

## 4. Circle-Clipped Horizontal Bar Technique

To draw a dark background bar that respects the circular screen boundary (arc radius 210 for Venu 3, not the full screen radius 227), fill it row by row using the arc equation:

```monkeyc
private function _fillCircularBar(dc as Graphics.Dc, y as Number, barH as Number,
                                   color as Number) as Void {
    dc.setColor(color, Graphics.COLOR_TRANSPARENT);
    var rSq = 210 * 210;  // arc radius, NOT CY*CY
    for (var row = y; row < y + barH; row++) {
        var dy  = row - CY;  // CY = 227
        var rem = rSq - dy * dy;
        if (rem <= 0) { continue; }
        var hw = Math.sqrt(rem.toFloat()).toNumber();
        dc.fillRectangle(CX - hw, row, hw * 2, 1);
    }
}
```

Using 227 (screen radius) instead of 210 (arc radius) draws the bar slightly wider than the visual arc, causing it to clip against the bezel unexpectedly at the bottom of the screen.

---

## 5. Confirmed Layout Constants for Secondary Screens (FONT_XTINY)

After iterative pixel adjustment in the simulator, these values produce clean results on Venu 3 (454×454) with `FONT_XTINY`:

| Element | Value | Notes |
|---------|-------|-------|
| Screen safe zone | y=42 to y=412 | Inside arc boundary |
| Secondary screen title | y=28–30 | Green, centred |
| Row height (settings list) | 58px | 6 rows fit y=55 to y=403 |
| Label-to-value gap (list) | +13px / +38px | From row top |
| Inter-line spacing (data lines) | 30px | Preview screen "After this" section |
| Green action button | h=38–50px, radius=10 | `fillRoundedRectangle` |
| Cancel/hint bar | y=380, h=23 | Full-width `fillRectangle` |
| Footer circle-clipped bar | h=27 | Both main screen bars |
| Gap between stacked footer bars | 2px | |

**Note:** These are simulator-confirmed. Record device-confirmed values separately after testing.

---

## 6. Word-Wrap Helper for Long Profile Names

Monkey C `drawText()` does not wrap text automatically. To word-wrap at a character boundary without splitting words:

```monkeyc
// Returns true if wrapping occurred (caller can use this to shift dependent elements)
private function _drawWrappedName(dc as Graphics.Dc, name as String, y as Number) as Boolean {
    if (name.length() <= 22) {
        dc.drawText(CX, y, Graphics.FONT_XTINY, name,
            Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER);
        return false;
    } else {
        var splitPos = 22;
        while (splitPos > 0 && !(name.substring(splitPos, splitPos + 1).equals(" "))) {
            splitPos--;
        }
        var line1;
        var line2;
        if (splitPos == 0) {
            line1 = name.substring(0, 22);
            line2 = name.substring(22, name.length());
        } else {
            line1 = name.substring(0, splitPos);
            line2 = name.substring(splitPos + 1, name.length());
        }
        dc.drawText(CX, y - 11, Graphics.FONT_XTINY, line1,
            Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER);
        dc.drawText(CX, y + 16, Graphics.FONT_XTINY, line2,
            Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER);
        return true;
    }
}
```

The 22-char threshold and 11px/16px line offsets are tuned for `FONT_XTINY` on Venu 3. Adjust for other fonts/screens.

---

## 7. `deleteProfile()` Does Not Affect Log History

Each `logDose()` call stores a self-contained snapshot `{ name, caffeineMg, timestampSec, profileId }` in the day log. The profile list is never consulted at history display time. Deleting a profile therefore has no effect on past log entries — they display correctly forever using the `name` and `caffeineMg` that were snapshotted at log time.

`profileId` is stored but currently unused at read-time; it exists for potential future correlation (e.g. rename propagation, ingredient expansion).

---

## 8. Settings Delegate: Swipes Scroll, Back Button Exits

For a scrollable settings/list screen where you want swipes to scroll rather than navigate:
- `onNextPage()` (swipe UP) → `view.scrollDown()`
- `onPreviousPage()` (swipe DOWN) → `view.scrollUp()`
- `onBack()` → `WatchUi.popView()` (the **only** exit)

Do **not** call `popView()` inside `onPreviousPage()` even when `scrollPos == 0`; that creates an accidental navigation trigger when the user swipes down on the first item.

---

## 9. Profile Delete Flow: Correct Pop Count After Nested Confirmation

When a confirmation dialog is presented from within a deeply-pushed screen and the user confirms a destructive action that should return several levels, pop the dialog first (implicit in `onResponse` return) then explicitly pop the remaining levels:

```monkeyc
// In ProfileDeleteDelegate.onResponse (after CONFIRM_YES):
// Stack at this point: Main → LogStimulant → Preview → ProfileEdit → Confirmation
WatchUi.popView(WatchUi.SLIDE_DOWN); // dismiss Confirmation (implicit in onResponse)
WatchUi.popView(WatchUi.SLIDE_DOWN); // dismiss ProfileEdit
WatchUi.popView(WatchUi.SLIDE_DOWN); // dismiss Preview
// Now back at LogStimulant with refreshed profile list
```

The list refresh must happen **before** the pops via the `_listView` reference passed through the delegate chain, so the refreshed list is ready to display when the stack unwinds.

---

## 10. Glance Decay Loop Must Read Yesterday's Log

The glance view has its own inline pharmacokinetic decay calculation (it cannot call `StimTrackerStorage` methods directly due to the `(:glance)` annotation context). If that loop only reads `"log_" + todayKey`, residual caffeine from doses taken the previous day will be silently ignored. This causes the glance to show a lower "Now:" figure than the main app, with the discrepancy largest first thing in the morning and narrowing to zero as yesterday's doses fully decay (~35 hours after last yesterday dose at default 5h half-life).

**Fix:** Compute both `_glanceTodayKey()` and `_glanceYesterdayKey()`, loop over both log keys, add yesterday's doses to `currentMg` (decay sum) but **not** to `totalMg` (today's count). Confirmed working on device — glance and main screen now match.

---

## 11. Hitbox Bug: Never Store a Live-Mutating List in the Delegate

When a delegate is constructed with a copy of a list that the view may later mutate via `refreshProfiles()` / `refreshData()`, the delegate's copy falls out of sync. Symptoms:
- Tapping scrolled-down rows does nothing (delegate computes wrong `totalRows`)
- Long-press targets wrong profile (index lookup uses stale array)
- Rows beyond the initial visible count are unreachable

**Fix:** Remove the list member from the delegate entirely. Add a `getProfiles()` (or equivalent) accessor to the view and call `_view.getProfiles()` inside `onTap` / `onHold` to get the live array at event time. The view is the single source of truth.

Also move the bounds check (`rowIdx >= totalRows`) into `rowForTapY()` on the view, so that it is always evaluated against the live list size.

---

## 12. Ghost-View Bug: Delegate Must Receive the Displayed View

If a delegate constructs its own view instance internally (e.g. `_view = new HistoryView(days)` inside `initialize()`), then `pushView` is called with a *different* view object. The displayed view and the delegate's `_view` are two separate instances. Scroll state (`_scrollPos`) and any other mutable state live in the delegate's phantom view, not the one on screen.

Result: `scrollDown()` / `scrollUp()` calls do nothing visible; `rowForTapY()` always evaluates against `_scrollPos = 0`; tapping any row below the initial viewport is silently ignored or resolves to the wrong row.

**Fix:** Always create the view first, pass it to both `pushView` and the delegate constructor.

```monkeyc
var histView = new HistoryView(days);
WatchUi.pushView(histView, new HistoryDelegate(histView), WatchUi.SLIDE_DOWN);
```

This is the same root cause as §1 — see there for the full explanation.

---

## 13. "Hold to Edit" Hint Label Pattern

For scrollable list screens where long-press reveals an edit action, add a dim grey hint label between the screen title and the first list row. This communicates the gesture without taking up a full row.

```monkeyc
// In onUpdate(), after drawing the title:
dc.setColor(Graphics.COLOR_DK_GRAY, Graphics.COLOR_TRANSPARENT);
dc.drawText(CX, 65, Graphics.FONT_XTINY, "Hold to edit",
    Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER);
```

The hint sits at y=65 (LogStimulantView) or y=74 (DayDetailView). If the hint is added to an existing screen, shift `LIST_TOP` down by ~15px to keep the first row from overlapping it. DayDetailView: LIST_TOP moved 80 → 95 when hint was added.

---

## 14. HH:MM Swipe Picker Template

For editing a time stored as total minutes (e.g. bedtime), use a two-column swipe picker rather than a raw integer editor. Swipe UP/DOWN increments/decrements the selected column; tapping the left half selects hour, right half selects minute.

**View skeleton:**
```monkeyc
class BedtimeEditView extends WatchUi.View {
    private const CX = 227;
    private var _hourVal   as Number;
    private var _minVal    as Number;
    private var _selCol    as Number;  // 0 = hour, 1 = min

    function initialize(bedtimeMinutes as Number) {
        View.initialize();
        _hourVal = bedtimeMinutes / 60;
        _minVal  = bedtimeMinutes % 60;
        _selCol  = 0;
    }

    function increment() as Void {
        if (_selCol == 0) { _hourVal = (_hourVal + 1)  % 24; }
        else              { _minVal  = (_minVal  + 1)  % 60; }
        WatchUi.requestUpdate();
    }

    function decrement() as Void {
        if (_selCol == 0) { _hourVal = (_hourVal + 23) % 24; }
        else              { _minVal  = (_minVal  + 59) % 60; }
        WatchUi.requestUpdate();
    }

    function selectCol(tapX as Number) as Void {
        _selCol = (tapX < CX) ? 0 : 1;
        WatchUi.requestUpdate();
    }

    function getMinutes() as Number { return _hourVal * 60 + _minVal; }
}
```

**Delegate skeleton:**
```monkeyc
class BedtimeEditDelegate extends WatchUi.BehaviorDelegate {
    function onNextPage() as Boolean {     // swipe UP
        _view.increment(); return true;
    }
    function onPreviousPage() as Boolean { // swipe DOWN
        _view.decrement(); return true;
    }
    function onBack() as Boolean {
        WatchUi.popView(WatchUi.SLIDE_RIGHT); return true;
    }
}
```

**Key points:**
- Store bedtime as total minutes (0–1439): `bedtimeMinutes = hour*60 + min`
- `onNextPage()` / `onPreviousPage()` map to swipe UP / swipe DOWN (Venu 3 confirmed)
- Digit columns wrap: hour 23→0, minute 59→0 (use modulo with offset for decrement: `(val + max - 1) % max`)

The same pattern is used in `AdjustTimeView` for the Start/Finish time picker — see that class for the full four-column (SH, SM, FH, FM) implementation.

---

## 15. Number Input Widget Template (±Buttons + Centre Tap → TextPicker)

The standard pattern for editing a numeric value on a secondary screen. The number is centred; `[-]` and `[+]` buttons flank it; tapping the number itself opens `WatchUi.TextPicker` pre-populated with the current value so the user can type directly.

**View hit-test functions (tune TOP_Y per screen — see §16):**
```monkeyc
function isMinusTap(tapX as Number, tapY as Number) as Boolean {
    return tapX >= 20 && tapX <= 145 && tapY >= TOP_Y && tapY <= SAVE_TOP;
}
function isPlusTap(tapX as Number, tapY as Number) as Boolean {
    return tapX >= 309 && tapX <= 434 && tapY >= TOP_Y && tapY <= SAVE_TOP;
}
function isNumberTap(tapX as Number, tapY as Number) as Boolean {
    return tapX >= 145 && tapX <= 309 && tapY >= TOP_Y && tapY <= SAVE_TOP;
}
```

**Delegate onTap handling:**
```monkeyc
if (_view.isMinusTap(tapX, tapY)) { _view.decrementMg(); return true; }
if (_view.isPlusTap(tapX, tapY))  { _view.incrementMg(); return true; }
if (_view.isNumberTap(tapX, tapY)) {
    WatchUi.pushView(
        new WatchUi.TextPicker(_view._caffMg.toString()),
        new CaffTextPickerDelegate(_view),
        WatchUi.SLIDE_UP
    );
    return true;
}
```

**TextPicker delegate:**
```monkeyc
class CaffTextPickerDelegate extends WatchUi.TextPickerDelegate {
    function onTextEntered(text as String, changed as Boolean) as Boolean {
        var num = text.toNumber();
        if (num != null) {
            if (num < 10)   { num = 10; }
            if (num > 1000) { num = 1000; }
            _editView._caffMg = num;
            WatchUi.requestUpdate();
        }
        return true;
    }
    function onCancel() as Boolean { return true; }
}
```

**Increment/decrement on the view:**
```monkeyc
function decrementMg() as Void {
    if (_caffMg > 10) { _caffMg -= 10; }
    WatchUi.requestUpdate();
}
function incrementMg() as Void {
    if (_caffMg < 1000) { _caffMg += 10; }
    WatchUi.requestUpdate();
}
```

**Drawing the number and flanking buttons:**
```monkeyc
// See §19 for why NUMBER_Y must account for font rendering offset
dc.drawText(CX, NUMBER_Y, Graphics.FONT_NUMBER_MEDIUM, _caffMg.toString(),
    Graphics.TEXT_JUSTIFY_CENTER);  // NO VCENTER — see §19
dc.setColor(Graphics.COLOR_BLUE, Graphics.COLOR_TRANSPARENT);
dc.drawText(110, NUMBER_Y, Graphics.FONT_NUMBER_MEDIUM, "-", Graphics.TEXT_JUSTIFY_CENTER);
dc.drawText(344, NUMBER_Y, Graphics.FONT_NUMBER_MEDIUM, "+", Graphics.TEXT_JUSTIFY_CENTER);
```

One `CaffTextPickerDelegate` subclass is needed per view type (they differ only in the view type annotation). Naming convention: `[ScreenName]CaffTextPickerDelegate`.

---

## 16. Hitbox Alignment for Numeric Input Widgets

The hit zones for `[-]`, number, and `[+]` must be calibrated per screen. The governing rules:

**Bottom:** Align with the top Y of the Save button immediately below — zero gap, no overlap.  
**Top:** 80–85px above the bottom (roughly covers the glyph plus comfortable tap margin).  
**Horizontal:**
- `[-]`: x 20–145 (left zone)
- Number: x 145–309 (centre zone)
- `[+]`: x 309–434 (right zone)

Note: for DoseEditView the `[-]` and `[+]` x-bounds are slightly different (20–165 and 289–434) because the glyph positions differ on that screen.

**Per-screen values (simulator-confirmed):**

| Screen | topY | bottomY | Save top | Notes |
|--------|------|---------|----------|-------|
| ValueEditView (Settings) | 220 | 305 | 305 | |
| ProfileEditView | 225 | 305 | 308 | |
| EditStimulantView | 200 | 280 | 305 | Bottom 25px above Save |
| MiscCaffeineView | 165 | 245 | 288 | Bottom 43px above Preview button |
| DoseEditView (History) | 230 | 288 | 288 | Different x-bounds — see above |

**Important:** The hitbox top should NOT extend up to the label area ("Caffeine (mg):", etc.). Keep it to the number zone only.

---

## 17. ValueEditView Title Word-Wrap

Settings titles for the `ValueEditView` editor may exceed 16 characters. Split the title at the space nearest to the midpoint and draw two lines:

```monkeyc
private function _drawTitle(dc as Graphics.Dc) as Void {
    if (_title.length() <= 16) {
        dc.drawText(CX, 46, Graphics.FONT_XTINY, _title,
            Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER);
    } else {
        var mid  = _title.length() / 2;
        var best = -1;
        var dist = 999;
        for (var i = 0; i < _title.length(); i++) {
            if (_title.substring(i, i + 1).equals(" ")) {
                var d = (i - mid).abs();
                if (d < dist) { dist = d; best = i; }
            }
        }
        if (best < 0) {
            dc.drawText(CX, 30, Graphics.FONT_XTINY,
                _title.substring(0, mid), Graphics.TEXT_JUSTIFY_CENTER);
            dc.drawText(CX, 63, Graphics.FONT_XTINY,
                _title.substring(mid, _title.length()), Graphics.TEXT_JUSTIFY_CENTER);
        } else {
            dc.drawText(CX, 30, Graphics.FONT_XTINY,
                _title.substring(0, best), Graphics.TEXT_JUSTIFY_CENTER);
            dc.drawText(CX, 63, Graphics.FONT_XTINY,
                _title.substring(best + 1, _title.length()), Graphics.TEXT_JUSTIFY_CENTER);
        }
    }
}
```

Line 1 at y=30, line 2 at y=63. Single-line titles draw at y=46 (midpoint between the two).

---

## 18. Compiler Warning Patterns to Avoid

**Unused local variable:**
```monkeyc
// BAD — triggers warning if never read
var w = dc.getWidth();
// FIX — remove the assignment
```

**Unused member variable after refactoring:**
If a member is assigned in `initialize()` but never read (e.g. after switching to `_view._field`), the analyser warns. Remove the dead member declaration.

**Unreachable branch due to Boolean member:**
The static analyser traces from `initialize()`. If a Boolean member is initialised to `false` and only set to `true` via a path the analyser considers unreachable, the `if (_flag)` branch is flagged as dead code. Prefer opaque API values (`info.timerState`) over Boolean flags where possible. See `monkeyc_analyzer_unreachable_statement_guide.md` for full detail.

---

## 19. FONT_NUMBER_MEDIUM Rendering Offset (Visual Top ≠ Y Coordinate)

**Discovery:** `drawText()` called with `FONT_NUMBER_MEDIUM` and **no** `TEXT_JUSTIFY_VCENTER` flag places the y coordinate at the top of the font's bounding box — which includes substantial internal ascent/leading padding built into the font metrics. The visual top of the digit glyph appears significantly *below* the y value specified.

**Measured values (Venu 3, simulator confirmed matching device coordinates):**
- Code: `drawText(CX, 219, FONT_NUMBER_MEDIUM, value, TEXT_JUSTIFY_CENTER)`
- Visual top of digit: y ≈ 244 (~25px below specified y)
- Visual bottom of digit: y ≈ 303
- Visual glyph height: ~59px

This also explains why hitbox tops historically needed to be set higher than the apparent number position — the glyph renders lower than the code coordinate suggests.

**Contrast with VCENTER behaviour:**
When `TEXT_JUSTIFY_VCENTER` is also passed, the y coordinate becomes the visual vertical centre of the text, and no offset correction is needed. However `FONT_NUMBER_MEDIUM` is typically drawn without VCENTER in this codebase (using `TEXT_JUSTIFY_CENTER` only), so the offset applies.

**FONT_XTINY with TEXT_JUSTIFY_VCENTER** behaves as expected — y is the visual centre. No correction needed for label text drawn with both `TEXT_JUSTIFY_CENTER | TEXT_JUSTIFY_VCENTER`.

**Rule of thumb:** When positioning `FONT_NUMBER_MEDIUM` without VCENTER, subtract ~25px from where you want the visual top to appear to get the code y value to use. Verify visually in the simulator with pixel ruler.

**Confirmed across screens:** The manual pixel compensation applied throughout StimTracker has been device-verified as correct. The following screens all use large number fonts without VCENTER and have been confirmed to have no visible misalignment between glyphs and hitboxes on the Venu 3:

| Screen | Font |
|--------|------|
| ProfileEditView (caffeine number) | `FONT_NUMBER_MEDIUM` |
| ValueEditView (Settings value) | `FONT_NUMBER_HOT` |
| AdjustTimeView (time columns) | `FONT_NUMBER_MILD` |
| MainView (current mg display) | `FONT_NUMBER_HOT` |

Each of these uses a different font size, so the exact offset in pixels varies — but the pattern (y is bounding-box top, visual glyph appears lower) holds for all of them.

---

## 20. Sort Order — Array-Position Reordering Pattern

Sort order for stimulant profiles is stored as **array position**, not as a dedicated field on each profile. Changing a profile's order means a remove-and-insert operation on the array itself.

### Storage function

```monkeyc
static function reorderProfile(fromIdx as Number, toIdx as Number) as Array<Dictionary> {
    // Guards: no-op if indices equal or out of bounds
    // 1. Remove item at fromIdx
    // 2. Build 'without' array
    // 3. Insert at toIdx position
    // 4. Save and return updated array
}
```

No migration is needed for existing profiles — the order they happen to be stored in simply becomes their initial sort order.

### ProfileEditView widget

`ProfileEditView` receives `sortOrder as Number` (1-based) and `totalProfiles as Number` in its constructor. It exposes a tap-to-activate swipe widget:

- Tap anywhere in y=120–200 → `_sortOrderSelected = true`; arrows turn green; dark highlight box appears behind the number
- Swipe UP → increment (clamped at `_totalProfiles`)
- Swipe DOWN → decrement (clamped at 1)
- Both swipe handlers always return `true` to consume the event — the back button is the only cancel
- Highlight box: `fillRoundedRectangle(CX-26, 150, 52, 46, 6)` — 46px height was arrived at through simulator tuning (initial 30px was too short for the glyph)

### Save sequence

`_save()` must call `reorderProfile(oldIdx, newIdx)` **before** `updateProfile(id, name, caffMg)`. This is safe because `updateProfile` finds the profile by ID, not by index — so it locates the correct entry even after the array has been reshuffled.

```monkeyc
private function _save() as Void {
    var oldIdx = _view._originalSortOrder - 1;
    var newIdx = _view._sortOrder - 1;
    if (oldIdx != newIdx) {
        StimTrackerStorage.reorderProfile(oldIdx, newIdx);
    }
    var updated = StimTrackerStorage.updateProfile(id, _view._name, _view._caffMg);
    // ... refresh views
}
```

### Callers

Both entry points to `ProfileEditView` (from `PreviewDelegate.onMenu` and `LogStimulantDelegate.onHold`) must search the live profiles array for the matching profile ID to determine the current index, then pass `idx + 1` and `profiles.size()` to the constructor. Do not pass a stale index stored at an earlier point.

### Sort order display on Log Stimulant screen

Each real profile row on the Log Stimulant screen shows its sort order number at the far left of the caffeine mg line (x=50, `TEXT_JUSTIFY_LEFT`, same y and font as the mg text). This gives the user a visual reference when deciding what number to set in the sort order widget.

```monkeyc
dc.drawText(50, y + 55, Graphics.FONT_XTINY,
    rowIdx.toString(),
    Graphics.TEXT_JUSTIFY_LEFT | Graphics.TEXT_JUSTIFY_VCENTER);
```

`rowIdx` here is the loop variable directly (1-based profile rows start at rowIdx=1).

---

## 21. Application.Storage Performance — Pre-Load Events for Batch Calculations

**Problem:** `Application.Storage.getValue()` is the main I/O operation on Connect IQ. Its per-call cost is not documented but is empirically significant at scale. In StimTracker, the core PK function `calcMgAt()` reads the day index (1 Storage read) plus each day’s log (up to 30 reads) every time it is called. When the bisection peak-finder called `calcMgAt` 50–100 times per frame, this produced **1,500–3,100 Storage reads per onUpdate()** — enough to cause a visible 2+ second freeze.

Even outside the bisection, the main screen called `calcMgAt` 4 times per frame (1× current + 3× trend), producing ~124 Storage reads per frame for data that hadn’t changed between calls.

**Fix: Pre-load once, pass through.** Load all events from Storage once into a flat array at the start of a computation batch, then pass that array into every evaluation function. The per-evaluation functions do the same PK math but skip Storage entirely.

```monkeyc
// Load once (~31 Storage reads)
static function _loadAllEvents() as Array {
    var dateStrings = loadDayIndex();
    var allEvents = [] as Array;
    for (var d = 0; d < dateStrings.size(); d++) {
        var dayLog = loadDayLog(dateStrings[d] as String);
        for (var i = 0; i < dayLog.size(); i++) {
            allEvents.add(dayLog[i]);
        }
    }
    return allEvents;
}

// Use in batch: zero Storage reads per call
static function _calcMgAtFromEvents(events as Array, asOfSec as Number,
                                    halfLifeHrs as Float, absorptionModel as Number,
                                    standardFoodState as Number,
                                    ke as Float, ln2 as Float) as Float {
    // Same PK math as calcMgAt, iterating events array instead of reading Storage
}
```

**Impact (StimTracker):**

| Scenario | Before (Storage reads/frame) | After |
|---|---|---|
| Main screen, no recording | ~124 | ~31 |
| Main screen, active recording | ~3,200 | ~31 |
| Preview screen (bisection) | ~1,500–3,100 | ~31 |

**General principle:** If you call a function that reads Storage multiple times within a single computation, and the underlying data cannot change between calls, pre-load the data once and pass it through. This applies to any iterative algorithm (bisection, search, trend calculation) where the same Storage keys are re-read on every iteration.

### Companion optimization: combine related callers

When two public functions both need the same Storage data, provide a combined function that loads once for both. In StimTracker, `calcCurrentMg()` and `calcTrendLevel()` both read the same events — a combined `calcTrendAndCurrent()` loads events once and evaluates all 4 time points (now, +60s, +900s for trend, now for current) in one pass.

### Rate-limiting time-dependent recomputation

When a cached value is time-dependent (e.g., sleep threshold during an active recording where the projected finish drifts), but the displayed precision is coarser than the recomputation rate, use an in-memory cache with a TTL:

```monkeyc
static var _recCacheSec = 0;
static var _recCacheVal = 0;

static function getCachedValue(settings as Dictionary) as Number {
    var nowSec = Time.now().value().toNumber();
    if (_recCacheSec > 0 && (nowSec - _recCacheSec) < 60) {
        return _recCacheVal;
    }
    var result = _expensiveComputation(settings);
    _recCacheSec = nowSec;
    _recCacheVal = result;
    return result;
}
```

Display resolution of 1 minute means a 60-second TTL introduces at most one frame of visible delay on the minute boundary. Invalidate the cache (set `_recCacheSec = 0`) at every mutation point.

---

## 22. Bitmap Resource Caching — Never Call loadResource in onUpdate

**Problem:** `WatchUi.loadResource()` loads a bitmap from flash storage every time it is called. If called inside `onUpdate()`, the same bitmap is reloaded on every frame render. This is wasteful and measurable on device.

**Fix:** Load bitmap resources once in `initialize()` and store them as member variables. Reference the member in `onUpdate()` and any drawing helpers.

```monkeyc
// CORRECT — load once
var _gearBmp as WatchUi.BitmapResource;
var _oopsBmp as WatchUi.BitmapResource;

function initialize(settings as Dictionary) {
    View.initialize();
    _gearBmp = WatchUi.loadResource(Rez.Drawables.GearIcon) as WatchUi.BitmapResource;
    _oopsBmp = WatchUi.loadResource(Rez.Drawables.OopsHeart) as WatchUi.BitmapResource;
}

function onUpdate(dc as Graphics.Dc) as Void {
    dc.drawBitmap(x, y, _gearBmp);  // member reference, no I/O
    dc.drawBitmap(x, y, _oopsBmp);  // member reference, no I/O
}

// BROKEN — reloads from flash every frame
private function _drawOopsButton(dc as Graphics.Dc) as Void {
    var heart = WatchUi.loadResource(Rez.Drawables.OopsHeart);  // flash I/O every frame!
    dc.drawBitmap(CX - 26, 26, heart);
}
```

This applies to all bitmap resources: icons, buttons, decorations. If you see `loadResource` inside any function that runs per-frame (`onUpdate`, `onShow`, drawing helpers called from `onUpdate`), move it to `initialize()`.

---

## 23. Complication Live Updates — FaceIt Does Not Subscribe to Change Callbacks

**Problem:** StimTracker publishes a complication via `Complications.updateComplication()` from a background service every 5 minutes. The push succeeds without error. However, the FaceIt watch face only displays the updated value after a screen transition (leaving and returning to the watch face). It does not update in real time while the face is displayed, even after hours.

**Root cause:** The Connect IQ complication system is publish/subscribe. For a watch face to receive live updates, it must:

1. Call `Complications.registerComplicationChangeCallback(method(:onChanged))`
2. Call `Complications.subscribeToUpdates(complicationId)`
3. Call `WatchUi.requestUpdate()` inside the callback to trigger a redraw

FaceIt reads the complication value when the watch face renders (on transitions), but **does not subscribe to live change callbacks**. This is a confirmed FaceIt limitation reported by multiple CIQ developers on the Garmin forums (thread: “Face It and CIQ Complications”, Nov 2023 — developer jim_m_58 confirmed his own watch faces update in real time but FaceIt does not).

**Nothing on the publisher side can fix this.** The `updateComplication()` call correctly writes the value to the system’s complication store. Whether and when a subscriber redraws is entirely the subscriber’s responsibility.

**The complications.xml and generated JSON are correct** — the JSON is auto-generated by the compiler from the XML and has no configurable fields that affect subscription behavior.

**Path forward:** A custom CIQ watch face that implements the full subscribe + callback + requestUpdate chain will receive live complication updates within seconds of each background push. This is the only reliable path for real-time complication display.

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — core SDK reference; §5 covers drawText, fonts, text justification; §13 covers performance
- `customhrv_development_lessons.md` — §1–2: confirmed swipe/tap mapping; §10: unicode arrow rendering
- `monkeyc_analyzer_unreachable_statement_guide.md` — full detail on static analyser warnings (see §18)
- `skin_temp_widget_development_lessons.md` — §21: glance annotation pattern used in StimTracker

---

*Last updated: March 2026 — SDK 8.4.1*
