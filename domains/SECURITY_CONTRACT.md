# HABiesie — Security Domain Living Contract
> **Authoritative reference.** Produced by deep audit 2026-04-13.  
> Update after every structural change to the security package.  
> Read alongside `PROJECT_STATE.md` when working on security.

---

## Section 1: System Overview

### Intended Design

A layered event engine that detects movement, classifies it by threat level, captures
evidence (snapshots), and routes notifications. All security state flows through a small
set of locked sensor names that dashboards and automations depend on.

```
NVR motion signal
      │
binary_sensor.camXX_motiondetection   ← raw Hikvision/go2rtc signal
      │   (may fire multiple times per event)
      ▼
binary_sensor.camXX_motion_valid      ← debounced, 15–30s delay_off
(cameras_processing.yaml)
      │
      ▼
binary_sensor.security_{zone}_motion  ← zone aggregation: perimeter / grounds / inside
(security_zones.yaml)
      │
      ├──▶ sensor.security_trigger_camera      ← best active camera (priority list)
      │    (security_logic.yaml)
      │
      ├──▶ sensor.security_movement_confidence ← high/medium/low/none
      │    (security_logic.yaml)
      │
      ├──▶ sensor.security_movement_path       ← current zone (single value)
      │    (security_logic.yaml)
      │
      ├──▶ sensor.security_correlation         ← final event classification
      │    (security_logic.yaml)
      │
      ├──▶ sensor.security_threat_score        ← 0–100
      │    (security_logic.yaml)
      │
      └──▶ sensor.security_threat_level        ← low/elevated/warning/critical
           (security_logic.yaml)
                  │
                  ▼
    automation triggers (security_automations.yaml)
                  │
          ┌───────┴────────┐
          ▼                ▼
   camera.snapshot    script.notify_security_event
   → /config/www/     (notifications/notify_security_events.yaml)
   → input_text.*
```

### Hardware Summary — Camera Fleet (updated 2026-05-08)

#### Hikvision NVR (analog — DS-7116HGHI-F1)

No AI detection, no per-object classification. Motion signal is pixel-diff only.
go2rtc timeout instability. Being progressively replaced by AcuSense IP cameras.

| Camera | Zone | Status | Notes |
|--------|------|--------|-------|
| cam01_street_driveway | Perimeter Front | **DEPRECATED 2026-05-07** | Replaced by camip03/05. Entity kept for UI compatibility. |
| cam04_car_port_front | Grounds Front | Active | Carport / front-left |
| cam05_front_driveway | **Inside House** | Active — **MOVED 2026-05-07** | Physical camera moved inside garage. Entity ID unchanged. Gated by inside_cameras_armed. |
| cam06_front_entrance | Grounds Front | **REMOVED 2026-05-08** | Physically uninstalled. Entity kept to avoid breaking UI. |
| cam07_front_kitchen | Grounds Front | Active | Side entry / kitchen window |
| cam09_back_bedroom | Grounds Rear | Active | Rear right |
| cam10_pool_bar | Grounds Rear | **DEPRECATED 2026-05-07** | Replaced by ipcam02. Entity kept for UI compatibility. |
| cam12_back_pond | Grounds Rear | Active | |
| cam14_lounge | Inside House | Active | Armed by schedule / occupancy |
| cam15_passage | Inside House | Active | Armed by schedule / occupancy |

#### Hikvision AcuSense IP Cameras (new fleet — from 2026-05-07)

Full AI detection: person/vehicle classification, region entrance/exit, line crossing, field intrusion.
Entity prefix pattern: `ipcamXX` for all IP cameras (ipcam01–ipcam05) — standardised 2026-05-08.

| Camera | Entity prefix | Zone | Status | Location |
|--------|---------------|------|--------|----------|
| IPCam03_Driveway | `ipcam03_driveway` | Grounds Front | **Active** | Driveway inside gate. regionentrance/exiting switches ON. |
| IPCam04_Pool_Bar | `ipcam04_pool_bar` | Grounds Rear | **Active** | Pool / bar area |
| IPCam02_Street_Driveway_Down | `ipcam02_street_driveway_down` | Perimeter Front | **Active 2026-05-08** | Street level — lower approach. Limited sensors while initialising. |
| IPCam05_Back_Boundary | `ipcam05_back_boundary` | Perimeter Rear | **Active 2026-05-08** | Rear boundary — first active rear perimeter sensor |
| IPCam01_Street_Driveway_Up | `ipcam01_street_driveway_up` | Perimeter Front | **Active 2026-05-08** | Street level — upper approach. regionentrance switch ON. |

**Available sensors per IP camera** (confirmed from entity registry):

| Sensor type | Entity suffix pattern | ipcam01 (street up) | ipcam02 (street down) | ipcam03 (driveway) | ipcam04 (pool bar) | ipcam05 (rear boundary) |
|-------------|----------------------|---------------------|----------------------|---------------------|---------------------|--------------------------|
| `motiondetection` | `binary_sensor.*_motiondetection` | ⚠️ disabled¹ | ⚠️ disabled¹ | ⚠️ disabled¹ | ⚠️ disabled¹ | ⚠️ disabled¹ |
| `regionentrance` | `binary_sensor.*_regionentrance` | ✅ switch ON | ✅ active | ✅ switch ON | switch off | switch off |
| `regionexiting` | `binary_sensor.*_regionexiting` | — | — | ✅ switch ON | switch off | — |
| `linedetection` | `binary_sensor.*_linedetection` | ✅ active | ✅ active | ✅ active | ✅ active | ✅ active |
| `fielddetection` | `binary_sensor.*_fielddetection` | ✅ active | ✅ active | ✅ active | ✅ active | ✅ active |
| `scenechangedetection` | `binary_sensor.*_scenechangedetection` | — | ✅ active | present | — | — |
| `tamperdetection` | `binary_sensor.*_tamperdetection` | — | — | present | — | — |

¹ `motiondetection` entity exists but "Notify Surveillance Center" is unchecked in camera UI — it will not fire. All cameras now use Smart Events (fielddetection + linedetection + regionentrance) as primary motion signal.

**Entrance/exit debounce sensors (wired in cameras_processing.yaml 2026-05-08):**
- `binary_sensor.ipcam03_driveway_entrance_valid` — property entry confirmed (gate open, vehicle/person in driveway)
- `binary_sensor.ipcam03_driveway_exit_valid` — departure from property
- `binary_sensor.ipcam01_street_driveway_up_entrance_valid` — person/vehicle approaching gate from street (primary)
- `binary_sensor.ipcam02_street_driveway_down_entrance_valid` — secondary street approach (may be inactive until ipcam02 initialises)

### Visitor vs Arrival Logic (updated 2026-05-08)

The gate is a closed structure — `ipcam03_driveway` can only detect motion AFTER the gate is opened.

| Scenario | Sensor(s) active | Classification |
|----------|-----------------|----------------|
| Someone on street approaching gate | ipcam01/02 motion or entrance | **Visitor** (if gate closed) |
| Gate opens + driveway activity | gate_sensor=on + ipcam03 entrance/motion | **Arrival** (family entering) |
| ipcam03 fires with gate closed | ipcam03 motion, gate=off | Potential intruder (not a visitor) |
| Vehicle on street, no driveway | ipcam01/02 only | Vehicle Approaching (perimeter) |
| Vehicle in driveway | ipcam03 motion or entrance | Vehicle In Driveway (inside boundary) |

---

## AcuSense Optimisation Roadmap

The new IP cameras (Hikvision AcuSense) provide far more reliable motion signals than the
NVR analog cameras. All 5 ipcams are now fully migrated to Smart Events.

### Phase 1 — Wired (done 2026-05-07)
- `motiondetection` as initial primary trigger for debounce sensors
- Full snapshot capture pipeline wired in
- `ipcam01` and `ipcam02` active

