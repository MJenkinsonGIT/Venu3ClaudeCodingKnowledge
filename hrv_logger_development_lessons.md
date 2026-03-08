# HRV Raw Logger — Development Lessons
## Real-World Discoveries — February 2026

**Project:** HRVRawLogger Widget for Garmin Venu 3
**SDK Version:** 8.4.1 / API Level 5.2
**App Type:** Widget

---

## Numbered Index

1. [setEnabledSensors Required Before registerSensorDataListener](#1-setenabledsensors-required-before-registersensordatalistener)
2. [Backlight Kills Sensor Callbacks on Venu 3](#2-backlight-kills-sensor-callbacks-on-venu-3)
3. [Venu 3 Button Mapping](#3-venu-3-button-mapping)
4. [heartBeatIntervals Warm-Up Period](#4-heartbeatintervals-warm-up-period)
5. [heartBeatIntervals Requires Active ActivityRecording Session](#5-heartbeatintervals-requires-active-activityrecording-session)
6. [Zero Values Indicate Signal Loss](#6-zero-values-indicate-signal-loss)
7. [Architecture Notes](#7-architecture-notes)

---

## 1. setEnabledSensors Required Before registerSensorDataListener

### The Bug
Using `registerSensorDataListener` with `:heartBeatIntervals => { :enabled => true }` — every callback fires correctly on schedule but `heartBeatIntervals` always returns `[]` (empty array).

### Root Cause
`registerSensorDataListener` only configures the *data format and delivery schedule*. It does **not** power on the underlying optical HR hardware. The sensor hardware must be explicitly activated first with `setEnabledSensors`.

### Fix
```monkeyc
// CORRECT — enable hardware FIRST, then register listener
Sensor.setEnabledSensors([Sensor.SENSOR_HEARTRATE]);
Sensor.registerSensorDataListener(
    method(:onSensorData),
    {
        :period             => 4,
        :heartBeatIntervals => { :enabled => true }
    }
);

// On cleanup — disable hardware when done
Sensor.unregisterSensorDataListener();
Sensor.setEnabledSensors([]);
```

### Why This Is Easy To Miss
SDK examples show both `setEnabledSensors` AND `registerSensorDataListener` used together, but it's easy to assume the listener registration alone is sufficient. It is not.

---

## 2. Backlight Kills Sensor Callbacks on Venu 3

### The Bug
Calling `Attention.backlight(true)` repeatedly (e.g., every 4 seconds) caused `heartBeatIntervals` to return `[]` after the first callback — even when the first callback was successful.

### Root Cause
On gesture-enabled displays (like the Venu 3 AMOLED), the SDK documentation notes that calling `Attention.backlight()` disables gesture detection for the duration that the backlight is on. The Venu 3's optical HR sensor appears to share or depend on the same subsystem.

### Fix
Do not use `Attention.backlight()` in any app that relies on HR sensor data. There is no reliable software way to prevent screen timeout without interfering with sensors on the Venu 3.

---

## 3. Venu 3 Button Mapping

From the SDK device reference (`venu3.html`), the Venu 3 has exactly **3 physical buttons**: `enter`, `menu`, `esc`.

| Physical Button | Location | SDK Constant | Reassignable? |
|---|---|---|---|
| ESC | Bottom-right | `KEY_ESC` | NO — OS captures for app exit |
| MENU | Top-right | `KEY_MENU` | YES — usable in `onKey()` handler |
| ENTER | Side | `KEY_ENTER` | Potentially usable |

Use `KEY_MENU` for custom button actions in widgets. `KEY_ESC` fires before the app's delegate receives it — you cannot intercept it.

```monkeyc
public function onKey(keyEvent as WatchUi.KeyEvent) as Boolean {
    if (keyEvent.getKey() == WatchUi.KEY_MENU) {
        // custom action
        return true;   // consume the event
    }
    return false;      // pass through
}
```

---

## 4. heartBeatIntervals Warm-Up Period

Even with correct sensor setup, the first 1–2 callbacks after startup will typically return `[]` while the optical HR sensor acquires a confident signal lock. This is normal hardware behavior, not a bug. After lock is acquired, callbacks return arrays like `[820, 795, 810, 834]`.

---

## 5. heartBeatIntervals Requires Active ActivityRecording Session

### The Bug
`heartBeatIntervals` returns `[]` on every callback even though `setEnabledSensors` succeeds, `registerSensorDataListener` registers without error, callbacks fire on schedule, and `heartRateData` is non-null.

### Root Cause
The SDK's own `unregisterSensorDataListener` example shows:
```
Sensor.unregisterSensorDataListener();
mSession.stop();
```
And `registerSensorDataListener` has "See Also: `ActivityRecording.createSession()`".

**Beat-to-beat interval data only flows when an `ActivityRecording.Session` is active.** This is a firmware-level requirement — the Venu 3 only runs its optical HR sensor in high-frequency beat-to-beat mode when the FIT recording subsystem is engaged.

### Evidence
- First callback of a session occasionally gets residual data from a recently-ended native activity/Health Snapshot on the watch — this explains the "one readout at the very beginning" phenomenon
- All subsequent callbacks return `[]` once that native session state clears

### Fix

```monkeyc
// In startSensor():
_session = ActivityRecording.createSession({
    :name  => "HRV",
    :sport => Activity.SPORT_GENERIC
});
_session.start();
Sensor.setEnabledSensors([Sensor.SENSOR_ONBOARD_HEARTRATE]);
Sensor.registerSensorDataListener(
    method(:onSensorData),
    { :period => 1, :heartBeatIntervals => { :enabled => true } }
);

// In stopSensor():
Sensor.unregisterSensorDataListener();
Sensor.setEnabledSensors([]);
_session.stop();
_session.discard();  // Prevents phantom workouts in Garmin Connect
```

### Required manifest.xml Permission

```xml
<iq:uses-permission id="FitRecording"/>
```

---

## 6. Zero Values Indicate Signal Loss

When the optical sensor loses skin contact or enters low-power state mid-session, it returns `[0]` rather than an empty array. Filter these out:

```monkeyc
var raw = sensorData.heartRateData.heartBeatIntervals;
if (raw != null && raw.size() > 0) {
    var filtered = [] as Array<Number>;
    for (var i = 0; i < raw.size(); i++) {
        if (raw[i] > 0) { filtered.add(raw[i]); }
    }
    if (filtered.size() > 0) { intervals = filtered; }
}
```

Logging the raw `[]` entries (without filtering) is still valuable as it marks exactly when the sensor dropped signal.

---

## 7. Architecture Notes

### App Type: Widget
Widget is required (not Glance, not Data Field) because:
- Widgets can use `registerSensorDataListener`
- Widgets have `onShow`/`onHide` lifecycle for sensor start/stop
- Widgets run as full-screen apps with persistent sensor registration while visible

### Sensor Lifecycle Pattern

```
onShow()  →  setEnabledSensors + registerSensorDataListener
onHide()  →  unregisterSensorDataListener + setEnabledSensors([])
```

This prevents the sensor from running in the background when the widget is not visible, which would drain battery and potentially conflict with other apps.

---

---

## Related Knowledge Base Documents

- `garmin_connectiq_knowledge_base.md` — Sensor APIs, `setEnabledSensors` pattern, permission table (`FitRecording`)
- `customhrv_development_lessons.md` — Full HRV watch app built on top of these lessons
- `skin_temp_widget_development_lessons.md` — Widget architecture, `onShow`/`onHide` sensor lifecycle pattern
- `garmin_development_addendum.md` — Installation workflow, HR data limitations overview

---

*Last updated: February 2026 — SDK 8.4.1*
