# HABiesie — Security Domain Context
> **Living document.** Update after every change to the security package.  
> Paste alongside `PROJECT_STATE.md` when working on security.
>
> **⚠️ Superseded by `docs/domains/SECURITY_CONTRACT.md` — use the contract for real
> work.** This file is a quick-reference summary, dated 2026-04-13, essentially
> untouched since while the security domain absorbed 75+ tracked bugs (BUG-S01 through
> BUG-S75+) and a full deep-drift sweep earlier the same session as this correction
> (2026-08-21). Package Files listed 4 nonexistent files; Known Problems #4 (trust model
> not wired) was fixed months ago via the S2/S3 classifier rebuild. #1/#3/#5 not
> individually re-verified this pass.

---

## 🎯 Intended Design

A layered event engine that detects movement, classifies it, captures evidence, and routes notifications:

```
Camera Motion → Trigger → Classify → Snapshot → Path Build → Notify
                              ↕
                      Presence + Trust Context
```

### Classification States
| State | Meaning |
|---|---|
| `arrival` | Known person arriving via gate + camera confirmation |
| `visitor` | Movement at gate/perimeter, gate closed |
| `service_person` | Low-trust person (staff/contractor) on property |
| `intruder` | Unknown motion, no presence explanation |
| `critical_intrusion` | Multiple zones, night mode, no known presence |
| `idle` | No active event |

---

## 📁 Package Files