### Phase 2 — Entrance/exit debounce added (done 2026-05-08)
Added `regionentrance`/`regionexiting` debounce sensors to `cameras_processing.yaml` for:
- `ipcam03_driveway_entrance_valid` + `ipcam03_driveway_exit_valid` (switches confirmed ON)
- `ipcam01_street_driveway_up_entrance_valid` (switch ON)
- `ipcam02_street_driveway_down_entrance_valid` (active 2026-05-10)

Visitor trigger uses entrance_valid as primary; arrival uses ipcam03_entrance_valid for confirmation.

### Phase 3 — Wire ipcam01/ipcam02/ipcam05 (done 2026-05-08)
IPCam01 (street up), IPCam02 (street down), IPCam05 (rear boundary) wired in:
1. Debounce sensors in `cameras_processing.yaml` with `ipcam` prefix
2. Groups updated in `cameras_core.yaml`
3. `security_zones.yaml` perimeter_front uses `ipcam01 OR ipcam02`; perimeter_rear uses `ipcam05`
4. Confidence scoring and trigger camera priority updated in `security_logic.yaml`
5. All entity names standardised to `ipcam01–ipcam05` prefix (renamed from mixed `camip`/`ipcam` 2026-05-08)

### Phase 3b — Smart Events migration (done 2026-05-10)
All 5 ipcams migrated to Smart Events — `motiondetection` "Notify Surveillance Center" disabled
in camera UI for each camera. `motion_valid` debounce sensors in `cameras_processing.yaml` now use:
- `fielddetection` (Intrusion zone — AI-filtered person/vehicle, configurable dwell threshold)
- `linedetection` (Line Crossing — directional trigger)
- `regionentrance` (Region Entrance — where applicable)

This eliminates false triggers from pixel-diff motion (rain, headlights, pool water, small animals).
Each camera's Intrusion zone/Line Crossing was configured in iVMS before disabling motiondetection.
Order of migration: ipcam05 → ipcam04 → ipcam03 → ipcam02 → ipcam01.

### Phase 4 — Per-camera region configuration (Hikvision app)
Configure regions/lines on each camera via Hikvision iVMS or web UI:
- **IPCam03 (driveway)**: Region entrance covering driveway entry point
- **IPCam04 (pool bar)**: Region entrance covering pool area gate/entry
- **IPCam01/02 (street)**: Line detection on pavement (direction: towards gate)
- **IPCam05 (rear boundary)**: Field detection covering rear wall/fence

### Phase 5 — Confidence scoring rethink
Once `regionentrance` is primary:
- Any single AcuSense camera = minimum `medium` confidence (person/vehicle verified at hardware)
- Multiple AcuSense cameras = `high` confidence
- NVR-only camera alone = `low` confidence (retain for backward compat)
- Allows removing the `nobody_home` dependency from intruder classification for AcuSense cameras

---

## Section 2: File Inventory

### Package Files (`packages/security/`)

| File | Purpose |
|------|---------|
| `cameras_core.yaml` | Group definitions: security_perimeter_cameras, security_grounds_front_cameras, security_grounds_rear_cameras, security_inside_house_cameras |
| `cameras_processing.yaml` | Debounce sensors (`camXX_motion_valid`), camera correlation binary sensors, per-camera last event timestamp sensors, trigger-based last_seen_seconds sensors (1-minute update), EZVIZ doorbell integration |
| `security_helpers.yaml` | All input helpers: 4 input_boolean, 3 input_number, 4 input_datetime, 22 input_text (per-camera images + history × 10 cams, plus event tracking) |
| `security_core.yaml` | Binary sensors for boundary_permissive_window, visibility/weather conditions, lighting state; Sensors for security_mode, trust_mode, lighting_intent |
| `security_logic.yaml` | Core logic sensors: event classification, trigger camera selection, correlation engine, movement confidence/path, intruder level, threat score and threat level |
| `security_zones.yaml` | Zone aggregation binary sensors: perimeter front/rear/combined, grounds, external, inside house |
| `security_automations.yaml` | All automations: snapshot capture (×2 overlapping), movement path tracking, event lifecycle start/end, event router, visitor detection, arrival detection, grounds/rear/house motion, rear perimeter, gate open action |

### No legacy automations in `automations.yaml`

Grep of `automations.yaml` for security/cam/motion returns **zero matches**. All security
automations are correctly in the package files.

### UI-Created Helpers (from Watchman / entity registry)

No security-domain helpers were found to be UI-created. All are YAML-defined in
`security_helpers.yaml` and `context/context_presence.yaml`.

### Cross-domain helpers that security depends on (defined elsewhere)

| Entity | Defined in | Status |
|--------|-----------|--------|
| `input_boolean.guest_mode` | `packages/context/context_presence.yaml` | OK |
| `input_boolean.low_trust_present` | `packages/context/context_presence.yaml` (as input_boolean) + template binary_sensor | OK |
| `input_boolean.staff_on_site` | `packages/context/context_presence.yaml` | OK (but also a derived binary_sensor) |
| `input_boolean.entertaining_mode` | `packages/context/context_presence.yaml` | OK |
| `input_boolean.holiday_mode` | `packages/context/context_presence.yaml` | OK |
| `input_datetime.low_trust_start` | `packages/context/context_presence.yaml` | **MISSING** — commented out |
| `input_datetime.low_trust_end` | `packages/context/context_presence.yaml` | **MISSING** — commented out |
| `input_boolean.arrival_detected` | `packages/presence/presence_helpers.yaml` | OK |
| `binary_sensor.night_confirmed` | `packages/context/context_night.yaml` | OK |
| `binary_sensor.night_early` | `packages/context/context_night.yaml` | OK |
| `binary_sensor.security_night_mode` | `packages/context/context_global.yaml` | OK (= night_confirmed alias) |
| `binary_sensor.anyone_connected_home` | `packages/presence/presence_core.yaml` | OK |
| `binary_sensor.main_gate_sensor` | Hardware / ZHA | OK |
| `weather.home` | OpenWeatherMap integration | **MISSING per Watchman** |
| `input_boolean.openweathermap_api_limited` | context package | OK |
| `sensor.weather_api_health` | context package | OK |

---

## Section 3: Entity Reference (Complete)

### Core Output Sensors (LOCKED — DO NOT RENAME)

| Entity | Type | File | Output States | Consumers |
|--------|------|------|---------------|-----------|
| `sensor.security_trigger_camera` | template sensor | security_logic.yaml | `camera.camXX_...` or `none` | security_automations.yaml, notify script |
| `sensor.security_event_classification` | template sensor | security_logic.yaml | critical_intrusion, intruder, service_person, arrival, family_movement, none | security_event_router automation |
| `sensor.security_threat_level` | template sensor | security_logic.yaml | low, elevated, warning, critical | security_event_router, lighting reset, security_event_end |
| `sensor.security_threat_score` | template sensor | security_logic.yaml | 0–100 | security_threat_level |
| `sensor.security_movement_confidence` | template sensor | security_logic.yaml | none, low, medium, high | security_grounds_motion condition, security_correlation |
| `sensor.security_movement_path` | template sensor | security_logic.yaml | street, driveway, front_door, side_entry, rear_left, rear_center, rear_right, none | lighting_security.yaml, security_event_router |
| `sensor.security_correlation` | template sensor (no unique_id!) | security_logic.yaml | family_arrival, visitor, service_visit, intruder, house_intrusion, intruder_high, ignore, none | security_grounds_motion, security_rear_grounds_motion, security_house_motion |
| `sensor.security_intruder_level` | template sensor (no unique_id!) | security_logic.yaml | critical, warning, info, none | Possibly UI only — partially redundant to threat_level |
| `sensor.security_mode` | template sensor | security_core.yaml | away, night, home | notify_security_events.yaml |
| `sensor.security_trust_mode` | template sensor | security_core.yaml | scheduled_staff, detected_staff, normal | UI only |
| `sensor.security_lighting_intent` | template sensor | security_core.yaml | ignore, full, area, perimeter | lighting_security.yaml |
| `sensor.security_last_active_camera` | template sensor | cameras_processing.yaml | cam01, cam05, cam06, cam07, cam11 | UI |

