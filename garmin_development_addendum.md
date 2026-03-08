# Garmin Connect IQ Development — Practical Lessons Addendum
## Real-World Solutions from Venu 3 Data Field Development

**Date:** February 17, 2026
**SDK Version:** 8.4.1
**Device:** Venu 3
**Project:** Interval HR Field (initial project)

This addendum covers practical solutions discovered during real development that are not clearly documented in the official SDK. Developer key details are summarized here for workflow reference; the full guide is in `garmin_developer_key_guide.md`.

---

## Numbered Index

1. [Developer Key — Quick Reference](#1-developer-key--quick-reference)
2. [Manifest.xml for Data Fields](#2-manifestxml-for-data-fields)
3. [Project Structure Requirements](#3-project-structure-requirements)
4. [Array Initialization Type Safety](#4-array-initialization-type-safety)
5. [Simulator Configuration](#5-simulator-configuration)
6. [Installing Apps on Physical Devices](#6-installing-apps-on-physical-devices)
7. [Uninstalling Sideloaded Data Fields](#7-uninstalling-sideloaded-data-fields)
8. [Heart Rate Data Limitations](#8-heart-rate-data-limitations)
9. [Complete Development Workflow](#9-complete-development-workflow)
10. [Key Takeaways Summary](#10-key-takeaways-summary)

---

## 1. Developer Key — Quick Reference

Garmin requires DER-format RSA keys to sign apps for physical device installation. See `garmin_developer_key_guide.md` for the complete guide. Quick reference:

```powershell
cd C:\Users\[YourName]\AppData\Roaming\Garmin\ConnectIQ
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" genrsa -out developer_key.pem 4096
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs8 -topk8 -inform PEM -outform DER -in developer_key.pem -out developer_key.der -nocrypt
```

VS Code `settings.json` (use double backslashes):
```json
{
    "monkeyC.developerKeyPath": "C:\\Users\\[YourName]\\AppData\\Roaming\\Garmin\\ConnectIQ\\developer_key.der"
}
```

After changing settings: reload VS Code (Ctrl+Shift+P → `Developer: Reload Window`).

---

## 2. Manifest.xml for Data Fields

### Critical Issue: fit-contrib-activities Causes Build Error

**WRONG (build error: "Invalid content starting with element 'fit-contrib-activities'"):**
```xml
<iq:manifest>
    <iq:application type="datafield">
        <iq:fit-contrib-activities>
            <iq:fit-contrib-activity>run</iq:fit-contrib-activity>
        </iq:fit-contrib-activities>
    </iq:application>
</iq:manifest>
```

**CORRECT — omit fit-contrib-activities entirely:**
```xml
<?xml version="1.0"?>
<iq:manifest version="3" xmlns:iq="http://www.garmin.com/xml/connectiq">
    <iq:application
        entry="MyDataFieldApp"
        id="A5B3C2D1E4F5A6B7C8D9E0F1A2B3C4D5"
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
    </iq:application>
</iq:manifest>
```

Data fields work in **all activities** automatically. Users configure which fields appear through watch settings. The `fit-contrib-activities` element is only for watch apps and widgets.

---

## 3. Project Structure Requirements

### Correct Folder Hierarchy

```
MyDataField/
├── manifest.xml
├── monkey.jungle
├── bin/                           (created by build)
│   └── *.prg
├── source/
│   ├── MyDataFieldApp.mc
│   └── MyDataFieldView.mc
└── resources/
    ├── drawables/
    │   ├── drawables.xml          ← MUST be INSIDE drawables/ folder, not beside it
    │   └── launcher_icon.png
    └── strings/
        └── strings.xml
```

### Critical: drawables.xml Location
`drawables.xml` **must** be inside the `drawables/` folder. Placing it beside the folder (at the `resources/` level) causes a build error.

### monkey.jungle (Minimal Working Example)
```
project.manifest = manifest.xml
```

### App Name Restrictions
- Ampersand (`&`) is prohibited, even escaped as `&amp;`. Use "and" instead.
- App display name comes from `resources/strings/strings.xml`, NOT from the `.prg` filename.

```xml
<!-- strings.xml -->
<string id="AppName">Steps and Time</string>
```

---

## 4. Array Initialization Type Safety

Arrays require explicit type casting in typed Monkey C:

```monkeyc
// WRONG — may cause type error:
var samples = new [60];

// CORRECT — explicit type:
var samples = [] as Array<Number>;
// or:
var samples = new [60] as Array<Number or Null>;
```

---

## 5. Simulator Configuration

### launch.json Required

The simulator needs to know which device to simulate. Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Run - venu3",
            "request": "launch",
            "type": "monkeyc",
            "device": "venu3"
        }
    ]
}
```

Without this file, the simulator may show a blank screen or fail to launch.

### Testing Data Fields in Simulator

1. Press **F5** with a `.mc` file open
2. In the simulator: Simulation → Activity Data → start a simulated activity
3. Navigate: Simulation → Data Fields → select your field for a slot

---

## 6. Installing Apps on Physical Devices

1. Build: press **Ctrl+F5** (or F5) to compile
2. Connect Venu 3 via USB
3. Copy `.prg` from `bin/` to `GARMIN\APPS\` on the watch
4. Press the **Back button on the watch** — watch shows "Verifying Apps"
5. Unplug after the watch finishes and disconnects
6. Add field to activity (if first time): start activity → long-press lower button → Data Screens → Edit

**New `.prg` files with the same app ID automatically replace the old version.** No manual removal needed during development iterations.

**Changing an app ID** causes the watch to treat it as a brand-new app — the old version stays installed and must be removed via Garmin Express.

---

## 7. Uninstalling Sideloaded Data Fields

**Only method that works: Garmin Express on a computer.**

There is no way to uninstall sideloaded apps directly from the watch or from the Garmin Connect phone app. Connect watch via USB → open Garmin Express → IQ Apps → Uninstall.

---

## 8. Heart Rate Data Limitations

- **No raw sensor data exists.** All SDK HR APIs return smoothed BPM values.
- Garmin applies smoothing algorithms at the firmware level. The SDK has no access to raw optical sensor output.
- External ANT+ chest straps provide faster response but still only deliver BPM through the SDK.
- **Beat-to-beat intervals (HRV)** require `registerSensorDataListener` with `heartBeatIntervals` enabled AND an active `ActivityRecording.Session`. See `hrv_logger_development_lessons.md` for full details.

---

## 9. Complete Development Workflow

### First-Time Setup

1. Install VS Code + Monkey C extension
2. Install SDK (download from Garmin developer portal)
3. Generate developer key (see `garmin_developer_key_guide.md`)
4. Configure VS Code settings with SDK path and key path
5. Reload VS Code

### Iterative Development Cycle

```
Edit .mc files → Save (Ctrl+S)
    ↓
Test in simulator (F5)
    ↓
Build for device (Ctrl+F5)
    ↓
Copy .prg to GARMIN\APPS\
    ↓
Press Back button on watch
    ↓
Test on device during actual activity
    ↓
Note issues → Return to top
```

### Troubleshooting Quick Reference

| Problem | Solution |
|---------|----------|
| "Unable to load private key" | Key not DER format; path wrong; reload VS Code |
| "Invalid content starting with fit-contrib-activities" | Remove from manifest — data fields don't use it |
| Simulator shows blank screen | Create `.vscode/launch.json` with device specified |
| Can't find field on watch | Must add via on-watch Data Screens config during activity |
| Can't uninstall field | Use Garmin Express |
| HR data lags | Normal — all APIs return smoothed values |

---

## 10. Key Takeaways Summary

1. **Developer keys MUST be DER format** — PEM won't work
2. **Data field manifests do NOT include fit-contrib-activities** — causes build error
3. **drawables.xml MUST be inside the drawables/ folder** — not beside it
4. **Arrays need explicit type casting** — `[] as Array<Number>`
5. **Simulator needs launch.json** — specifies which device
6. **Use Back button to install** — faster than restarting watch
7. **Garmin Express is the only way to uninstall** sideloaded apps
8. **There is NO raw HR data** — all APIs return smoothed BPM
9. **App name comes from strings.xml** — not from the .prg filename
10. **Changing app ID creates a new separate app** on the watch

---

---

## Related Knowledge Base Documents

- `garmin_developer_key_guide.md` — Complete developer key generation guide (DER format, two-step method)
- `garmin_connectiq_knowledge_base.md` — Full permission reference table, manifest configuration, API reference
- `connectiq_code_patterns.md` — Code patterns, file structure, manifest examples
- `venu3_practical_lessons.md` — On-watch data screen configuration, sideloading behaviour
- `monkeyc_analyzer_unreachable_statement_guide.md` — Compiler warnings and static analyzer

---

*Last updated: February 2026 — SDK 8.4.1*