**⚠️ Corrected 2026-08-21 — this list named 4 files that don't exist
(`security_templates.yaml`, `security_state.yaml`, `security_cameras.yaml`,
`security_notifications.yaml`). Replaced with the live 9-file inventory (was also
missing 2 files in SECURITY_CONTRACT.md's own table until this session — see that
contract's File Inventory for full descriptions).**

```
packages/security/  (9 files)
  cameras_core.yaml           # group definitions (perimeter/grounds/inside cameras)
  cameras_processing.yaml     # debounce sensors, correlation, last-event timestamps
  security_helpers.yaml       # input_boolean/input_number/input_datetime/input_text helpers
  security_core.yaml          # boundary_permissive_window, security_mode, trust_mode
  security_logic.yaml         # event classification, correlation, threat score/level
  security_zones.yaml         # zone aggregation binary sensors
  security_automations.yaml   # snapshot capture, event lifecycle, event router
  security_alarm.yaml         # IDS Hyyp interface stub (not yet wired)
  security_history_cleanup.yaml # one-shot manual camera-history cleanup script
```

---

## 🔴 Critical Entities (DO NOT RENAME)

```
sensor.security_trigger_camera          # Central trigger — which camera fired
sensor.security_event_classification    # Current event type
sensor.security_threat_level            # none/low/medium/high/critical
sensor.security_threat_score            # 0-100 numeric score
sensor.security_movement_confidence     # Confidence of current classification
input_text.security_event_session       # Current event timeline (pipe-delimited)
input_text.security_last_path           # Movement path across zones
input_text.security_last_motion_camera  # Last camera that triggered
input_text.security_last_motion_image   # Path to last snapshot
input_datetime.security_event_start     # When current event started
input_datetime.last_security_event      # Last any security event
input_datetime.last_intruder_event      # Last intruder classification
input_datetime.last_visitor_event       # Last visitor classification
input_boolean.security_alert_active     # Active security alert flag
input_boolean.security_system_enabled   # Master on/off — stops ALL captures + notifications
input_boolean.inside_cameras_armed      # Managed by arming automation — do not set manually
input_boolean.inside_cameras_schedule_override  # Dashboard toggle — forces inside cams armed
```

### Zone Aggregation Sensors
```
binary_sensor.security_perimeter_motion   # cam01 street area
binary_sensor.security_grounds_motion     # cam04/05/09/10/11 property
binary_sensor.security_inside_house_motion # cam14/15 interior
```

### Camera Naming (LOCKED)
```
camera.cam01_street_driveway
camera.cam04_car_port_front
camera.cam05_front_driveway
camera.cam06_front_entrance
camera.cam07_front_kitchen
camera.cam09_back_bedroom
camera.cam10_pool_bar
camera.cam11_back_pond
camera.cam14_lounge
camera.cam15_passage
```

### Per-camera History Helpers
```
input_text.cam01_street_driveway_history   # pipe-delimited image paths
input_text.cam04_car_port_front_history
input_text.cam05_front_driveway_history
input_text.cam06_front_entrance_history
input_text.cam07_front_kitchen_history
# ... pattern: input_text.camXX_location_history
```

### Per-camera Last-seen Sensors
```
sensor.cam01_last_seen_seconds
sensor.cam04_last_seen_seconds
sensor.cam05_last_seen_seconds
# ... pattern: sensor.camXX_last_seen_seconds
```

---

## ✅ Inside Camera Arming Schedule (2026-04-14)

cam14 (lounge) and cam15 (passage) use an occupancy+time arming schedule:

| Condition | Inside Cameras |
|---|---|
| 22:00–06:00 (nighttime) | Armed — always |
| Nobody home (any time) | Armed — always |
| 06:00–22:00 + family home | Disarmed — snapshots + captures suppressed |
| `inside_cameras_schedule_override` ON | Armed — always (dashboard override) |

**Automation:** `security_inside_cameras_arming` manages `input_boolean.inside_cameras_armed`.  
**Guard:** `security_capture_each_camera_motion` checks the boolean and stops for cam14/cam15 when disarmed.  
**Dashboard:** Add `inside_cameras_schedule_override` + `security_system_enabled` to security control card.

---

## ✅ Motion Filter Tuning (2026-04-14)

All outdoor cameras have `delay_on` added to filter rain/wind/spider false triggers:

| Camera group | delay_on | delay_off | Rationale |
|---|---|---|---|
| cam05, cam06 (front — arrival critical) | 1s | 25s | Fast response for gate/entrance |
| cam01, cam04 (street/carport) | 1s | 30s | Fast + longer capture window |
| cam07 (kitchen/side) | 2s | 30s | Side entry, more wind exposure |
| cam09, cam10, cam11 (rear) | 2s | 45s | Trees/pool/wind false triggers |
| cam14, cam15 (inside) | none | 15s | No weather, instant response |

**Hikvision side recommendations:**  
- Reduce detection area to exclude tree lines, sky, pool water surface  
- Sensitivity 20 is good — do not raise above 30 on outdoor cameras  
- Dynamic analysis for motion: keep enabled (helps filter stationary false triggers)  
- If available on NVR model: use Perimeter Intrusion instead of Motion Detection for more accurate person/vehicle detection

---

## ⚠️ Known Problems (Root Causes Understood)

**⚠️ Only #4 individually re-verified 2026-08-21 (confirmed fixed). #1/#3/#5 not
re-checked this pass — #2 was independently re-confirmed still open via
SECURITY_CONTRACT.md ISSUE 9 earlier the same session. SECURITY_CONTRACT.md's Bug
Catalog (75+ BUG-S entries, all current) is the authoritative source — this section
predates the vast majority of it.**

### 1. Duplicate Images
- **Cause:** Multiple triggers fire on same motion event (NVR sends multiple signals)
- **Fix:** `delay_on` in cameras_processing.yaml now filters rapid re-fires
- **Fix:** Deduplication check in history writer (checks last entry == new entry)
- **Status:** Substantially reduced — monitor

### 2. input_text Overflow
- **Cause:** `input_text.security_event_session` stores `cam_id,image_path,time|cam_id,...` — exceeds 255 chars in active sessions
- **Fix needed:** Replace with pyscript-managed state or split into multiple bounded entities
- **Status:** Not yet fixed — workaround is truncating to last 3 events

### 3. Camera Feed Instability
- **Cause:** Hikvision NVR analog RTSP reliability + go2rtc timeout issues
- **Fix needed:** Move to direct IP cameras (ColorVu/AcuSense) — in progress
- **Status:** Hardware transition ongoing

### 4. ✅ FIXED — Trust Model Not Wired to Security
**Status corrected 2026-08-21** (SECURITY_CONTRACT.md ISSUE 8, doc-drift confirmed
2026-07-10 and re-verified again live this session): the S2/S3 classifier rebuild wired
trust filtering into `sensor.security_correlation`/`security_event_router` — not by
re-enabling the old commented-out per-automation conditions (which would have restored a
broken path referencing a deprecated helper), but structurally, at the classifier level.
Security does not run blind.
- ~~**Fix needed:** Re-enable `low_trust_present` and `staff_on_site` conditions in security classification~~
- ~~**Status:** Security currently runs blind — no presence/trust filtering~~

### 5. Template Errors on Unknown Timestamps
- **Cause:** `sensor.camXX_last_seen_seconds` fails when camera has never triggered
- **Fix needed:** Add `| default(0)` to all timestamp calculations
- **Status:** Partially fixed

---

## ✅ What Works Well

- Event classification logic (sensor.security_event_classification)
- Threat scoring (sensor.security_threat_score)
- Zone aggregation (perimeter/grounds/inside)
- Door/gate status display with escalation timing
- Security timeline markdown card (input_text.security_event_session rendering)
- Notification routing via script.notify_security_event

---

## 🚫 Deferred (Do Not Implement Yet)

- Frigate AI detection integration
- Face recognition
- Automatic gate control
- Vehicle detection/classification
- Multi-camera correlation scoring beyond current path tracking

---

## 🎯 Next Steps (Agreed Priority)

1. **Fix cam01/cam04/cam07 blank snapshots** — draw motion detection areas in Hikvision NVR → Configuration → Event → Motion Detection → Area Settings for each camera. No HA changes needed.
2. **Replace input_text session** — pyscript state object or split helpers
3. **Re-enable trust model filtering** in classification logic
4. **Add `from:` constraints** to all motion triggers
5. **Harden last_seen templates** with defaults

---

## 📐 Event Session Format (Current — Problematic)

```
# input_text.security_event_session format:
cam_id,/local/image_path.jpg,HH:MM:SS|cam_id,/local/image_path.jpg,HH:MM:SS

# Per-camera history format:
/local/snapshot1.jpg|/local/snapshot2.jpg|/local/snapshot3.jpg
```

**Problem:** Both formats overflow 255 chars. Replacement design TBD.

---

## 🔗 Dependencies

- **Presence system:** `binary_sensor.anyone_connected_home`, `binary_sensor.low_trust_present`, `binary_sensor.staff_on_site`
- **Context system:** `binary_sensor.night_confirmed`, `binary_sensor.boundary_permissive_window`
- **Alert system:** `script.notify_security_event`, `input_boolean.security_alert_active`
- **Lighting:** Security lighting automations watch `binary_sensor.security_perimeter_motion`

---

*Last updated: <!-- DATE -->*  
*Updated by: <!-- CHANGE SUMMARY -->*
