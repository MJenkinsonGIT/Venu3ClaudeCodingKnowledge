# Garmin Connect IQ — Code Patterns & Templates
## Practical Implementations for Venu 3 Development

**SDK Version:** 8.4.1 / API Level 5.2

---

## Numbered Index

1. [SensorHistory — Heart Rate History Iterator](#1-sensorhistory--heart-rate-history-iterator)
2. [SimpleDataField — Basic Single-Value Field](#2-simpledatafield--basic-single-value-field)
3. [DataField — Adaptive Layout with Dynamic Font Selection](#3-datafield--adaptive-layout-with-dynamic-font-selection)
4. [Input — Button and Touch Handling](#4-input--button-and-touch-handling)
5. [Manifest.xml Patterns](#5-manifestxml-patterns)
6. [Common Patterns for Data Fields](#6-common-patterns-for-data-fields)
7. [Best Practices](#7-best-practices)
8. [Code Organization](#8-code-organization)

---

## 1. SensorHistory — Heart Rate History Iterator

### Capability-Guarded Iterator Access

Always guard SensorHistory calls with `has` checks — the API is optional and may not exist on all devices:

```monkeyc
private function getHRIterator() as SensorHistoryIterator? {
    if ((Toybox has :SensorHistory) && (SensorHistory has :getHeartRateHistory)) {
        return SensorHistory.getHeartRateHistory({
            :period => 100,   // NUMBER OF SAMPLES — not seconds!
            :order  => SensorHistory.ORDER_OLDEST_FIRST,
        });
    }
    return null;
}
```

**Key fact:** `:period` for HR history is the **number of samples**, not a time duration in seconds.

### Safe Iteration Pattern

```monkeyc
var iter = getHRIterator();
if (iter != null) {
    // Quick stats without iterating:
    var minHR = iter.getMin();
    var maxHR = iter.getMax();

    var sample = iter.next();
    while (sample != null) {
        var hr        = sample.data;   // Heart rate value (Number or null — always check)
        var timestamp = sample.when;   // Time.Moment

        if (hr != null) {
            // Use hr value here
        }
        sample = iter.next();
    }
}
```

**Rules:**
- Always null-check the iterator before use
- Always null-check `sample.data` — gaps produce null values
- Use `getMin()` / `getMax()` for quick stats without a full iteration

### Drawing a Heart Rate Graph

```monkeyc
var graphLeft   = 15;
var graphRight  = dc.getWidth() - 15;
var graphBottom = dc.getHeight() / 2 + 45;
var graphHeight = 90;
var graphWidth  = graphRight - graphLeft;
var hrMin       = 50.0f;
var hrRange     = 140.0f;

var iter = getHRIterator();
if (iter != null) {
    var totalSamples = 100;
    var xStep = graphWidth.toFloat() / (totalSamples - 1);
    var i = 0;
    var prev = iter.next();
    var curr = iter.next();

    while (prev != null && curr != null) {
        if (prev.data != null && curr.data != null) {
            var x1 = (graphLeft + xStep * i).toNumber();
            var x2 = (graphLeft + xStep * (i + 1)).toNumber();
            var y1 = graphBottom - ((prev.data - hrMin) / hrRange * graphHeight).toNumber();
            var y2 = graphBottom - ((curr.data - hrMin) / hrRange * graphHeight).toNumber();
            dc.setColor(Graphics.COLOR_GREEN, Graphics.COLOR_TRANSPARENT);
            dc.drawLine(x1, y1, x2, y2);
        }
        i++;
        prev = curr;
        curr = iter.next();
    }
}
```

---

## 2. SimpleDataField — Basic Single-Value Field

Use `SimpleDataField` when you need to display a single value and are happy for Garmin to handle formatting and rendering. Garmin renders the label above the value automatically.

```monkeyc
class MySimpleField extends WatchUi.SimpleDataField {

    private var _counter as Number;

    public function initialize() {
        SimpleDataField.initialize();
        label = "Steps";   // Displayed above the value
        _counter = 0;
    }

    // Return a Number, Float, String, Duration, or null
    public function compute(info as Activity.Info) as Numeric or Duration or String or Null {
        // Rotate through three values as an example
        var result = "--" as Numeric or Duration or String or Null;

        if (_counter == 0) {
            if (info.currentHeartRate != null) {
                result = info.currentHeartRate;
            }
        } else if (_counter == 1) {
            var dist = info.elapsedDistance;
            if (dist != null) {
                result = (dist * 0.000621371f);   // Metres → miles
            }
        } else if (_counter == 2) {
            var elapsed = info.timerTime;
            if (elapsed != null) {
                var secs = (elapsed * 0.001).toNumber();
                result = Time.Gregorian.duration({:seconds => secs});
            }
        }

        _counter = (_counter + 1) % 3;
        return result;
    }
}
```

**When to use SimpleDataField vs DataField:**

| Feature | SimpleDataField | DataField |
|---------|----------------|-----------|
| Rendering | Garmin handles | Custom `onUpdate()` |
| Full-screen auto-view | No | Yes — automatic dual view |
| Multiple values | No | Yes |
| Complexity | Minimal | More code, full control |

---

## 3. DataField — Adaptive Layout with Dynamic Font Selection

A full `DataField` with custom `onUpdate()` gives complete control over layout. Garmin also automatically creates a full-screen page alongside the field slot — the same `onUpdate()` code runs at full 454×454 resolution there, so use percentage-based positioning throughout.

### Dynamic Font Selection Helper

Finds the largest font whose rendered test string fits within available dimensions:

```monkeyc
private function pickFont(dc as Dc, availW as Number, availH as Number) as FontDefinition {
    var fonts = [
        Graphics.FONT_XTINY,
        Graphics.FONT_TINY,
        Graphics.FONT_SMALL,
        Graphics.FONT_MEDIUM,
        Graphics.FONT_LARGE,
        Graphics.FONT_NUMBER_MILD,
        Graphics.FONT_NUMBER_MEDIUM
    ] as Array<FontDefinition>;

    var testStr = "888";   // Use worst-case test string for your content
    for (var i = fonts.size() - 1; i > 0; i--) {
        var dim = dc.getTextDimensions(testStr, fonts[i]);
        if (dim[0] <= availW && dim[1] <= availH) {
            return fonts[i];
        }
    }
    return Graphics.FONT_XTINY;
}
```

### Complete Adaptive DataField Skeleton

```monkeyc
class AdaptiveField extends WatchUi.DataField {

    private const BORDER_PAD = 4;

    private var _dataFont as FontDefinition;
    private var _xCenter  as Number;
    private var _yCenter  as Number;
    private var _value    as Number;

    public function initialize() {
        DataField.initialize();
        _dataFont = Graphics.FONT_XTINY;
        _xCenter  = 0;
        _yCenter  = 0;
        _value    = 0;
    }

    public function onLayout(dc as Dc) as Void {
        // Calculate positions once — store as members, never recalculate in onUpdate()
        var w = dc.getWidth();
        var h = dc.getHeight();
        _xCenter  = w / 2;
        _yCenter  = h / 2;
        _dataFont = pickFont(
            dc,
            w - (2 * BORDER_PAD),
            h - (2 * BORDER_PAD)
        );
    }

    public function compute(info as Activity.Info) as Void {
        // Do all data work here, NOT in onUpdate()
        if (info.currentHeartRate != null) {
            _value = info.currentHeartRate;
        }
    }

    public function onUpdate(dc as Dc) as Void {
        // Drawing only — use pre-calculated members
        var bgColor = getBackgroundColor();
        var fgColor = (bgColor == Graphics.COLOR_WHITE)
            ? Graphics.COLOR_BLACK
            : Graphics.COLOR_WHITE;

        dc.setColor(fgColor, bgColor);
        dc.clear();
        dc.setColor(fgColor, Graphics.COLOR_TRANSPARENT);
        dc.drawText(
            _xCenter, _yCenter,
            _dataFont,
            _value.format("%d"),
            Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER
        );
    }

    // Required lifecycle stubs:
    public function onTimerStart()  as Void { }
    public function onTimerStop()   as Void { }
    public function onTimerPause()  as Void { }
    public function onTimerResume() as Void { }
    public function onTimerLap()    as Void { }
    public function onTimerReset()  as Void { }
}
```

---

## 4. Input — Button and Touch Handling

### Full BehaviorDelegate Template

```monkeyc
class MyDelegate extends WatchUi.BehaviorDelegate {

    public function initialize() { BehaviorDelegate.initialize(); }

    // Physical SELECT button only — does NOT fire for screen taps
    public function onSelect() as Boolean {
        return true;
    }

    // Swipe UP fires onNextPage(); Swipe DOWN fires onPreviousPage()
    // (Counterintuitive — confirmed on Venu 3; see customhrv_development_lessons.md §2)
    public function onNextPage() as Boolean {
        return true;
    }

    public function onPreviousPage() as Boolean {
        return true;
    }

    public function onKey(evt as WatchUi.KeyEvent) as Boolean {
        if (evt.getKey() == WatchUi.KEY_MENU) {
            // KEY_MENU (top-right button) — reassignable
            return true;  // Consume event
        }
        return false;     // Pass through
    }

    // Use onTap() for screen taps on touchscreen devices, NOT onSelect()
    public function onTap(evt as WatchUi.ClickEvent) as Boolean {
        var coords = evt.getCoordinates();   // [x, y]
        var tapY   = coords[1];
        // Use tapY to determine which row/region was tapped
        return true;
    }

    public function onHold(evt as WatchUi.ClickEvent) as Boolean {
        return true;
    }

    public function onBack() as Boolean {
        return false;   // Return false to allow default back navigation
    }
}
```

**Venu 3 input notes:**
- `onSelect()` = physical SELECT button only, never screen tap
- `onTap()` = touchscreen tap — use `getCoordinates()` to map to regions
- `KEY_ESC` fires before the app delegate receives it; cannot be intercepted
- `KEY_MENU` (top-right button) is the recommended choice for custom actions
- Never use `onSwipe()` for primary navigation — unreliable on Venu 3

---

## 5. Manifest.xml Patterns

### Correct Data Field Manifest

```xml
<?xml version="1.0"?>
<iq:manifest version="3" xmlns:iq="http://www.garmin.com/xml/connectiq">
    <iq:application
        entry="MyDataFieldApp"
        id="UNIQUE-UUID-HERE"
        launcherIcon="@Drawables.LauncherIcon"
        minApiLevel="5.2.0"
        name="@Strings.AppName"
        type="datafield"
        version="1.0.0">

        <iq:products>
            <iq:product id="venu3"/>
        </iq:products>

        <iq:permissions>
            <iq:uses-permission id="Sensor"/>
            <iq:uses-permission id="SensorHistory"/>
        </iq:permissions>

        <iq:languages>
            <iq:language>eng</iq:language>
        </iq:languages>
        <!-- NO fit-contrib-activities for data fields — they work in all activities automatically -->
    </iq:application>
</iq:manifest>
```

See `garmin_connectiq_knowledge_base.md` §7 for the full permission reference table.

---

## 6. Common Patterns for Data Fields

### Circular Buffer

```monkeyc
class CircularBuffer {
    private var _buf        as Array<Number or Null>;
    private var _size       as Number;
    private var _writeIdx   as Number;
    private var _count      as Number;

    public function initialize(size as Number) {
        _size     = size;
        _buf      = new [size] as Array<Number or Null>;
        _writeIdx = 0;
        _count    = 0;
    }

    public function add(val as Number) as Void {
        _buf[_writeIdx] = val;
        _writeIdx = (_writeIdx + 1) % _size;
        if (_count < _size) { _count++; }
    }

    public function getAverage() as Number or Null {
        if (_count == 0) { return null; }
        var sum = 0;
        for (var i = 0; i < _count; i++) {
            var v = _buf[i];
            if (v != null) { sum += v; }
        }
        return sum / _count;
    }

    public function getMostRecent() as Number or Null {
        if (_count == 0) { return null; }
        var idx = (_writeIdx - 1 + _size) % _size;
        return _buf[idx];
    }

    public function getCount() as Number { return _count; }
}
```

### Multi-Algorithm Manager

```monkeyc
class AlgorithmManager {
    private var _algorithms  as Array;
    private var _names       as Array<String>;
    private var _currentIdx  as Number;

    public function initialize() {
        _currentIdx = 0;
        _names      = ["Raw", "10s Avg", "60s Avg", "Weighted"];
        _algorithms = [
            new RawAlgorithm(),
            new RollingAverage(10),
            new RollingAverage(60),
            new WeightedAverage(20)
        ];
    }

    public function next() as Void {
        _currentIdx = (_currentIdx + 1) % _algorithms.size();
    }

    public function previous() as Void {
        _currentIdx = (_currentIdx - 1 + _algorithms.size()) % _algorithms.size();
    }

    public function getName() as String { return _names[_currentIdx]; }

    public function compute(data as Array) as Number or Null {
        return _algorithms[_currentIdx].process(data);
    }
}
```

### Safe Type Conversion and Null Handling

```monkeyc
// Null-safe value with default
var hr = (info.currentHeartRate != null) ? info.currentHeartRate : 0;

// Number formatting
hr.format("%d")        // Integer, no leading zero
hr.format("%02d")      // 2-digit with leading zero
val.format("%.1f")     // One decimal place

// Multi-value formatted string
var text = Lang.format("HR: $1$ BPM ($2$)", [hr, algoName]);
```

### Efficient onUpdate() with Change Detection

```monkeyc
class MyField extends WatchUi.DataField {

    private var _displayHR   as Number;
    private var _needsRedraw as Boolean;

    public function initialize() {
        DataField.initialize();
        _displayHR   = 0;
        _needsRedraw = true;
    }

    public function compute(info as Activity.Info) as Void {
        var newHR = calculateHR(info);
        if (newHR != _displayHR) {
            _displayHR   = newHR;
            _needsRedraw = true;
        }
    }

    public function onUpdate(dc as Dc) as Void {
        if (!_needsRedraw) { return; }
        dc.setColor(Graphics.COLOR_WHITE, Graphics.COLOR_BLACK);
        dc.clear();
        dc.drawText(
            dc.getWidth() / 2, dc.getHeight() / 2,
            Graphics.FONT_NUMBER_MILD,
            _displayHR.format("%d"),
            Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER
        );
        _needsRedraw = false;
    }

    private function calculateHR(info as Activity.Info) as Number {
        if (info.currentHeartRate != null) { return info.currentHeartRate; }
        return 0;
    }
}
```

**Note on `_needsRedraw`:** Boolean member variables initialized to `false` will trigger "Statement is not reachable" warnings from the Monkey C static analyzer for any `if (_needsRedraw)` branch, because the analyzer traces from `initialize()` and sees the member is always `false`. Initializing to `true` (as above) avoids this. See `monkeyc_analyzer_unreachable_statement_guide.md` §3.

### Indoor Step/Distance (ActivityMonitor with Baseline)

`Activity.Info.elapsedDistance` is GPS-based and null for indoor activities. Use `ActivityMonitor` instead, capturing a baseline at `onTimerStart()` to isolate the current session from lifetime totals:

```monkeyc
// Member variables:
// _baselineSteps  as Number
// _baselineDistCm as Number

// In onTimerStart():
var am = ActivityMonitor.getInfo();
_baselineSteps  = (am.steps    != null) ? am.steps    : 0;
_baselineDistCm = (am.distance != null) ? am.distance : 0;

// In compute():
var am = ActivityMonitor.getInfo();
var currentSteps  = (am.steps    != null) ? am.steps    : _baselineSteps;
var currentDistCm = (am.distance != null) ? am.distance : _baselineDistCm;

var sessionSteps  = currentSteps  - _baselineSteps;
var sessionDistCm = currentDistCm - _baselineDistCm;
if (sessionSteps  < 0) { sessionSteps  = 0; }
if (sessionDistCm < 0) { sessionDistCm = 0; }

var distMiles = sessionDistCm.toFloat() / 160934.4f;   // cm per mile
```

See `consolidated_field_development_lessons.md` §7 for the full explanation.

---

## 7. Best Practices

**1. Guard optional APIs before use:**
```monkeyc
if ((Toybox has :SensorHistory) && (SensorHistory has :getHeartRateHistory)) {
    // Safe to call
}
```

**2. Calculate layout positions once in `onLayout()`, not every frame:**
```monkeyc
// GOOD — runs once per slot resize:
public function onLayout(dc as Dc) as Void {
    _centerX = dc.getWidth() / 2;
    _centerY = dc.getHeight() / 2;
}
// BAD — runs on every draw:
public function onUpdate(dc as Dc) as Void {
    var cx = dc.getWidth() / 2;   // Don't do this
}
```

**3. Null-check sensor data everywhere:**
```monkeyc
if (sample != null && sample.data != null) {
    var hr = sample.data;
}
```

**4. Integer math first — convert to float only when fractional precision is needed.**

**5. `fillPolygon` can cause watchdog timeout** when called in tight loops — use `fillRectangle` for graph fills. `fillPolygon` is fine for isolated triangles (arrows, indicators). See `skin_temp_widget_development_lessons.md` §15.

**6. Restrict rendering area with `dc.setClip()` / `dc.clearClip()`** to prevent content bleeding outside intended zones.

**7. Always verify layout with real data values**, not just "--" placeholders. The "--" string is much shorter than real values and will not reveal overlap or bezel clipping issues.

---

## 8. Code Organization

### Project File Structure

```
MyDataField/
├── manifest.xml
├── monkey.jungle
├── resources/
│   ├── drawables/
│   │   ├── drawables.xml      ← MUST be INSIDE drawables/ folder
│   │   └── launcher_icon.png
│   └── strings/
│       └── strings.xml
└── source/
    ├── MyDataFieldApp.mc
    └── MyDataFieldView.mc
```

### Class Member Order

```monkeyc
class MyView extends WatchUi.DataField {

    // 1. Constants
    private const WINDOW_SIZE = 60;

    // 2. Member variables (layout positions and data cache)
    private var _xCenter  as Number;
    private var _displayHR as Number;

    // 3. Constructor
    public function initialize() {
        DataField.initialize();
        _xCenter   = 0;
        _displayHR = 0;
    }

    // 4. Lifecycle methods in natural order
    public function onLayout(dc as Dc) as Void { }
    public function compute(info as Activity.Info) as Void { }
    public function onUpdate(dc as Dc) as Void { }
    public function onTimerStart()  as Void { }
    public function onTimerStop()   as Void { }
    public function onTimerPause()  as Void { }
    public function onTimerResume() as Void { }
    public function onTimerLap()    as Void { }
    public function onTimerReset()  as Void { }

    // 5. Private helpers at the bottom
    private function pickFont(dc as Dc, w as Number, h as Number) as FontDefinition {
        return Graphics.FONT_XTINY;  // Implement as needed
    }
}
```

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Full API reference, device specs, permission table
- `consolidated_field_development_lessons.md` — Indoor step/distance baseline pattern, `getObscurityFlags()`
- `venu3_practical_lessons.md` — Layout guidelines, bezel constraints, simulator workflow
- `monkeyc_analyzer_unreachable_statement_guide.md` — Static analyzer patterns

---

*Last updated: March 2026 — SDK 8.4.1*
