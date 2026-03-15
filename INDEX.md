# Venu 3 Connect IQ Knowledge Base — Master Index

**SDK Version:** 8.4.1 / API Level 5.2
**Target Device:** Garmin Venu 3 (454×454 AMOLED, round)
**Last Updated:** February 2026

This index catalogs all knowledge base files in this repository. Each entry lists the file's purpose, the numbered sections it contains, and the key lessons it captures. Files are grouped by type: **Foundation** documents establish core knowledge, **Project** documents capture lessons from specific apps.

---

## Quick Reference — Where to Look for What

| If you need... | Go to... |
|----------------|----------|
| Device specs, API overview, permission table | `garmin_connectiq_knowledge_base.md` |
| Developer key generation (DER format) | `garmin_developer_key_guide.md` |
| Manifest, project structure, install workflow | `garmin_development_addendum.md` |
| Copy-paste code patterns and templates | `connectiq_code_patterns.md` |
| "Statement is not reachable" compiler warnings | `monkeyc_analyzer_unreachable_statement_guide.md` |
| On-watch data screen configuration | `venu3_practical_lessons.md` |
| App list placement (Activities vs Apps vs Glances) | `customhrv_development_lessons.md` §11 |
| Indoor steps/distance (ActivityMonitor baseline) | `consolidated_field_development_lessons.md` §7, §10 |
| HR zone coloring (UserProfile API) | `timerhr_field_development_lessons.md` §1 |
| DST-safe "today" boundary for activity history | `timerhr_field_development_lessons.md` §6 |
| `ClockTime.dst` always 0 on real devices | `timerhr_field_development_lessons.md` §6 |
| Swipe/tap input on Venu 3 touchscreen | `customhrv_development_lessons.md` §1–2 |
| Beat-to-beat HRV intervals (ActivityRecording) | `hrv_logger_development_lessons.md` §5 |
| Glance view styling, AMOLED color calibration | `skin_temp_widget_development_lessons.md` §2–5 |
| `(:glance)` annotation on watch-app with glance view | `skin_temp_widget_development_lessons.md` §21 |
| Persistent storage across sessions | `skin_temp_widget_development_lessons.md` §19–20 |
| fillPolygon vs fillRectangle performance | `skin_temp_widget_development_lessons.md` §15 |
| Bezel-safe positioning values (tested) | `consolidated_field_development_lessons.md` §9 |
| Negative y-coordinates / font ascent padding | `timerhr_field_development_lessons.md` §2 |

---

## Foundation Documents

### 1. `garmin_connectiq_knowledge_base.md`
**Purpose:** Core SDK reference — the primary starting point for all development.

| § | Section |
|---|---------|
| 1 | Device Specifications — Venu 3 (display, memory limits by app type, sensors, buttons) |
| 2 | Core Development Concepts (Monkey C, project types, Toybox namespace, essential imports) |
| 3 | Heart Rate Data Access (Sensor, SensorHistory, ActivityMonitor, HRV limitation) |
| 4 | Data Field Development (structure, 2-app limit, SimpleDataField vs DataField, Activity.Info) |
| 5 | Drawing UI — Graphics.Dc (methods, colors, fonts, text justification, negative y, theme adaptation) |
| 6 | User Input and Menus (BehaviorDelegate, swipe direction, onTap vs onSelect) |
| 7 | Manifest.xml Configuration (complete template, full permission reference table) |
| 8 | Data Storage and Persistence (Application.Storage, Properties) |
| 9 | Time and Duration Handling (Time.now, System.getClockTime, timezone warning) |
| 10 | Math Operations |
| 11 | Arrays and Dictionaries (type casting, cast-as-lvalue error) |
| 12 | Common HR Smoothing Patterns (CircularBuffer implementation) |
| 13 | Performance Considerations (compute vs onUpdate, onLayout caching, fillPolygon warning) |
| 14 | Debugging and Testing |
| 15 | Build and Deployment |
| 16 | Key SDK Resources |
| 17 | Common Gotchas (12 quick-reference pitfalls) |

**Key facts:** Memory limits (Data Field 256 KB, Watch App 768 KB, Widget 64 KB). Valid permission IDs listed. `onSelect()` ≠ tap on touchscreen. Swipe UP = `onNextPage()`.

---

### 2. `garmin_developer_key_guide.md`
**Purpose:** Complete guide to generating and configuring the DER-format RSA developer key required to install apps on physical devices.

