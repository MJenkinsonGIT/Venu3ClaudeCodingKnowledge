# Garmin Connect IQ SDK Knowledge Base
## For Venu 3 Development

**SDK Version:** 8.4.1 (Released 2026-02-03)
**API Level:** 5.2
**Target Device:** Garmin Venu® 3

---

## Numbered Index

1. [Device Specifications — Venu 3](#1-device-specifications--venu-3)
2. [Core Development Concepts](#2-core-development-concepts)
3. [Heart Rate Data Access](#3-heart-rate-data-access)
4. [Data Field Development](#4-data-field-development)
5. [Drawing UI — Graphics.Dc](#5-drawing-ui--graphicsdc)
6. [User Input and Menus](#6-user-input-and-menus)
7. [Manifest.xml Configuration](#7-manifestxml-configuration)
8. [Data Storage and Persistence](#8-data-storage-and-persistence)
9. [Time and Duration Handling](#9-time-and-duration-handling)
10. [Math Operations](#10-math-operations)
11. [Arrays and Dictionaries](#11-arrays-and-dictionaries)
12. [Common HR Smoothing Patterns](#12-common-hr-smoothing-patterns)
13. [Performance Considerations](#13-performance-considerations)
14. [Debugging and Testing](#14-debugging-and-testing)
15. [Build and Deployment](#15-build-and-deployment)
16. [Key SDK Resources](#16-key-sdk-resources)
17. [Common Gotchas](#17-common-gotchas)

---

## 1. Device Specifications — Venu 3

### Hardware Capabilities
- **Display:** 454×454 pixels, AMOLED, 16-bit color
- **Device Family:** round-454x454
- **Orientation:** Fixed (no rotation support)
- **Alpha Blending:** Supported
- **Enhanced Graphics:** Supported

### Memory Limits by App Type
- **Data Field:** 262,144 bytes (256 KB)
- **Watch App:** 786,432 bytes (768 KB)
- **Watch Face:** 131,072 bytes (128 KB)
- **Widget/Glance:** 65,536 bytes (64 KB)
- **Background:** 65,536 bytes (64 KB)
- **Audio Content Provider:** 524,288 bytes (512 KB)

### Sensors Available
- Heart Rate Monitor (optical)
- GPS
- Accelerometer / Gyroscope
- Compass (Magnetometer)
- Barometer / Altimeter
- Blood Oxygen (Pulse Ox)
- Temperature
- Body Battery / Stress Tracking

### Physical Buttons
The Venu 3 has exactly **3 physical buttons**: `enter`, `menu`, `esc`.

| Button | Location | SDK Constant | Reassignable? |
|--------|----------|-------------|---------------|
| ESC | Bottom-right | `KEY_ESC` | NO — OS captures for app exit |
| MENU | Top-right | `KEY_MENU` | YES — usable in `onKey()` |
| ENTER | Side | `KEY_ENTER` | Potentially usable |

Use `KEY_MENU` for custom actions. `KEY_ESC` fires before your app's delegate receives it.

---

## 2. Core Development Concepts

### Programming Language
**Monkey C** — Garmin's proprietary language
- Syntax similar to JavaScript/C
- Strongly typed with dynamic typing support
- Object-oriented with classes and inheritance
- Garbage collected
- **Modules do not support private access modifiers**
- **Local variables cannot have explicit type annotations**

### Project Types
1. **Data Field** — Displays during activities alongside native screens (256 KB)
2. **Watch Face** — Custom clock face (128 KB)
3. **Watch App** — Full standalone application (768 KB)
4. **Widget/Glance** — Quick information accessible from the watch face (64 KB)
5. **Background** — Runs in background with limited resources (64 KB)

### Toybox Namespace
All Connect IQ APIs are under the `Toybox` namespace. Key modules:
- `Toybox.Sensor` — Real-time sensor access
- `Toybox.SensorHistory` — Historical sensor data
- `Toybox.ActivityMonitor` — Activity tracking data
- `Toybox.Activity` — Current activity session
- `Toybox.ActivityRecording` — Record activities
- `Toybox.WatchUi` — User interface components
- `Toybox.Graphics` — Drawing and rendering
- `Toybox.System` — System information
- `Toybox.Time` — Time and duration handling
- `Toybox.Lang` — Core language types
- `Toybox.Math` — Mathematical operations
- `Toybox.Application` — App lifecycle and storage
- `Toybox.UserProfile` — User profile and HR zones

### Essential Imports
```monkeyc
import Toybox.WatchUi;
import Toybox.Graphics;
import Toybox.Sensor;
import Toybox.SensorHistory;
import Toybox.ActivityMonitor;
import Toybox.Activity;
import Toybox.System;
import Toybox.Time;
import Toybox.Math;
import Toybox.Lang;
import Toybox.Application;
import Toybox.UserProfile;
```

---

## 3. Heart Rate Data Access

### Real-Time Heart Rate: Toybox.Sensor

```monkeyc
Sensor.setEnabledSensors([Sensor.SENSOR_HEARTRATE]);
Sensor.enableSensorEvents(method(:onSensor));

function onSensor(sensorInfo) {
    var hr = sensorInfo.heartRate;  // Current HR in BPM (Number or null)
}
```

**Important:** `setEnabledSensors` must be called BEFORE `registerSensorDataListener`. The listener only configures data delivery — it does NOT power on the hardware.

### Historical Heart Rate: Toybox.SensorHistory

```monkeyc
// :period is NUMBER OF SAMPLES for HR history (NOT seconds!)
var hrIterator = SensorHistory.getHeartRateHistory({
    :period => 100,
    :order  => SensorHistory.ORDER_OLDEST_FIRST,
});

var sample = hrIterator.next();
while (sample != null) {
    var hr = sample.data;        // Heart rate value (null possible — check before use)
    var timestamp = sample.when; // Time.Moment
    sample = hrIterator.next();
}
```

**SensorHistory Methods:**
- `getHeartRateHistory(options)` — `:period` = sample count
- `getTemperatureHistory(options)` — `:period` = sample count
- `getBodyBatteryHistory(options)`
- `getElevationHistory(options)`
- `getOxygenSaturationHistory(options)`
- `getPressureHistory(options)`
- `getStressHistory(options)`

Use `sensorIter.getMin()` and `sensorIter.getMax()` for quick stats without iterating.

### ActivityMonitor Heart Rate

```monkeyc
var hrIterator = ActivityMonitor.getHeartRateHistory(null, false);
var sample = hrIterator.next();
while (sample != null) {
    if (sample.heartRate != ActivityMonitor.INVALID_HR_SAMPLE) {
        var hr = sample.heartRate;
    }
    sample = hrIterator.next();
}
// ActivityMonitor.INVALID_HR_SAMPLE = 255
```

### ActivityMonitor.Info (Step/Activity Data)

```monkeyc
var info = ActivityMonitor.getInfo();  // Non-nullable — do NOT null-check return value
var steps    = info.steps;             // nullable
var calories = info.calories;          // nullable
var distance = info.distance;          // in cm, nullable
var goal     = info.stepGoal;          // nullable
var activeMinutes = info.activeMinutesDay;
```

`ActivityMonitor.getInfo()` returns `ActivityMonitor.Info` (not nullable). The **fields** inside it are nullable.

### HR Raw Data Limitation
**There is NO raw HR data.** All APIs return smoothed BPM values. External ANT+ chest straps provide faster response but still only give BPM through the SDK. Beat-to-beat intervals (HRV) require an active `ActivityRecording.Session` — see `hrv_logger_development_lessons.md`.

---

## 4. Data Field Development

### Why Data Field?
- Shows during activities alongside native screens
- Low memory overhead (256 KB)
- Can access real-time and historical sensor data
- User can view while exercising

### Venu 3 2-App Limit
The Venu 3 enforces a hard limit of **2 concurrent Connect IQ app instances** during an activity. Each data field app counts as one instance regardless of how many screen slots it occupies. Design fields to display multiple metrics in one app.

### Basic Data Field Structure

```monkeyc
class MyDataField extends WatchUi.DataField {

    public function initialize() {
        DataField.initialize();
    }

    public function onLayout(dc as Dc) as Void {
        // Calculate positions once — store as member variables
        // Called when slot dimensions are known
    }

    public function compute(info as Activity.Info) as Void {
        // Called periodically — do data processing here, NOT in onUpdate()
    }

    public function onUpdate(dc as Dc) as Void {
        // Called to render — drawing only, use cached values from compute()
    }

    // Timer lifecycle
    public function onTimerStart() as Void { }
    public function onTimerStop() as Void { }
    public function onTimerPause() as Void { }
    public function onTimerResume() as Void { }
    public function onTimerLap() as Void { }
    public function onTimerReset() as Void { }
}
```

### SimpleDataField vs DataField

| Feature | SimpleDataField | DataField |
|---------|----------------|-----------|
| Base Class | `WatchUi.SimpleDataField` | `WatchUi.DataField` |
| Rendering | Garmin handles | Custom `onUpdate()` |
| Full-Screen | No automatic full screen | Yes — automatic dual view |
| Multiple Values | No — single value only | Yes |
| Complexity | Very simple | More code, full control |

When a `WatchUi.DataField` with custom `onUpdate()` is added to an activity, Garmin **automatically creates BOTH** a field slot view and a full-screen page. This is automatic Garmin behavior, not programmed.

### Activity.Info Reference

```monkeyc
info.currentHeartRate      // Current HR from activity
info.averageHeartRate      // Session average HR
info.maxHeartRate          // Session max HR
info.currentSpeed          // Current speed
info.averageSpeed          // Average speed
info.elapsedDistance       // GPS distance (null indoors — use ActivityMonitor for indoor)
info.elapsedTime           // Elapsed time (milliseconds)
info.timerTime             // Timer time (milliseconds)
info.timerState            // Activity.TimerState or Null
// Activity.TIMER_STATE_OFF / STOPPED / PAUSED / ON
```

**Indoor Activities:** `info.elapsedDistance` is GPS-based and null indoors. Use `ActivityMonitor.getInfo().distance` (accelerometer-based, works indoors) and subtract a baseline captured at `onTimerStart()`.

---

## 5. Drawing UI — Graphics.Dc

### Basic Drawing Methods

```monkeyc
dc.setColor(Graphics.COLOR_WHITE, Graphics.COLOR_BLACK);
dc.clear();
dc.drawText(x, y, font, "Hello", Graphics.TEXT_JUSTIFY_CENTER);
dc.drawRectangle(x, y, width, height);
dc.fillRectangle(x, y, width, height);
dc.drawCircle(x, y, radius);
dc.fillCircle(x, y, radius);
dc.drawLine(x1, y1, x2, y2);
dc.fillPolygon([[x1,y1],[x2,y2],[x3,y3]]);  // Array<Point2D>
dc.setClip(x, y, width, height);   // Restrict rendering area
dc.clearClip();                     // Remove clip restriction
```

**Constructing arbitrary gray colors:**
```monkeyc
var brightness = 91;  // 0-255
var col = (brightness << 16) | (brightness << 8) | brightness;
dc.setColor(col, Graphics.COLOR_TRANSPARENT);
```

### Color Constants
```
Graphics.COLOR_BLACK, COLOR_WHITE, COLOR_RED, COLOR_ORANGE,
COLOR_YELLOW, COLOR_GREEN, COLOR_BLUE, COLOR_PURPLE,
COLOR_PINK, COLOR_DK_GRAY, COLOR_LT_GRAY, COLOR_TRANSPARENT
```

### Font Constants
```
FONT_XTINY (~13px), FONT_TINY (~16px), FONT_SMALL (~22px),
FONT_MEDIUM (~28px), FONT_LARGE (~36px),
FONT_NUMBER_MILD (~45px), FONT_NUMBER_MEDIUM (~60px),
FONT_NUMBER_HOT, FONT_NUMBER_THAI_HOT
```

Heights are empirical (measured on Venu 3) — SDK does not publish glyph heights.

### Text Justification
```
Graphics.TEXT_JUSTIFY_LEFT, TEXT_JUSTIFY_CENTER, TEXT_JUSTIFY_RIGHT, TEXT_JUSTIFY_VCENTER
```

### Negative Y Coordinates
Setting `y` to a small negative value strips the font's ascent padding (blank space above glyphs). `y = 0` is NOT the same as "text touching the top edge." For `FONT_NUMBER_MILD`, approximately `y = -9%` of slot height pushes digits flush to the top.

### Theme Adaptation
```monkeyc
var bgColor = getBackgroundColor();  // White or Black depending on user theme
var fgColor = (bgColor == Graphics.COLOR_WHITE) ? Graphics.COLOR_BLACK : Graphics.COLOR_WHITE;
```

### Bezel Geometry
At x = 18% of 454px = 82px from edge. For that column to be safe at height y: `√(227² − y_from_centre²) > 145`. Content below y ≈ 410 risks bezel clipping. Tested safe zone: y = 70 to y = 350 at full width.

---

## 6. User Input and Menus

### Input Delegate

```monkeyc
class MyInputDelegate extends WatchUi.BehaviorDelegate {
    public function initialize() { BehaviorDelegate.initialize(); }

    public function onSelect() as Boolean { return true; }   // SELECT/tap
    public function onBack() as Boolean { return false; }    // BACK/ESC

    // SWIPE UP fires onNextPage(); SWIPE DOWN fires onPreviousPage()
    // (This is counterintuitive but confirmed on device — see customhrv_development_lessons.md)
    public function onNextPage() as Boolean { return true; }
    public function onPreviousPage() as Boolean { return true; }

    public function onKey(evt as KeyEvent) as Boolean {
        if (evt.getKey() == WatchUi.KEY_MENU) {
            // Custom action
            return true;
        }
        return false;
    }

    // Touchscreen
    public function onTap(evt as ClickEvent) as Boolean {
        var coords = evt.getCoordinates();  // [x, y]
        return true;
    }
}
```

**Swipe direction on Venu 3 (confirmed by device video):**
- Swipe UP → `onNextPage()` fires
- Swipe DOWN → `onPreviousPage()` fires
- Do NOT use `onSwipe()` for primary navigation on Venu 3 — unreliable.

**Touchscreen:** `onSelect()` fires only for physical SELECT button press, NOT for screen taps. Use `onTap()` for tap handling.

---

## 7. Manifest.xml Configuration

### Complete Data Field Manifest

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
        <!-- NO fit-contrib-activities for data fields — they work in all activities -->
    </iq:application>
</iq:manifest>
```

### Permission Reference

| API | Permission Required? | Declaration |
|-----|---------------------|-------------|
| ActivityMonitor | No — core API | (none) |
| System, Graphics, WatchUi, Lang, Math | No | (none) |
| Application.Storage | No — available to all types | (none) — do NOT add `id="Storage"` |
| Sensor | Yes | `<iq:uses-permission id="Sensor"/>` |
| SensorHistory | Yes | `<iq:uses-permission id="SensorHistory"/>` |
| Positioning (GPS) | Yes | `<iq:uses-permission id="Positioning"/>` |
| Communications | Yes | `<iq:uses-permission id="Communications"/>` |
| UserProfile | Yes | `<iq:uses-permission id="UserProfile"/>` |
| ActivityRecording | Yes | `<iq:uses-permission id="Fit"/>` — confirmed working; `FitRecording` is also a valid ID but untested |

**Valid permission IDs:** Ant, Background, BluetoothLowEnergy, Communications, ComplicationProvider, ComplicationSubscriber, DataFieldAlert, Fit, FitRecording, PersistedContent, Positioning, Sensor, SensorHistory, SensorLogging, UserProfile. Adding an invalid ID causes a build error.

### App Name Restrictions
Ampersand (`&`) is prohibited in app names even as `&amp;`. Use "and" instead. App display name comes from `resources/strings/strings.xml`, NOT from the `.prg` filename.

### fit-contrib-activities
Do NOT include `<iq:fit-contrib-activities>` in data field manifests — it causes a build error. Data fields work in all activities automatically.

---

## 8. Data Storage and Persistence

### Application.Storage

```monkeyc
import Toybox.Application.Storage;

Storage.setValue("key", value);       // Store (Number, String, Boolean, Array, Dictionary, null)
var value = Storage.getValue("key");  // Retrieve (null if key doesn't exist)
Storage.deleteValue("key");           // Delete
```

Persists between app runs. No manifest permission required for any app type.

### Properties (User-Configurable Settings)

```monkeyc
// Define in resources/properties.xml:
// <property id="smoothingWindow" type="number">10</property>

var app = Application.getApp();
var value = app.getProperty("smoothingWindow");
app.setProperty("smoothingWindow", 20);
```

---

## 9. Time and Duration Handling

```monkeyc
var now = Time.now();            // Time.Moment (current time)
var unix = now.value();          // Unix timestamp (seconds since epoch)

var duration = new Time.Duration(60);   // 60 seconds
var later = now.add(duration);          // New Moment
var diff = now.subtract(earlier);       // Returns Duration

// Clock time
var clock = System.getClockTime();
var hour = clock.hour;    // 0-23
var min  = clock.min;     // 0-59
var sec  = clock.sec;     // 0-59

// Device settings
var settings = System.getDeviceSettings();
var is24hr = settings.is24Hour;
```

**Timezone bug:** `Gregorian.moment()` uses UTC. Never pass local time fields from `Gregorian.info()` to it. Use `System.getClockTime()` subtraction for local midnight.

---

## 10. Math Operations

```monkeyc
import Toybox.Math;
Math.floor(x), Math.ceil(x), Math.round(x), Math.abs(x)
Math.pow(x, y), Math.sqrt(x)
Math.sin(x), Math.cos(x), Math.tan(x)   // radians
Math.asin(x), Math.acos(x), Math.atan(x), Math.atan2(y, x)
Math.ln(x), Math.log(x, b)
Math.PI
```

Prefer integer math for performance. Use `.toFloat()` only when fractional precision is needed.

---

## 11. Arrays and Dictionaries

```monkeyc
// Arrays — explicit type casting required for typed arrays
var array = [] as Array<Number>;
array.add(6);
array.remove(0);    // Remove by value
var len = array.size();

// Dictionaries
var dict = { "key1" => "value1", :symbolKey => 42 };
var v = dict["key1"];
dict.hasKey("key1");
dict.keys();         // Returns array of keys
dict.remove("key1");
```

**Type casting on indexed access:** Cast expressions are not lvalues — you cannot assign through a cast. Extract to a local variable first:

```monkeyc
// ILLEGAL:
(flags[i] as Dictionary)[:key] = true;

// CORRECT:
var entry = flags[i] as Dictionary;
entry[:key] = true;
```

---

## 12. Common HR Smoothing Patterns

### Circular Buffer

```monkeyc
class CircularBuffer {
    private var _buffer;
    private var _size;
    private var _writeIndex;
    private var _count;

    public function initialize(size) {
        _size = size;
        _buffer = new [size] as Array<Number or Null>;
        _writeIndex = 0;
        _count = 0;
    }

    public function add(val) {
        _buffer[_writeIndex] = val;
        _writeIndex = (_writeIndex + 1) % _size;
        if (_count < _size) { _count++; }
    }

    public function getAverage() {
        if (_count == 0) { return null; }
        var sum = 0;
        for (var i = 0; i < _count; i++) { sum += _buffer[i]; }
        return sum / _count;
    }
}
```

---

## 13. Performance Considerations

- Do all heavy computation in `compute()`, only rendering in `onUpdate()`
- Calculate positions once in `onLayout()`, store as member variables — never recalculate in `onUpdate()`
- Cache values — redraw only when values change
- Integer math is faster than float
- Avoid `fillPolygon` in high-frequency loops on device (watchdog timeout risk) — use `fillRectangle` instead
- `const` for large lookup tables to reduce memory pressure
- Avoid creating large objects in `onUpdate()`
- Disable sensors when not needed

---

## 14. Debugging and Testing

```monkeyc
System.println("Debug: " + value);   // Appears in simulator console and device logs
```

```monkeyc
try {
    riskyOperation();
} catch (exception) {
    System.println("Error: " + exception.getErrorMessage());
}
```

**Capability checking (always check before using optional APIs):**
```monkeyc
if ((Toybox has :SensorHistory) && (SensorHistory has :getHeartRateHistory)) {
    // Safe to use
}
```

---

## 15. Build and Deployment

1. Open a `.mc` file → press **F5** to build and launch simulator
2. Press **Ctrl+F5** to build `.prg` for device
3. Connect watch via USB → copy `.prg` to `GARMIN\APPS\`
4. Press **Back button on watch** to trigger "Verifying Apps"
5. Unplug after watch disconnects

**Uninstalling sideloaded apps:** Use Garmin Express — there is no other method.

**Data screen configuration:** Only accessible from the watch during an active activity session via long-press → Data Screens. Cannot be done from the main settings menu or the phone app for sideloaded apps.

**Simulator configuration:** Create `.vscode/launch.json` specifying the device to simulate.

---

## 16. Key SDK Resources

- API Documentation: `doc/Toybox/` (HTML files)
- Sample Projects: `samples/` directory
- Device Specs: `Devices/venu3/compiler.json`

**Important Samples:**
- `SimpleDataField` — Basic data field template
- `SensorHistory` — Historical sensor patterns (note: `:period` for HR = sample count, not seconds)
- `MoxyField` — Advanced adaptive layout data field

---

## 17. Common Gotchas

1. **Null checks on sensor data** — always check before using
2. **ActivityMonitor.INVALID_HR_SAMPLE = 255** — filter this value
3. **`ActivityMonitor.getInfo()` is non-nullable** — do NOT null-check the return; the *fields* inside are nullable
4. **`Activity.Info.elapsedDistance` is GPS-only** — null indoors; use `ActivityMonitor` for indoor steps/distance
5. **Power cycles clear SensorHistory** — data only goes back to last power cycle
6. **Permissions must be declared** — missing permission throws at runtime, not compile time
7. **`fit-contrib-activities` causes build error** in data field manifests — remove it
8. **`drawables.xml` must be INSIDE the `drawables/` folder** — not beside it
9. **App display name comes from `strings.xml`** — the `.prg` filename is irrelevant
10. **Swipe UP = `onNextPage()`, Swipe DOWN = `onPreviousPage()`** on Venu 3 — counterintuitive
11. **`onSelect()` does not fire for screen taps** — use `onTap()` on touchscreen devices
12. **Monkey C static analyzer traces from `initialize()`** — branches on member variables initialized to a fixed value may be flagged as unreachable; see `monkeyc_analyzer_unreachable_statement_guide.md`

---

**End of Knowledge Base**

*Last updated: February 2026 — SDK 8.4.1, API Level 5.2, Garmin Venu 3*