### Debounce Motion Sensors (per-camera)

| Entity | Type | File | Delay Off | Note |
|--------|------|------|-----------|------|
| `binary_sensor.cam01_street_driveway_motion_valid` | template BS | cameras_processing.yaml | 20s | |
| `binary_sensor.cam04_car_port_front_motion_valid` | template BS | cameras_processing.yaml | 20s | |
| `binary_sensor.cam05_front_driveway_motion_valid` | template BS | cameras_processing.yaml | 20s | |
| `binary_sensor.cam06_front_entrance_motion_valid` | template BS | cameras_processing.yaml | 20s | |
| `binary_sensor.cam07_front_kitchen_motion_valid` | template BS | cameras_processing.yaml | 25s | |
| `binary_sensor.cam09_back_bedroom_motion_valid` | template BS | cameras_processing.yaml | 25s | |
| `binary_sensor.cam10_pool_bar_motion_valid` | template BS | cameras_processing.yaml | 25s | |
| `binary_sensor.cam11_back_pond_motion_valid` | template BS | cameras_processing.yaml | 30s | |
| `binary_sensor.cam14_lounge_motion_valid` | template BS | cameras_processing.yaml | 15s | |
| `binary_sensor.cam15_passage_motion_valid` | template BS | cameras_processing.yaml | 15s | |

### Zone Aggregation Sensors

| Entity | Type | File | Sources | Note |
|--------|------|------|---------|------|
| `binary_sensor.security_perimeter_front_motion` | template BS | security_zones.yaml | ipcam01 OR ipcam02 | Updated 2026-05-08 |
| `binary_sensor.security_perimeter_rear_motion` | template BS | security_zones.yaml | ipcam05_back_boundary | **Fixed 2026-05-08** — was ALWAYS OFF (cam03 never existed) |
| `binary_sensor.security_perimeter_motion` | template BS | security_zones.yaml | perimeter_front OR perimeter_rear | Covers all 3 street/boundary cameras |
| `binary_sensor.security_grounds_motion` | template BS | security_zones.yaml | expand(grounds_front + grounds_rear groups) | cam04/07/ipcam03/cam09/ipcam04/cam12/ipcam05 |
| `binary_sensor.security_external_motion` | template BS | security_zones.yaml | perimeter OR grounds | |
| `binary_sensor.security_inside_house_motion` | template BS | security_zones.yaml | expand(inside_house group) | cam14/cam15 only (cam05 moved to grounds_front 2026-05-08 — garage ≠ living space) |

### Per-Camera Timestamp Sensors

| Entity | Unique ID | Note |
|--------|-----------|------|
| `sensor.cam01_street_last_event` | cam01_last_event | Timestamp device_class |
| `sensor.cam04_carport_last_event` | cam04_last_event | Timestamp device_class |
| `sensor.cam05_driveway_last_event` | cam05_last_event | Timestamp device_class |
| `sensor.cam06_entrance_last_event` | cam06_last_event | **BUG: reads cam09 not cam06** |
| `sensor.cam07_kitchen_side_last_event` | cam07_last_event | |
| `sensor.cam09_rear_bedroom_last_event` | cam09_last_event | |
| `sensor.cam10_pool_bar_last_event` | cam10_last_event | |
| `sensor.cam11_back_pond_last_event` | cam11_last_event | |

**Note: No cam14, cam15 last event sensors defined.**

### Last-Seen Age Sensors (trigger-based, 1-min update)

| Entity | Note |
|--------|------|
| `sensor.cam01_last_seen_seconds` | Watchman reports missing in dashboard — possible load issue |
| `sensor.cam04_last_seen_seconds` | |
| `sensor.cam05_last_seen_seconds` | Watchman: missing |
| `sensor.cam06_last_seen_seconds` | Watchman: missing |
| `sensor.cam07_last_seen_seconds` | |
| `sensor.cam09_last_seen_seconds` | |
| `sensor.cam10_last_seen_seconds` | |
| `sensor.cam11_last_seen_seconds` | |
| `sensor.cam14_last_seen_seconds` | |
| `sensor.cam15_last_seen_seconds` | |

### Input Helpers (security_helpers.yaml)

| Entity | Type | Max | Purpose |
|--------|------|-----|---------|
| `input_boolean.security_system_enabled` | boolean | — | Master on/off switch |
| `input_boolean.unknown_entry_event` | boolean | — | Unknown entry flag |
| `input_boolean.security_event_active` | boolean | — | Active event flag (write-back from automations) |
| `input_boolean.security_alert_active` | boolean | — | Alert active flag (consumed by alerts domain) |
| `input_number.perimeter_open_escalation_minutes` | number | 1–60 | Escalation timeout |
| `input_number.house_entry_escalation_minutes` | number | 1–60 | Escalation timeout |
| `input_number.door_warning_escalation_minutes` | number | 1–240 | Door alert escalation |
| `input_datetime.last_security_event` | datetime | — | Cooldown reference |
| `input_datetime.last_intruder_event` | datetime | — | Cooldown reference |
| `input_datetime.last_visitor_event` | datetime | — | Cooldown reference |
| `input_datetime.security_event_start` | datetime | — | Event lifecycle tracking |
| `input_text.security_event` | text | 255 | Last event description |
| `input_text.security_last_motion_camera` | text | 255 | Last camera entity_id |
| `input_text.security_last_motion_image` | text | 255 | Path to latest snapshot |
| `input_text.door_alert_last_open` | text | 255 | Alerts domain helper |
| `input_text.security_motion_sequence` | text | 255 | Camera ID path (append-only, capped 5) |
| `input_text.security_last_path` | text | 255 | Zone name path (append-only, capped 5) |
| `input_text.security_event_images` | text | 255 | Image buffer, 3 entries max |
| `input_text.security_event_active_id` | text | 50 | Active event timestamp ID |
| `input_text.security_event_session` | text | 255 | Pipe-delimited session (overflows) |
| `input_text.cam01_street_driveway_images` | text | 255 | Per-cam image buffer |
| `input_text.cam04_car_port_front_images` | text | 255 | Per-cam image buffer |
| `input_text.cam05_front_driveway_images` | text | 255 | Per-cam image buffer |
| `input_text.cam06_front_entrance_images` | text | 255 | Per-cam image buffer |
| `input_text.cam07_front_kitchen_images` | text | 255 | Per-cam image buffer |
| `input_text.cam09_back_bedroom_images` | text | 255 | Per-cam image buffer |
| `input_text.cam10_pool_bar_motion_images` | text | 255 | Per-cam image buffer — **naming has extra `_motion_`** |
| `input_text.cam11_back_pond_images` | text | 255 | Per-cam image buffer |
| `input_text.cam14_lounge_images` | text | 255 | Per-cam image buffer |
| `input_text.cam15_passage_images` | text | 255 | Per-cam image buffer |
| `input_text.cam01_street_driveway_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam04_car_port_front_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam05_front_driveway_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam06_front_entrance_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam07_front_kitchen_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam09_back_bedroom_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam10_pool_bar_motion_history` | text | 255 | Per-cam snapshot history — **naming has extra `_motion_`** |
| `input_text.cam11_back_pond_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam14_lounge_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam15_passage_history` | text | 255 | Per-cam snapshot history |

### Condition / Context Sensors (security_core.yaml)