| § | Section |
|---|---------|
| 1 | What Is a Developer Key? |
| 2 | Key Requirements (DER format, 4096-bit RSA) |
| 3 | How to Generate a Developer Key (two-step OpenSSL method) |
| 4 | Configuring VS Code (settings.json with double backslashes) |
| 5 | Common Errors and Solutions |
| 6 | What the Key Looks Like (PEM vs DER) |
| 7 | Security Considerations |
| 8 | Testing Without a Key (simulator only) |
| 9 | Troubleshooting Checklist |
| 10 | Quick Reference (PowerShell commands) |

**Key facts:** PEM format will NOT work — must convert to DER with `pkcs8 -topk8`. Simulator does not require a key. Use double backslashes in VS Code JSON paths.

---

### 3. `garmin_development_addendum.md`
**Purpose:** Practical development workflow — setup, manifest pitfalls, project structure, and install/uninstall procedures discovered during real development.

| § | Section |
|---|---------|
| 1 | Developer Key — Quick Reference |
| 2 | Manifest.xml for Data Fields (fit-contrib-activities causes build error) |
| 3 | Project Structure Requirements (drawables.xml location, monkey.jungle) |
| 4 | Array Initialization Type Safety (`[] as Array<Number>`) |
| 5 | Simulator Configuration (launch.json required) |
| 6 | Installing Apps on Physical Devices (Back button trigger) |
| 7 | Uninstalling Sideloaded Data Fields (Garmin Express only) |
| 8 | Heart Rate Data Limitations (no raw HR data; smoothed BPM only) |
| 9 | Complete Development Workflow (first-time setup + iterative cycle) |
| 10 | Key Takeaways Summary |

**Key facts:** `fit-contrib-activities` in a data field manifest = build error. `drawables.xml` must be INSIDE the `drawables/` folder. Garmin Express is the only way to uninstall sideloaded apps. Changing app ID creates a new separate app.

---

### 4. `connectiq_code_patterns.md`
**Purpose:** Copy-paste code patterns and independently written templates — the fastest path to working code.

| § | Section |
|---|---------|
| 1 | SensorHistory — Heart Rate History Iterator (capability guard, safe iteration, graph drawing) |
| 2 | SimpleDataField — Basic Single-Value Field (cycling values example) |
| 3 | DataField — Adaptive Layout with Dynamic Font Selection (pickFont helper, full skeleton) |
| 4 | Input — Button and Touch Handling (BehaviorDelegate full template) |
| 5 | Manifest.xml Patterns (correct data field manifest) |
| 6 | Common Patterns for Data Fields (CircularBuffer, AlgorithmManager, null handling, indoor step/distance baseline, change-detection redraw) |
| 7 | Best Practices (capability checking, layout caching, null safety, fillPolygon warning) |
| 8 | Code Organization (file structure, class member order) |
| 9 | getInitialView — Correct Return Type Syntax (bracket notation, no public, no ?, no cast) |

**Key facts:** `:period` for HR history = sample count, not seconds. `SimpleDataField` for single values; `DataField` for custom layouts + automatic full-screen view. Indoor step/distance requires `ActivityMonitor` with baseline captured at `onTimerStart()`. Boolean members initialized to `false` trigger static analyzer warnings on `if (member)` — initialize to `true` when the first state should be active. `getInitialView()` must use `[WatchUi.Views] or [WatchUi.Views, WatchUi.InputDelegates]` bracket notation — `Array<...>` form causes a build error.

---

### 5. `monkeyc_analyzer_unreachable_statement_guide.md`
**Purpose:** Root cause analysis of the Monkey C static analyzer's "Statement is not reachable" warning and permanent solutions that produce a zero-warning build.

| § | Section |
|---|---------|
| 1 | The Core Problem |
| 2 | How the Analyzer Works (traces from initialize(), ignores runtime changes) |
| 3 | Patterns That Trigger the Warning (Boolean members, null guards on non-nullable returns, arithmetic on init values) |
| 4 | Solutions That Work (use parameters/nullable API returns; branchless arithmetic; remove unnecessary null guards) |
| 5 | Solutions That Do NOT Work |
| 6 | Diagnostic Technique |
| 7 | Activity.Info Timer State Reference (TIMER_STATE_ON pattern) |
| 8 | Non-Nullable API Methods (ActivityMonitor.getInfo, System.getTimer, etc.) |
| 9 | Checklist — Preventing Future Warnings |
| 10 | Summary |

