# HABiesie — Security Domain Context
> **Living document.** Update after every change to the security package.  
> Paste alongside `PROJECT_STATE.md` when working on security.

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

```
packages/security/
  security_helpers.yaml       # input_text, input_datetime, input_boolean helpers
  security_templates.yaml     # threat score, classification, movement path sensors
  security_state.yaml         # zone aggregation, perimeter/grounds/house sensors
  security_automations.yaml   # event router, snapshot capture, path tracking
  security_core.yaml          # Hikvision integration config
  security_cameras.yaml       # Camera entity definitions
  security_notifications.yaml # Security-specific notification routing
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

### 4. Trust Model Not Wired to Security
- **Cause:** Trust filtering logic was commented out during testing and never restored
- **Fix needed:** Re-enable `low_trust_present` and `staff_on_site` conditions in security classification
- **Status:** Security currently runs blind — no presence/trust filtering

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