| Entity | Purpose | Note |
|--------|---------|------|
| `binary_sensor.boundary_permissive_window` | True during maid/guest window | **BROKEN** — depends on missing input_datetime.low_trust_start/end |
| `binary_sensor.security_visibility_poor` | Weather poor visibility | |
| `binary_sensor.security_weather_low_light` | Weather low light | |
| `binary_sensor.security_lighting_required` | Lighting should be on | |
| `binary_sensor.security_lighting_allowed` | Lighting permitted (night/bad weather) | **BROKEN** — uses wrong entity IDs (see Issue #4) |
| `binary_sensor.security_lighting_suppressed` | Guest/entertaining mode active | |
| `binary_sensor.security_low_trust_active` | Maid/gardener/contractor present | |

---

## Section 4: Data Flow Map

### Snapshot Capture Flow (Currently Broken — Dual Path)

```
binary_sensor.camXX_motion_valid → on
         │
         ├── [Path A] security_capture_each_camera_motion
         │   mode: parallel (fires for ALL cameras simultaneously)
         │   1s delay
         │   camera.snapshot → /config/www/cam05_front_driveway_TIMESTAMP.jpg
         │   → input_text.cam05_front_driveway_history (appends "cam05_TIMESTAMP")
         │
         └── [Path B] sensor.security_trigger_camera changes
             (picks BEST active camera by priority+recency)
             │
             └── security_capture_best_snapshot
                 mode: restart
                 2s delay
                 camera.snapshot → /config/www/security_cam05_front_driveway_TIMESTAMP.jpg  ← ORPHAN (nothing reads this)
                 camera.snapshot → /config/www/security_latest.jpg  ← global latest
                 → input_text.security_last_motion_image = "/local/security_latest.jpg?v=TIMESTAMP"
                 → input_text.cam05_front_driveway_history (appends "cam05_TIMESTAMP")  ← DUPLICATE write to same entity
                 → input_text.security_event_images (3-entry rolling buffer)
                 → input_text.security_event_session (session entries: "cam05,/local/cam05_TIMESTAMP.jpg,HH:MM:SS")
                 → input_text.security_last_motion_camera = "camera.cam05_front_driveway"
                 → input_text.security_motion_sequence (cam ID path, capped 5)
```

**Result:** Two snapshot files per event. `security_cam*` files are unreferenced orphans.
History entities receive duplicate writes. `security_event_session` stores paths to files
created by Path A (`cam05_TIMESTAMP.jpg`), NOT Path B (`security_cam05_TIMESTAMP.jpg`).

### Notification Flow

```
Motion event → zone aggregation → security_correlation sensor
                                           │
              ┌────────────────────────────┼──────────────────────────┐
              ▼                            ▼                          ▼
  security_grounds_motion        security_rear_grounds_motion   security_house_motion
  (trigger: correlation==intruder) (trigger: correlation==intruder)  (trigger: correlation==intruder)
         │                                 │                          │
         ALL THREE fire simultaneously → TRIPLE NOTIFICATIONS for same event
         │
         └── script.notify_security_event (severity from sensor.security_threat_level)
              → notify.STD_Information / STD_Warning / STD_Critical
              → notify.telegram_bot_5527

OR via:
  security_event_router (trigger: zone sensors + gate)
    → classifies by threat level + classification
    → script.notify_security_event

BOTH PATHS CAN FIRE FOR SAME EVENT → duplicate notifications possible.
```

### Event Lifecycle

```
sensor.security_trigger_camera changes (any camera fires)
    │
    ▼
security_event_start
  condition: security_event_active_id is empty
  → sets input_text.security_event_active_id = timestamp
  → sets input_datetime.security_event_start = now()

[event in progress — snapshots, notifications fire]

sensor.security_threat_level == "low" for 2 minutes
    │
    ▼
security_event_end
  → clears input_text.security_event_active_id = ""
  → clears input_text.security_event_session = ""
```

---

## Section 5: Cross-Domain Interface

### What Security CONSUMES (inputs from other domains)

| From Domain | Entity | Used For |
|-------------|--------|---------|
| Presence | `binary_sensor.anyone_connected_home` | Classification: intruder vs family |
| Presence | `binary_sensor.low_trust_present` | Suppress intruder alerts during staff hours |
| Presence | `binary_sensor.staff_on_site` | Trust filtering (maid/gardener) |
| Presence | `input_boolean.guest_mode` | Suppress alerts, permissive window |
| Presence | `input_boolean.holiday_mode` | Escalate to intruder_high (but no automation for this state) |
| Presence | `input_boolean.entertaining_mode` | Ignore grounds motion |
| Presence | `input_boolean.contractor_on_site` | Trust filtering |
| Presence | `input_boolean.arrival_detected` | Written TO by security (see outputs) |
| Context | `binary_sensor.security_night_mode` | Threat score +20 at night |
| Context | `binary_sensor.night_confirmed` | Security mode = night |
| Context | `binary_sensor.night_early` | Lighting allowed gate |
| Context | `input_boolean.openweathermap_api_limited` | Weather API health fallback |
| Context | `sensor.weather_api_health` | Weather API health check |
| Context | `input_datetime.low_trust_start` | Boundary permissive window (MISSING) |
| Context | `input_datetime.low_trust_end` | Boundary permissive window (MISSING) |
| Hardware | `binary_sensor.main_gate_sensor` | Arrival/visitor detection, correlation |
| Hardware | `weather.home` | Visibility sensors (MISSING per Watchman) |
| Notifications | `script.notify_security_event` | All outbound notifications |

### What Security PROVIDES (public API to other domains)

| To Domain | Entity | Meaning |
|-----------|--------|---------|
| Lighting | `sensor.security_lighting_intent` | ignore / perimeter / area / full |
| Lighting | `sensor.security_movement_path` | Current active zone |
| Lighting | `binary_sensor.security_lighting_required` | Lighting should be on |
| Lighting | `binary_sensor.security_lighting_allowed` | Lighting permitted (night/weather) |
| Lighting | `sensor.security_threat_level` | Drives lighting reset after event |
| Alerts | `input_boolean.security_alert_active` | Alert active flag |
| Presence | `input_boolean.arrival_detected` | Security writes ON when arrival gate+camera event fires |
| Any | `binary_sensor.security_perimeter_motion` | Someone at the street |
| Any | `binary_sensor.security_grounds_motion` | Someone in the grounds |
| Any | `binary_sensor.security_inside_house_motion` | Someone inside |
| Any | `sensor.security_threat_level` | Current threat level |
| Any | `sensor.security_event_classification` | Event type classification |

---

## Section 6: Known Issues (Prioritised)

---

### ISSUE 1 — Duplicate snapshot files (confirmed)
**Priority: HIGH | Risk to fix: LOW**

**Symptom:** Every motion event creates two snapshot files:
- `security_cam05_front_driveway_TIMESTAMP.jpg` (security_ prefix)
- `cam05_front_driveway_TIMESTAMP.jpg` (no prefix)

Total in www/: 1,178 `security_*` files + 693 plain `camXX_*` files = 1,871 files, no cleanup.

**Root cause:** Two automations both fire on every motion event:
1. `security_capture_each_camera_motion` — triggers on `*_motion_valid → on`, writes `cam05_TIMESTAMP.jpg`
2. `security_capture_best_snapshot` — triggers on `sensor.security_trigger_camera` change, writes `security_cam05_TIMESTAMP.jpg`

The `security_cam*` files are **unreferenced orphans** — nothing reads `security_cam*.jpg` paths.
Session entries and history use the plain `cam*` naming. The `security_` prefix on the saved file
is a code bug (leftover from a prior naming migration).

**Fix:**
Option A (minimal): Remove the `security_` prefix from `filename_cam` in `security_capture_best_snapshot`,
making it `filename_cam: "/config/www/{{ cam_id }}_{{ timestamp }}.jpg"`. Then the file matches
what `security_capture_each_camera_motion` also saves — still two saves, but same name, so second
write overwrites with slightly fresher frame (the +1s benefit).

Option B (correct): Remove `security_capture_best_snapshot` entirely. Let
`security_capture_each_camera_motion` handle all per-camera saves. Add a separate
"best camera" automation that only writes `security_latest.jpg` and updates
`input_text.security_last_motion_image`. Remove the history/session writing from
the best-snapshot automation since `security_capture_each_camera_motion` already
writes history.

**Also needed:** A retention/cleanup script to remove snapshots older than N days.
Current 1,871 files will grow without bound.

---

### ISSUE 2 — Triple notification on `sensor.security_correlation == "intruder"`
**Priority: HIGH | Risk to fix: MEDIUM**

**Symptom:** When grounds motion fires with `correlation = intruder`, three automations
fire simultaneously: `security_grounds_motion`, `security_rear_grounds_motion`, and
`security_house_motion`. All three send separate notifications for the same event.

**Root cause:** All three automations trigger on `sensor.security_correlation to: "intruder"`.
The intruder state doesn't distinguish between grounds and house. There are no
conditions in `security_grounds_motion` or `security_rear_grounds_motion` to separate their
coverage from each other.

**Fix:** Change triggers:
- `security_grounds_motion` → trigger on `binary_sensor.security_grounds_motion`
- `security_rear_grounds_motion` → trigger on `binary_sensor.security_perimeter_rear_motion` (currently always off — deferred)
- `security_house_motion` → trigger on `binary_sensor.security_inside_house_motion`

Alternatively: keep `security_event_router` as the sole notification path (it already
distinguishes by threat level and classification) and remove the redundant individual
automations, or convert them to non-notification actions.

---

### ISSUE 3 — `binary_sensor.boundary_permissive_window` always false
**Priority: HIGH | Risk to fix: LOW**

**Symptom:** The boundary_permissive_window sensor always returns `false` unless
`guest_mode` is on, meaning the maid/staff window is never recognised.

**Root cause:** The sensor in `security_core.yaml:31` reads from
`input_datetime.low_trust_start` and `input_datetime.low_trust_end`, but these are
commented out in `context_presence.yaml:60-67`. Watchman confirms both are missing.

**Fix:** Either:
A) Uncomment `low_trust_start` and `low_trust_end` in `context_presence.yaml`
B) Rewrite `boundary_permissive_window` to use `input_datetime.maid_start`/`maid_end`
   and `input_datetime.gardener_start`/`gardener_end` which DO exist