**Key facts:** The analyzer traces from `initialize()` — branch on nullable API returns or function parameters, never on member variables initialized to a fixed value. `ActivityMonitor.getInfo()` is non-nullable (don't null-check the return). Use `+1.0f` branchless trick instead of a `if (deltaMs <= 0)` guard.

---

### 6. `venu3_practical_lessons.md`
**Purpose:** Device-specific practical knowledge that is not in the SDK docs — data screen configuration, layout constraints, and timing edge cases.

| § | Section |
|---|---------|
| 1 | Data Screen Configuration on Venu 3 (phone app vs on-watch split, how to access during activity) |
| 2 | Layout Guidelines for 1-of-4 Data Field Displays (tested working values, bezel interaction rules) |
| 3 | Intensity Minutes Field — Baseline Capture Timing Note |

**Key facts:** Data screen configuration is only accessible during an active activity (long-press lower button). Phone app cannot see screens created on-watch. Sideloaded apps never auto-create screens. Bezel clips corners — pushing columns outward requires pulling rows toward vertical center.

---

## Project Documents

### 7. `steps_and_time_development_lessons.md`
**Project:** StepsAndTime data field (daily step count + current time of day)
**App type:** Data Field

| § | Section |
|---|---------|
| 1 | Project Overview |
| 2 | Automatic Full-Screen View Discovery (DataField auto-creates dual view) |
| 3 | XML Special Character Handling (ampersand prohibited even as &amp;) |
| 4 | Permission Requirements — ActivityMonitor Needs None |
| 5 | Font Sizing and Layout (two-row percentage layout) |
| 6 | Color Consistency with Native Fields (getBackgroundColor() pattern) |
| 7 | SimpleDataField vs DataField |
| 8 | Activity Compatibility (sideloaded vs store behavior table) |
| 9 | Time Formatting Best Practices (12/24hr auto-detect, edge cases, format specifiers) |
| 10 | Complete Working Code (StepsTimeView.mc + StepsTimeApp.mc) |

**Key facts:** `ActivityMonitor` requires no manifest permission. `DataField` auto-creates a full-screen page. App name comes from `strings.xml` not `.prg` filename. Ampersand is prohibited in app names.

---

### 8. `consolidated_field_development_lessons.md`
**Project:** ConsolidatedField — dual-triangle layout showing steps, distance, speed, intensity minutes, time, and daily step progress in one field slot
**App type:** Data Field

| § | Section |
|---|---------|
| 1 | The 2-App Limit Problem and Consolidation Strategy |
| 2 | Dual-Triangle Layout Design (concept diagram, why triangles work) |
| 3 | Position-Aware Rendering via getObscurityFlags() (top vs bottom slot detection) |
| 4 | Bezel-Aware Horizontal and Vertical Positioning (column values, rule of thumb) |
| 5 | Iterative Label and Unit Offset Tuning (labelOff values, unit subscript evolution) |
| 6 | Shifting the Entire Layout as a Unit (uniform percentage shift) |
| 7 | The Steps/Distance Zero Bug — Wrong API (elapsedDistance is GPS-only) |
| 8 | App Name vs. PRG Filename |
| 9 | Final Working Layout Values (tested x/y percentages for top and bottom slots) |
| 10 | Complete Compute Pattern for Steps and Distance (onTimerStart baseline + compute rolling speed) |

**Key facts:** Venu 3 hard limit of 2 concurrent app instances. `OBSCURE_BOTTOM` flag detects bottom slot. `Activity.Info.elapsedDistance` is null indoors — use `ActivityMonitor.getInfo().distance` with baseline. Label offsets are fixed pixels, not percentages.

---

### 9. `timerhr_field_development_lessons.md`
**Project:** TimerHRCalories data field (elapsed timer, heart rate with zone coloring, calories)
**App type:** Data Field

| § | Section |
|---|---------|
| 1 | HR Zone Coloring via UserProfile API (6-element zone array, sport-specific zones, color scheme, complete helper function) |
| 2 | Negative Y-Coordinates Strip Font Ascent Padding (y=0 ≠ top edge; empirical values for NUMBER_MILD) |
| 3 | FONT_NUMBER_MILD for Tight Vertical Spaces (font size hierarchy, approximate heights table) |
| 4 | Partial Edit Failure — "Undefined Symbol" Build Error (diagnostic and prevention) |
| 5 | Dual Independent X-Column Strategy (different row positions at different x spreads) |
| 6 | DST-Proof "Today" Boundary for Activity History (what fails, ClockTime.dst bug, Application.Storage minimum-offset solution) |

**Key facts:** `UserProfile.getHeartRateZones()` requires `UserProfile` permission. Zone colors: Z1=LT_GRAY, Z2=BLUE, Z3=GREEN, Z4=ORANGE, Z5=RED. Negative y strips font ascent padding. `FONT_NUMBER_MILD` ≈ 45px; `FONT_NUMBER_MEDIUM` ≈ 60px. Store HR zone array in local variable before null-checking to avoid static analyzer warning. `ClockTime.dst` is always 0 on real devices (firmware bug) — never use it. DST-safe "today" boundary: persist minimum `timeZoneOffset` in `Application.Storage`; use for day-number comparison via integer division of UTC epoch.

---

### 10. `hrv_logger_development_lessons.md`
**Project:** HRVRawLogger widget (real-time beat-to-beat HRV data capture)
**App type:** Widget

| § | Section |
|---|---------|
| 1 | setEnabledSensors Required Before registerSensorDataListener |
| 2 | Backlight Kills Sensor Callbacks on Venu 3 |
| 3 | Venu 3 Button Mapping (enter, menu, esc — KEY_ESC not interceptable) |
| 4 | heartBeatIntervals Warm-Up Period (first 1–2 callbacks return empty) |
| 5 | heartBeatIntervals Requires Active ActivityRecording Session (firmware-level requirement) |
| 6 | Zero Values Indicate Signal Loss (filter `[0]` from interval arrays) |
| 7 | Architecture Notes (why Widget type is required; sensor lifecycle pattern) |

**Key facts:** `setEnabledSensors()` must be called BEFORE `registerSensorDataListener()`. `Attention.backlight()` kills HR sensor callbacks on Venu 3. Beat-to-beat intervals ONLY flow when an `ActivityRecording.Session` is active — firmware requirement. `FitRecording` permission required. Call `session.discard()` to prevent phantom workouts.

---

### 11. `skin_temp_widget_development_lessons.md`
**Project:** SkinTempWidget (skin temperature history widget with glance view)
**App type:** Widget

| § | Section |
|---|---------|
| 1 | Glance View Architecture (dc is bounded by glance slot, starts right of launcher icon) |
| 2 | Glance View Styling — Full Walkthrough (7-iteration process) |
| 3 | AMOLED Brightness Calibration (empirical brightness values; custom gray construction) |
| 4 | Pixel Sampling for UI Matching |
| 5 | Gradient + Corner Masking Technique |
| 6 | Noise Texture via RLE Column Patterns |
| 7 | Launcher Icon — Transparent Trick |
| 8 | Swipe and Touch Navigation on Venu 3 (confirmed direction mapping) |
| 9 | onSwipe vs onNextPage / onPreviousPage (why onSwipe is unreliable) |
| 10 | Drag Events for Smooth Scrolling (getCoordinates returns position, not delta) |
| 11 | dc.setClip() for Graph Rendering |
| 12 | App ID Conflicts Between Projects |
| 13 | Timezone Bug — Gregorian.moment() Uses UTC |
| 14 | Widget Memory Limit (64 KB) |
| 15 | fillPolygon Performance on Device (watchdog timeout in loops — use fillRectangle) |
| 16 | Graph Layout — graphBottom and Bezel Clearance (tested safe constants) |
| 17 | SensorHistory.getTemperatureHistory() — Period Parameter (count, not seconds) |
| 18 | Page Indicator Dots (custom implementation — why ViewLoop was avoided) |
| 19 | Application.Storage — No Permission Required ("Storage" is not a valid permission ID) |
| 20 | Persistence Layer Architecture (TempStorage — 7-day rolling window design) |
| 21 | Glance Annotations — complete guide: annotation pattern, AppBase crash, onStart() lifecycle, 32 KB limit |

**Key facts:** Glance dc x=0 is already past the launcher icon. AMOLED: brightness 91 = visible dark gray. `fillPolygon` causes watchdog timeout in loops. `DragEvent.getCoordinates()` returns position not delta. `Gregorian.moment()` uses UTC — use `System.getClockTime()` for local time. Widget memory limit is 64 KB. Temperature `:period` = sample count. Correct `(:glance)` pattern: annotate the `getGlanceView()` *function* and the `GlanceView` *class* — never annotate the AppBase *class* (causes silent crash). On non-live-update devices `onStart()` runs every 30 seconds in the background glance cycle; guard expensive work with `isGlanceModeEnabled`. Glance memory limit is 32 KB; check simulator View Memory screen before release.

---

### 12. `customhrv_development_lessons.md`
**Project:** CustomHRV watch app (multi-view app for recording and reviewing HRV readings)
**App type:** Watch App (768 KB)

| § | Section |
|---|---------|
| 1 | onSelect() Does Not Fire for Row Taps on Touchscreen (use onTap with coordinate mapping) |
| 2 | Swipe Direction vs Callback Name — Confirmed on Device (UP=onNextPage, DOWN=onPreviousPage — authoritative) |
| 3 | Adding Time to Storage Records — Schema Migration Awareness |
| 4 | HRVStorage.addReading() Signature Change (updating caller chains when schemas evolve) |
| 5 | Multi-Flag Data Quality System (qualityBad, duplicateDate, timeOutlier flags) |
| 6 | DetailView Delete Confirmation Pattern (two-tap destructive action without extra screen) |
| 7 | Display Index vs Storage Index (display reverses order; conversion formula) |
| 8 | Cast Expressions Are Not lvalues (extract to local variable before assigning through cast) |
| 9 | Bezel Safe Zone — Full-Screen Watch App Layout (header y=42, content y=70–355, footer y=355) |
| 10 | Unicode Arrow Characters Don't Render — Use fillPolygon Instead (ArrowUtils pattern, inline arrow layout) |
| 11 | App List Placement — Activities vs Apps (what determines which list an app appears on) |

**Key facts:** `onSelect()` fires only for physical button, not screen tap — use `onTap()`. Swipe UP = `onNextPage()`, Swipe DOWN = `onPreviousPage()` — confirmed by device video. Cast expressions (`(x as Type).field = val`) are illegal — extract to local var first. Unicode `▲▼` don't render; use `fillPolygon()`. Full-screen safe zone: y=70 to y=355. Schema migration: always treat new storage fields as nullable. `watch-app` + `Fit` permission + `ActivityRecording` → Activities list; `watch-app` without these → Apps list; `widget` type → Apps list (+ Glances if `getGlanceView()` implemented). Use `Fit` permission (not `FitRecording`) for `ActivityRecording` — confirmed working.

---

### 13. `stimtracker_development_lessons.md`
**Project:** StimTracker watch app (caffeine tracking with decay model, profiles, history, settings)
**App type:** Watch App (768 KB) with Glance

| § | Section |
|---|---|
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
| 19 | FONT_NUMBER_MEDIUM rendering offset (visual top ≠ y coordinate) |

**Key facts:** Always pass the constructed view to the delegate — never let the delegate construct its own view. `onMenu()` = long-press back button. `FONT_NUMBER_MEDIUM` without VCENTER places y at bounding-box top, not visual glyph top — visual top appears ~25px below the specified y value (measured: y=219 → visual top at y=244, visual bottom at y=303). Circle-clipped bars use arc radius 210, not screen radius 227. Hitbox bottom should be flush with Save button top. Swipe UP/DOWN scroll list screens; back button is the only exit.

---

## Document Relationships

```
garmin_connectiq_knowledge_base.md  ← Hub: referenced by all other files
        │
        ├── garmin_developer_key_guide.md       (setup prerequisite)
        ├── garmin_development_addendum.md      (setup prerequisite)
        ├── connectiq_code_patterns.md          (code templates)
        ├── monkeyc_analyzer_unreachable_statement_guide.md  (build quality)
        │
        ├── venu3_practical_lessons.md          (device-specific)
        │
        ├── steps_and_time_development_lessons.md    (first project — simplest)
        ├── consolidated_field_development_lessons.md (data field — complex layout)
        ├── timerhr_field_development_lessons.md      (data field — HR zones)
        │
        ├── hrv_logger_development_lessons.md    (widget — sensor lifecycle)
        ├── skin_temp_widget_development_lessons.md  (widget — glance, storage)
        │
        ├── customhrv_development_lessons.md    (watch app — most complex)
        └── stimtracker_development_lessons.md  (watch app — StimTracker patterns)
```

---

*Last updated: March 2026 — SDK 8.4.1*