---

### ISSUE 4 — `binary_sensor.security_lighting_allowed` always false
**Priority: HIGH | Risk to fix: LOW | ✅ FIXED 2026-04-15**

**Symptom:** Security lighting triggers never fire via the allowed-gate because
`security_lighting_allowed` returns false (except at night).

**Root cause:** `security_core.yaml` referenced:
- `binary_sensor.security_poor_visibility` (doesn't exist)
- `binary_sensor.security_low_light_weather` (doesn't exist)

The actual entity IDs are:
- `binary_sensor.security_visibility_poor`
- `binary_sensor.security_weather_low_light`

**Fix:** Two string replacements applied in `security_core.yaml` (D1 — Group D).

---

### ISSUE 5 — `sensor.cam06_entrance_last_event` reads cam09
**Priority: MEDIUM | Risk to fix: LOW | ✅ FIXED 2026-04-15**

**Symptom:** `sensor.cam06_entrance_last_event` (unique_id: cam06_last_event) in
`cameras_processing.yaml` read from `binary_sensor.cam09_back_bedroom_motion_valid`
instead of `binary_sensor.cam06_front_entrance_motion_valid`. Copy-paste error.

**Fix:** Changed `cam09_back_bedroom_motion_valid` → `cam06_front_entrance_motion_valid`
in `cameras_processing.yaml` (D2 — Group D).

---

### ISSUE 6 — cam10 history entity naming mismatch
**Priority: MEDIUM | Risk to fix: LOW | ✅ FIXED 2026-04-15**

**Symptom:** The cam10 snapshot image entity was never updated. Automations write to
`input_text.cam10_pool_bar_images` but the entity was defined as
`input_text.cam10_pool_bar_motion_images` (extra `_motion_` in key).
Note: `cam10_pool_bar_history` was already correctly named.

**Fix:** Renamed `cam10_pool_bar_motion_images` → `cam10_pool_bar_images`
in `security_helpers.yaml` (D3 — Group D).

---

### ISSUE 7 — No `from:` constraints on motion triggers
**Priority: MEDIUM | Risk to fix: LOW**

**Symptom:** Automations trigger on `unavailable → on` transitions (e.g., after HA
restart, NVR reconnect), causing spurious snapshots and notifications.

**Root cause:** None of the motion-trigger automations in `security_automations.yaml`
specify `from: "off"` on their state triggers, violating `CODING_STANDARDS.md Rule 3`.

Affected automations:
- `security_capture_each_camera_motion`
- `security_track_movement_path`
- `security_visitor`
- `security_arrival_detected`
- `security_event_start`

**Fix:** Add `from: "off"` to all motion sensor triggers in these automations.

---

### ISSUE 8 — Trust model disabled — security runs blind
**Priority: MEDIUM | Risk to fix: MEDIUM**  
**Status:** 🔧 Deferred to S2/S3 — NOT a simple uncomment. See note below.

**Symptom:** Security notifications fire even when maid, gardener, or contractor is on
site — no suppression occurs.

**Root cause:** The presence/trust conditions in the intruder detection automations
(`security_grounds_motion`, `security_rear_grounds_motion`, `security_house_motion`)
are all commented out. The comments read `# disable for testing` but were never
re-enabled.

`sensor.security_event_classification` does check `low_trust_present` for
`service_person` classification, but only if the event router fires on that classification
path. The individual motion automations bypass the event router.

**S1 session note (2026-05-17):** Uncommenting the old conditions is NOT the fix.
The commented code references `input_boolean.staff_on_site` (now a deprecated
display-only helper). The proper fix is the S2/S3 classifier rebuild: trust filtering
moves into `sensor.security_correlation` and `security_event_router`, so individual
motion automations become thin triggers only. Uncommenting old code would restore
a broken trust path and is explicitly out of scope for S1.

**Fix (S2/S3):**
Trust filtering to be handled structurally via classifier rebuild — individual
motion automations become thin triggers; `security_event_router` applies trust
gate using `binary_sensor.low_trust_present` before dispatching notifications.

---

### ISSUE 9 — `input_text.security_event_session` overflows 255 chars
**Priority: MEDIUM | Risk to fix: HIGH**

**Symptom:** Event session logs are truncated after ~3 entries. Session format stores
`cam_id,/local/cam_id_TIMESTAMP.jpg,HH:MM:SS|...` per entry. Each entry is ~50 chars.
After 5 entries the 255-char limit is hit.

**Root cause:** `input_text` max is 255 chars. The session is currently capped to 3
entries in the automation (good), but even 3 entries can exceed 255 chars if camera IDs
are long.

**Fix:** Replace `input_text.security_event_session` with a pyscript-managed state
object or split into 3 discrete fixed-size `input_text` entities
(e.g., `security_session_slot_1/2/3`). See CODING_STANDARDS.md Rule 5.

---

### ISSUE 10 — www/ snapshots accumulate without cleanup
**Priority: MEDIUM | Risk to fix: LOW**

**Symptom:** 1,871 snapshot files in `/config/www/`. No retention policy. Disk will fill
over time.

**Root cause:** Neither automation deletes old files. `security_event_end` only clears
`input_text` state, not disk files.

**Fix:** Add a pyscript or shell_command that runs daily to delete snapshot files older
than 7 days from `/config/www/`, excluding `avatars/` and static images.

---

### ISSUE 11 — Duplicate condition check in `security_capture_best_snapshot`
**Priority: LOW | Risk to fix: LOW**

**Symptom:** The same condition appears twice at lines 57–60 of `security_automations.yaml`:
```yaml
- condition: template
  value_template: >
    {{ trigger.to_state.state != trigger.from_state.state }}
- condition: template  # ← exact duplicate
  value_template: >
    {{ trigger.to_state.state != trigger.from_state.state }}
```

**Fix:** Remove one duplicate condition block.

---

### ISSUE 12 — `sensor.security_correlation` has no `unique_id`
**Priority: LOW | Risk to fix: LOW | ✅ FIXED 2026-04-15**

**Symptom:** Could not be managed via UI entity registry.

**Fix:** Added `unique_id: security_correlation` to sensor definition in
`security_logic.yaml` (D4 — Group D).

---

### ISSUE 13 — `sensor.security_intruder_level` has no `unique_id` and is redundant
**Priority: LOW | Risk to fix: LOW**

**Symptom:** Partially duplicates `sensor.security_threat_level` (critical/warning/info/none
vs critical/warning/elevated/low). Nothing appears to consume `security_intruder_level`
in any automation.

**Fix:** Add unique_id, or remove if unused after confirming no dashboard dependency.

---

### ISSUE 14 — `intruder_high` state produced but never consumed
**Priority: LOW | Risk to fix: LOW**

**Symptom:** `sensor.security_correlation` produces `intruder_high` when `holiday_mode`
is on, but no automation has a trigger or condition for this state.

**Fix:** Add a condition branch in `security_event_router` to handle `intruder_high`
with escalated notification (critical severity), OR escalate threat score when holiday_mode
is on (change threat scoring to add bonus points).

---

## Section 7: Active Log Errors

**Note:** `home-assistant.log` is not available in the local git repository (it lives only
on the running HA instance). The following errors are inferred from Watchman report and
static analysis:

### Confirmed Missing Entities (Watchman Report)

```
MISSING: input_datetime.low_trust_start
  Referenced in: packages/security/security_core.yaml:31
  Effect: binary_sensor.boundary_permissive_window always returns false

MISSING: input_datetime.low_trust_end
  Referenced in: packages/security/security_core.yaml:31
  Effect: same as above

✅ FIXED 2026-05-08: binary_sensor.security_perimeter_rear_motion
  Was: referenced cam03_rear_perimeter_motion_valid (never existed) → always off
  Now: references ipcam05_back_boundary_motion_valid — first active rear perimeter sensor

MISSING: weather.home
  Referenced in: packages/security/security_core.yaml:76,82
  Effect: security_visibility_poor / security_weather_low_light always return 
          their false-branch (weather not in list)

MISSING: binary_sensor.security_visibility_poor (in Watchman)
  Actually defined as binary_sensor.security_visibility_poor — entity exists.
  But security_core.yaml:111 references binary_sensor.security_poor_visibility
  which does NOT exist — confirmed by Watchman.

MISSING: binary_sensor.security_poor_visibility (Watchman confirms)
  Referenced in: packages/security/security_core.yaml:111
  
MISSING: binary_sensor.security_low_light_weather (Watchman confirms)
  Referenced in: packages/security/security_core.yaml:111

DISABLED: sensor.db2c_ezviz_doorbell_last_alarm_picture_url
  Referenced in: packages/security/cameras_processing.yaml:235
  Effect: sensor.doorbell_last_image_safe returns '' always

MISSING IN DASHBOARD: sensor.cam01_last_seen_seconds (lovelace:2685)
MISSING IN DASHBOARD: sensor.cam05_last_seen_seconds (lovelace:2717)
MISSING IN DASHBOARD: sensor.cam06_last_seen_seconds (lovelace:2701)
MISSING IN DASHBOARD: sensor.cam11_last_seen_seconds (lovelace:2733)
  These entities ARE defined in cameras_processing.yaml.
  Possible cause: trigger-based template failed to load, or unique_id conflict.
```

---

## Section 8: Optimisation Recommendations

These work but could be improved. Do NOT mix with the bugs above.

### 8.1 — `sensor.security_last_active_camera` only covers 5 cameras

Currently tracks cam01, cam05, cam06, cam07, cam11. Does not include cam04, cam09, cam10,
cam14, cam15. Add the missing cameras to the list.

### 8.2 — `security_event_router` reads sensor path, not stored path

`security_event_router` reads `sensor.security_movement_path` (real-time current zone)
rather than `input_text.security_last_path` (accumulated zone history). Notifications
show single zone. Consider using `security_last_path` for richer movement context.

### 8.3 — No Telegram photo attachment in notifications

`notify_security_events.yaml:196-199` has the Telegram photo block commented out.
The image is included in HA mobile app notifications but not Telegram. Uncomment when
Telegram file URL approach is confirmed working.

### 8.4 — `sensor.security_trigger_camera` camera priority list excludes cam14/cam15

The priority list in `security_logic.yaml:72-82` omits cam14 and cam15 (inside house).
These would only fire during a house intrusion — highest-priority event — and should be
added at the TOP of the priority list.

### 8.5 — `security_capture_best_snapshot` mode: restart loses frames

With `mode: restart`, if two cameras trigger rapidly, the second restart cancels the
first, potentially missing the initial frame. Consider `mode: queued` with a max_runs
limit or `mode: parallel` with a per-camera semaphore.

### 8.6 — Movement path sensors are redundant

Three parallel path tracking mechanisms exist:
1. `sensor.security_movement_path` — real-time, single current zone
2. `input_text.security_last_path` — zone history (zone names), capped 5
3. `input_text.security_motion_sequence` — camera ID history, capped 5

Items 2 and 3 are nearly identical in purpose. Consolidate to one or clearly assign
different roles (one for UI display, one for automation conditions).

### 8.7 — EZVIZ doorbell disabled

`sensor.db2c_ezviz_doorbell_last_alarm_picture_url` is disabled per Watchman. If the
EZVIZ doorbell is still in use, re-enable the integration. If removed, delete
`sensor.doorbell_last_image_safe` from cameras_processing.yaml.

---

## Section 9: Implementation Checklist

Work in this order — later items depend on earlier ones.

```
SPRINT 1 — Fix Silent Failures (no risk, immediate value)
[✅] Fix Issue 4: two entity ID typos in security_core.yaml — DONE 2026-04-15
      binary_sensor.security_poor_visibility → binary_sensor.security_visibility_poor
      binary_sensor.security_low_light_weather → binary_sensor.security_weather_low_light
[✅] Fix Issue 5: cam06 last event reads cam09 — DONE 2026-04-15
[✅] Fix Issue 6: rename cam10 helper keys — DONE 2026-04-15
                  cam10_pool_bar_motion_images → cam10_pool_bar_images
[✅] Fix Issue 12: add unique_id to sensor.security_correlation — DONE 2026-04-15
[✅] D5: remove presence_test_arrival test automation from production — DONE 2026-04-15
[ ] Fix Issue 11: remove duplicate condition block in security_capture_best_snapshot
[ ] Fix Issue 3: restore low_trust_start/end OR rewrite boundary_permissive_window
                 to use maid_start/maid_end + gardener_start/gardener_end

SPRINT 2 — Snapshot Deduplication (Issue 1)
[ ] Decide on Option A or Option B (see Issue 1)
[ ] Implement chosen option
[ ] Add www/ retention cleanup (daily cron via pyscript or shell_command)
[ ] Validate no dashboard cards break

SPRINT 3 — Notification Architecture (Issues 2 + 8)
[ ] Fix Issue 2: change triggers on grounds/rear/house automations to use
                 binary_sensor.security_{zone}_motion instead of correlation sensor
[ ] Fix Issue 7: add from: "off" to all motion triggers
[ ] Fix Issue 8: re-enable low_trust_present and guest_mode conditions
                 after trigger changes are validated

SPRINT 4 — Session State Machine (Issue 9)
[ ] Design pyscript state object for event session OR 3-slot fixed entities
[ ] Implement replacement for input_text.security_event_session
[ ] Update dashboard cards that render the session timeline

SPRINT 5 — Optimisations (Section 8)
[ ] Add cam14/cam15 to security_trigger_camera priority list (top)
[ ] Add missing cameras to security_last_active_camera sensor
[ ] Add intruder_high handling in event router or threat score
[ ] Resolve/remove sensor.security_intruder_level if unused

SPRINT 6 — 2026-05-10/11/12/14 changes (done)
[✅] Warning branch made zone-aware: perimeter-only events say "Perimeter Activity", not "Intruder on Property"
[✅] Warning/critical messages check actual occupancy instead of hardcoding "nobody home"
[✅] Camera health sensor made trigger-based (every 5 min) — fixes missed staleness alerts
[✅] Camera health now monitors ipcam01–05 availability + staleness (last_seen_seconds)
[✅] Camera health NVR indoor threshold: 86400 → 28800 (8h) for cam14/cam15
[✅] Smart Events migration Phase 3b complete: all ipcams motiondetection disabled, using fielddetection/linedetection/regionentrance
[✅] inside_cameras_armed split: lounge (cam14+cam05) vs passage (cam15) — separate booleans
[✅] inside_cameras_passage_armed: arms only when nobody home; passage is secure zone at night
[✅] Inside camera arming: 23:00 trigger + all_family_in_bedrooms (all 4 family) for 5min
[✅] binary_sensor.all_family_in_bedrooms: checks ALL group.family_ap_locations in "Bedroom Zone"
[✅] security_inside_house_motion zone gated on arming booleans (prevents false critical from family movement)
[✅] input_boolean.security_event_lights: suppress event-triggered lights (notifications still fire)
[✅] security_lighting_reset gated on security_lighting_required=off — boundary lights no longer killed at night
[✅] scene_kids_bedtime: back_house_security removed (stays on until full bedtime)
[✅] Confidence tiers: NVR front cameras (cam04/cam07) demoted to low when alone; AcuSense ai_front = medium alone
[✅] cam04 delay_on: 1s → 4s; cam07 delay_on: 2s → 6s (filters washing line / leaf flickers)
[✅] NVR front cameras restored to medium confidence after delay_on filtering
[✅] security_capture_best_snapshot: mode restart → mode single (was spinning Pi CPU to 91%)
[✅] Recorder: 30 new exclusions for security/camera transient sensors
[✅] last_seen_seconds trigger: every 1 min → every 5 min
[✅] zone_label derived from security_trigger_camera (not stale security_last_path)
[✅] zone_label substring bug fixed: ipcam before cam in elif chain (ipcam04 was matching cam04)
[✅] Duplicate "Camera:" line removed from messages (notify script already adds from source)
[✅] Visitor/arrival staleness filter: 10s → 30s (Pi CPU load caused queue delay > 10s)
[✅] Arrival detection: wait_for_trigger ipcam03 up to 60s after gate opens (was checking before person entered)
[✅] Arrival image: 3s delay after wait_for_trigger so snapshot pipeline completes before notification fires
[✅] Threat rule 3: perimeter + confirmed_human + evening now requires nobody_home for CRITICAL
      — family arriving home no longer triggers CRITICAL (falls to WARNING via rule 6 instead)
[✅] Visitor/arrival staleness filter: 10s → 30s (Pi queue delay was causing missed notifications)
```

---

## Section 10: Design Decisions (Locked)

The following MUST NOT change without full review — dashboards and automations in other
domains depend on these exact values.

### 10.1 — Camera entity naming (LOCKED)

```
camera.camXX_location_description
binary_sensor.camXX_location_motiondetection   ← NVR raw
binary_sensor.camXX_location_motion_valid      ← debounced
```

Renaming any camera entity breaks: snapshot references, history entity keys, UI cards,
the trigger camera priority list, and presence boundary automations.

### 10.2 — Core sensor entity IDs (LOCKED)

```
sensor.security_trigger_camera
sensor.security_event_classification
sensor.security_threat_level
sensor.security_threat_score
sensor.security_movement_confidence
input_text.security_event_session
input_text.security_last_path
input_text.security_last_motion_camera
input_text.security_last_motion_image
input_datetime.security_event_start
input_datetime.last_security_event
input_datetime.last_intruder_event
input_datetime.last_visitor_event
input_boolean.security_alert_active
input_boolean.security_system_enabled
```

### 10.3 — Snapshot path convention (LOCKED for UI)

Dashboard cards reference `/local/security_latest.jpg?v=TIMESTAMP` for the global latest.
This MUST remain. Per-camera references use `cam05_front_driveway_TIMESTAMP` image names
(without `security_` prefix). The `security_cam*` prefixed files are unreferenced orphans
and can be cleaned up.

### 10.4 — Notification script interface (LOCKED)

All security notifications go through `script.notify_security_event` with fields:
`severity`, `title`, `message`, `image`, `source`, `gate_control`. Do not bypass this
with direct `notify.*` calls.

### 10.5 — Zone naming (LOCKED)

```
Perimeter  = outside property boundary (cam01 + future cam03)
Grounds    = inside property, outside house (cam04–cam11)
Inside     = inside house (cam14, cam15)
```

### 10.6 — www/ snapshot serving path

`/config/www/` → served as `/local/` in HA. All snapshot paths stored in input_text
entities use the `/local/` prefix. Do not change.

### 10.7 — Warning threshold: 75% (LOCKED 2026-04-15)

`sensor.security_threat_level` warning fires at score ≥ 75 (not 50).
Rationale: analog NVR cameras have no AI classification — dogs, rain, wind produce
frequent low-confidence motion. The 75% floor requires either night + grounds + medium
confidence, or perimeter + grounds + nobody + high confidence. Reducing this threshold
produces notification spam from the analog system.
Do not lower below 75 without confirming AI cameras are in use.

### 10.8 — Event router cooldown: 300s (LOCKED 2026-04-15)

`security_event_router` enforces a 5-minute minimum between warning/elevated notifications
via `input_datetime.last_security_event`. Critical events (score ≥ 80, inside house) bypass
the cooldown to ensure immediate notification.
Visitor/arrival info events are also outside the cooldown.
Architecture preserved for new AI camera integration — cooldown is in the router only,
not in the classification sensors, so new camera triggers flow through unchanged.

### 10.9 (new) — ipcam02 Smart Event incompatibility with hikvision_next

**Camera:** IPCam02-Street-Driveway-Down, model DS-2CD2047G3-LI2UY, firmware V5.8.13 branch B-R-H13U-0  
**Compared to ipcam01:** DS-2CD2087G2H-LI, firmware V5.7.19 branch B-R-G5-0

**Symptom:** hikvision_next only discovers `scenechangedetection` for ipcam02.
`fielddetection`, `linedetection`, `regionentrance`, `regionexiting` are never created
even after full device delete + re-add + HA restart.

**Root cause:** Firmware branch H13U (V5.8.13) presents Smart Events via a different
ISAPI structure than branch G5 (V5.7.19). hikvision_next's ISAPI query finds Smart Events
for the G5 branch but not H13U. All other cameras are G5 firmware; ipcam02 is H13U.

**Troubleshooting attempted (2026-05-11):**
- Smart Event rules configured (Intrusion, Line Crossing, Region Entrance, Region Exiting) ✅
- Notify Surveillance Center ticked on all rules ✅
- Alarm server: 10.10.1.5:8123/api/hikvision — alarm server TEST passes ✅
- Full device delete + HA restart + re-add → still only scenechangedetection discovered ✅
- Admin user "Remote: Notify Surveillance Center" permission ticked → no change ✅
- New user created with Notify Surveillance Center permission → still no events reach HA ✅
- Physical walk-in-front test → zero snapshot files created, zero entity state changes ✅
- Bonjour enabled on ipcam02 → no change ✅

**Conclusion:** Not a permissions or config issue. Firmware branch H13U (V5.8.13) fundamentally
incompatible with this version of hikvision_next. Installer contacted for assistance 2026-05-11.

**Current HA state:** `ipcam02_street_driveway_down_motion_valid` always reads `off` (references
fielddetection/linedetection/regionentrance — none exist). Camera entity shows idle/connected.
ipcam02 contributes ZERO signal to the security system until resolved.

**Pending installer:** May need firmware rollback to V5.7.19 G5 branch, factory reset + re-provision,
or replacement camera. Do not waste further time on HA-side config changes until installer confirms
what's different between ipcam01 and ipcam02 at hardware/firmware level.

### 10.10 — Pool alarm tiered threat gate (UPDATED 2026-05-10)

`security_pool_alarm_trigger` fires ipcam04's physical alarm output for rear property events.

**Time window:** Arms from 20:00 (`security_night_mode` ON OR `now().hour >= 20`).
Old behaviour was night_mode only (~22:30+) — missed the 20:00–22:30 early-evening window.

**Threat logic (tiered):**
| Threat | Family home | night_mode | Fires? |
|--------|-------------|-----------|--------|
| critical | any | any | ✅ always |
| warning | away | any | ✅ nobody home |
| warning | home | OFF (20:00–22:30) | ✅ early deterrent |
| warning | home | ON (asleep) | ❌ suppressed |

Rationale: critical always justifies waking the house; warning while asleep is likely dogs/wind
on analog NVR (no AI) — alarm would cause unnecessary disturbance.

Also gated by: `security_dogs_out` OFF + `guest_mode` OFF + 5-min cooldown on `last_intruder_event`.

---

*Audit completed: 2026-04-13*  
*Audited by: Claude Code deep audit*  
*Updated 2026-04-15: Sprint 1 bugs resolved (D1/D2/D3/D4/D5 — Issues 4/5/6/12 fixed)*  
*Updated 2026-04-15: Classification audit — warning threshold 75%, 5-min cooldown,*
*  cam04 carport added to movement path, event router fires on to:"on" only.*
*Updated 2026-05-07: cam01/cam10 deprecated; cam05 moved inside garage; ipcam01/02 wired in*  
*Updated 2026-05-08: cam06 removed (uninstalled); all 5 IP cameras wired and renamed to ipcam01–ipcam05*
*  (was mixed camip/ipcam — single-pass rename across 7 config files); perimeter_rear fixed;*
*  elevated branch added to event_router (Root Cause 1); cam07 added to confidence front tier*
*  (Root Cause 2); AcuSense entrance/exit debounce sensors added; visitor logic corrected*
*  (street cameras only — ipcam03 = driveway = inside gate = arrival, not visitor);*
*  cam05 garage moved to grounds_front (garage ≠ living space — was causing false critical_intrusion);*
*  notification spam fixed (event_router sole notifier); critical message now contextual.*
*Updated 2026-05-10: Warning branch in event_router made context-aware — title/message now distinguish*
*  perimeter-only events ("⚠️ Perimeter Activity" / "outside boundary") from grounds events, and*
*  check actual occupancy (nobody_home vs at_night) instead of hardcoded "nobody home" string.*
*  Camera health alert extended to monitor ipcam01–05 availability — IP cameras have no videoloss*
*  sensor so camera entity state is checked instead (unavailable = offline fault).*
*Updated 2026-05-10: security_pool_alarm_trigger tiered threat gate — arms from 20:00. Section 10.9 added.*
*Updated 2026-05-10/11: Sprint 6 complete — see checklist. Key changes: warning branch zone-aware;*
*  camera health trigger-based + ipcam staleness; inside cam arming split (lounge/passage);*
*  all_family_in_bedrooms sensor; security_event_lights toggle; lighting reset night guard;*
*  kids bedtime front-only; NVR confidence tiers + delay_on; capture mode:single; recorder exclusions;*
*  arrival wait_for_trigger; staleness filter 30s; zone_label from trigger cam; substring bug fixed.*
*  ipcam02: DS-2CD2047G3-LI2UY firmware V5.8.13 H13U branch incompatible with hikvision_next Smart Events.*
*  Remote:Notify permission missing on admin user — fix pending; motiondetection fallback TBD.*
*Updated 2026-05-12/14: Perimeter threat rule 3 — nobody gate added. Arrival image 3s delay.*
*  cam_label removed from variables; logbook.log elevated branch updated to use cam.split('.')[-1].*
*  Overview dashboard Firefox crash: html-template-card (load shedding 48 timeslots) + power-flow-card-plus*
*  + pulseCritical infinite CSS animation combined to freeze Firefox render thread. Fixed: animations*
*  changed to 3-cycle (was infinite); html-template-card and power-flow disabled pending investigation.*
*DESIGN PENDING: Arrival/exit/visitor flow redesign — see Section 11 below.*
*Next: Arrival/visitor redesign (Section 11); restore disabled overview cards; ipcam02 installer*

---

## Section 11: Pending Design — Arrival / Exit / Visitor Flow Redesign

> **Status:** Design phase. Not yet implemented. Plan in Claude chat, then implement here.
> **Triggered by:** Repeated false CRITICAL alerts for family arriving home; missed arrivals;
> no departure detection; presence not integrated into threat classification.

### Problem Statement

The current system treats all gate/street activity as a security event first, presence second.
A family member pulling up to their own gate at 19:00 triggers CRITICAL INTRUDER because:
- ipcam01 fires regionentrance (→ confirmed_human = true)
- evening = true, nobody_home = true (family just arrived, not yet home)
- Rule 3: perim + evening + confirmed_human + nobody → CRITICAL

The correct classification should be: gate + AP approaching = ARRIVAL, not INTRUDER.

### Desired Classification Table

| Scenario | Key Signals | Target Classification | Notification |
|---|---|---|---|
| Family arriving | Gate opens + ipcam03 fires + AP connects ≤5min | **Arrival** | 🏠 info, no alert |
| Family departing | Gate opens + family AP disconnecting | **Departure** | 👋 info, no alert |
| Known visitor | Street cam + gate closed + no family AP | **Visitor** | 👤 warning + gate button |
| Delivery/service | Street cam + no AP + daytime | **Visitor** | 👤 info + gate button |
| Intruder grounds | Grounds motion + no AP + no gate + night | **Threat** | 🚨 critical/warning |
| Intruder perimeter | Street cam + no AP + no gate + confirmed_human + nobody_home | **Perimeter Threat** | ⚠️ warning |

### Design Principles

1. **Presence-first:** check `anyone_connected_home` and `group.family_ap_locations` BEFORE
   classifying gate/perimeter events as threats.
2. **Gate = trusted action:** if the family opened the gate (AP within range), treat as
   arrival/departure, not a security event.
3. **Visitor vs intruder:** street cam without gate open = visitor (offer gate control).
   Only escalate if grounds penetration confirmed without gate event.
4. **Departure detection:** gate opens + AP signal weakening/disconnecting = departure;
   useful for presence engine and for not firing arrival false positives on the way out.

### Entities Available for Redesign

```
binary_sensor.anyone_connected_home         ← anyone on home AP
group.family_ap_locations                    ← per-person AP zone (Home/Away/Bedroom Zone/etc)
binary_sensor.main_gate_sensor              ← gate open/closed
binary_sensor.ipcam01_street_driveway_up_*  ← street upper approach
binary_sensor.ipcam02_street_driveway_down_* ← street lower (currently non-functional)
binary_sensor.ipcam03_driveway_entrance_valid ← confirmed inside gate (AI-filtered)
binary_sensor.ipcam03_driveway_exit_valid    ← confirmed departure from gate
sensor.security_threat_level               ← current threat state
input_datetime.last_visitor_event          ← cooldown reference
input_datetime.last_security_event         ← cooldown reference
```

### Known Edge Cases to Resolve in Design

- Family member arrives home while others already home (anyone_connected_home = on throughout)
- Gate opens but nobody enters (opened remotely for a delivery, ipcam03 may not fire)
- Guest/visitor arrives when family is home (gate_sensor + no AP = visitor, not arrival)
- Staff/domestic arrives (low_trust_present flow — should NOT trigger arrival)
- False gate open (sensor bounce, wind) with no camera corroboration
- Multiple back-to-back arrivals within cooldown window
- Departure immediately followed by arrival (school run — out and back within minutes)

### Current Partial Fixes in Place (2026-05-14)

- Threat rule 3: perimeter critical now requires `nobody_home` ← reduces false CRITICALs
- Arrival: `wait_for_trigger` ipcam03 up to 60s after gate ← catches most arrivals
- Visitor: ipcam01/02 + gate closed → visitor notification + gate button
- Visitor staleness filter: 30s ← handles loaded Pi queue delay

### Next Steps

1. Design full state machine in Claude chat (use this section as the brief)
2. Agree on edge case handling
3. Implement in: `security_logic.yaml` (threat level), `security_automations.yaml`
   (visitor/arrival/departure automations), `security_core.yaml` (new sensors if needed)
4. Test with controlled gate open/close scenarios before enabling at night
