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

### Hardware Summary — Camera Fleet (verified 2026-05-17 — 7 NVR + 5 IP)

#### Active NVR cameras (Hikvision DS-7116HGHI-F1 analog, no AI, motion-only)

No AI detection, no per-object classification. Motion signal is pixel-diff only.
go2rtc timeout instability. Being progressively replaced by AcuSense IP cameras.

| Entity | Friendly | Zone |
|--------|----------|------|
| `cam04_car_port_front` | Carport / front-left | grounds_front |
| `cam05_inside_garage` | Garage interior | **inside_garage** (renamed from cam05_front_driveway 2026-05-17) |
| `cam07_front_kitchen` | Side entry / kitchen window | grounds_front |
| `cam09_back_bedroom` | Back / bedroom side | grounds_rear |
| `cam12_back_pond` | Back pond | grounds_rear |
| `cam14_lounge` | Lounge (TV Room) | **inside_main** |
| `cam15_passage` | Bedroom passage | **inside_bedrooms** |

**NVR placeholder slots — removed 2026-05-17:** The 16-channel DVR previously had unused channels appearing in HA as `cam02-future`, `cam03-future`, `cam11-future`, `cam13-future`, `cam16-future` (NO VIDEO devices). These were removed 2026-05-17. NVR roadmap: the NVR itself will eventually be retired in favor of an all-IP camera fleet. No additional analog cameras will be added to this NVR — any new cameras going forward will be IP (ipcam06+).

**Deprecated / removed (DO NOT reintroduce):**

| Entity | Replaced by | Removed |
|--------|-------------|---------|
| `cam01_street_driveway` | ipcam01 + ipcam02 | Deprecated 2026-05-07; entities purged 2026-05-17 |
| `cam06_front_entrance` | ipcam03 (single turret over garage approach) | Physically uninstalled 2026-05-08; entities purged 2026-05-17 |
| `cam10_pool_bar` | ipcam04 | Deprecated 2026-05-07; entities purged 2026-05-17 |

#### Garage zone: camera vs driveway

- **`cam05_inside_garage`** — garage interior camera, mounted inside the garage since 2026-05-07. Watches garage interior activity. Feeds `security_inside_garage_motion`.
- **`ipcam03_driveway`** — driveway approach camera (AI). Watches the driveway between the gate and the garage. Feeds `security_grounds_motion` via `grounds_front`.
- These are distinct signals. cam05 was incorrectly in `grounds_front` 2026-05-08→2026-05-17 as a workaround for the old single-zone inside model. With S2 zone arming gates, the workaround is gone.
- NVR entity `cam05_front_driveway_motiondetection` keeps its hardware name (integration-provided). Our template sensor reads it as `cam05_inside_garage_motion_valid`.
- **UI cosmetic:** Device area in HA still reads "Driveway" — worth updating via Settings → Devices when convenient.

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

#### ipcam01 Smart Event Zone Configuration (verified 2026-05-20)

Accessed via http://10.10.1.11. Current state (observed from camera web UI):

| Rule | Current config | Issue | Recommendation |
|------|---------------|-------|----------------|
| Region Entrance Detection | Enabled, Rule 1. Human+Vehicle. Zone covers right side (gate area). Target Validity: Higher | Zone is large enough that any vehicle near gate triggers entrance. | Acceptable — keep as visitor approach signal |
| Region Exiting Detection | Enabled, Rule 1. Human+Vehicle. **Same zone position as entrance.** | Identical zone → same vehicle triggers both entrance AND exiting | **Disable or move zone** to cover only vehicles retreating FROM gate back down the street. If not needed, disable (currently unused in HA arrival/departure logic) |
| Line Crossing | Enabled, Rule 1. Human+Vehicle. **Direction: A↔B (bidirectional)** | Fires for movement in either direction — no directional signal | Change to **A→B** where B is the gate side. Only fires for objects approaching gate from street. |
| Intrusion (Field) | Enabled, Human+Vehicle, Threshold 2s | Fine for general grounds motion | No change |

#### ipcam03 Smart Event Zone Configuration (verified 2026-05-20)

Accessed via http://10.10.1.12. **Root cause of BUG-S29 identified here.**

| Rule | Current config | Issue | Recommendation |
|------|---------------|-------|----------------|
| Intrusion (Field) | Enabled, **Human only (Vehicle unchecked)**, Threshold 2s | Misses vehicle-only events (car with tinted windows or no visible driver) | **Add Vehicle** to detection targets |
| Line Crossing | Enabled, Human+Vehicle. **Direction: A↔B (bidirectional)** | Fires in both directions → HA `linedetection` fires for both arrival and departure | Change to **A→B** (one direction). Recommend: A = garage side → B = gate side = departure direction; or use two separate rules if hikvision_next exposes them separately |
| Region Entrance | Enabled, Human+Vehicle. **Red zone positioned mid-driveway** | Zone overlaps with Region Exiting zone → a car traversing the driveway in either direction crosses BOTH zones | **Reposition to gate-mouth only** — narrow strip at the top of frame right where the gate opens. Only a car coming THROUGH the gate enters this zone |
| Region Exiting | Enabled, Human+Vehicle. **Red zone overlaps entrance zone position** | Same overlap problem | **Reposition to mid-driveway** — between garage and gate mouth, lower 40% of frame. Only a departing car reversing from garage toward gate crosses this zone. Must NOT overlap entrance zone |

**Target geometry (no-overlap zones):**
```
ipcam03 field of view (gate at top, garage at bottom):
┌─────────────────────────────┐
│  [GATE MOUTH]               │
│  ┌──────────┐               │  ← Region Entrance: narrow strip at gate opening
│  │ ENTRANCE │               │     Arriving car crosses this once coming through gate
│  └──────────┘               │     Departing car is already past this, won't re-enter
│                             │
│     [DRIVEWAY]              │
│  ┌──────────────────────┐   │  ← Region Exiting: mid-driveway strip
│  │     EXITING          │   │     Departing car crosses this as it reverses toward gate
│  └──────────────────────┘   │     Arriving car drives forward, doesn't double back
│                             │
│  [GARAGE / HOUSE END]       │
└─────────────────────────────┘
```

**CRITICAL: the two zones must not overlap.** With the current config they cover the same area, causing both `regionentrance` and `regionexiting` to fire for any vehicle movement (BUG-S29). This is what caused arrival/departure confusion in notifications.

**HA-side implementation:** `security_gate_vehicle_stage1` merged automation. S9 used 90s AP settle delay; **S9b (2026-05-20) reduced to 5s** now zones are fixed — direction now comes from trigger entity (entrance vs exit) which is reliable with non-overlapping zones. AP state used only as fallback if both sensors somehow fire simultaneously.

**Updated ipcam03 config (user applied 2026-05-20):**

| Rule | Change made | Notes |
|------|------------|-------|
| Region Entrance | Repositioned to gate-mouth area (upper-right of frame, small zone) | ✅ No longer overlapping with exit zone |
| Region Exiting | Repositioned to lower driveway near garage end (larger zone) | ✅ Separate from entrance zone |
| Line Crossing | Kept A↔B — positioned between the two zones | General motion between zones; A↔B correct since hikvision_next fires single entity regardless of direction |
| Intrusion (Field) | Human only, threshold reduced 2s→1s | Vehicle NOT added — entrance/exit already cover vehicles; intrusion = person loitering |

**Updated ipcam01 config (user applied 2026-05-20):**

| Rule | Change made |
|------|------------|
| Region Exiting | Disabled |
| Line Crossing | Changed A↔B → A→B (approach direction only) |

#### Street-Down Camera (ipcam02 / pending ipcam06+) — Smart Event Zone Recommendations

Generic recommendations for any street-facing camera covering the lower driveway approach (same position as ipcam02 but applicable to any replacement). These match the ipcam01 pattern adjusted for the different viewing angle:

| Rule | Recommendation | Rationale |
|------|---------------|-----------|
| Intrusion (Field) | Human only, Threshold 2-3s. Zone: full gate approach area on street | Detects loitering person at gate. Vehicle-only detections not needed — entrance handles that |
| Line Crossing | A→B where B = gate side. Human+Vehicle. | Approach-to-gate direction only; bidirectional creates noise with no HA benefit |
| Region Entrance | Small zone at gate approach point (pavement in front of gate). Human+Vehicle. | Visitor/arrival approach signal. Zone should be narrow enough that only someone standing at the gate triggers it, not passing street traffic |
| Region Exiting | **Disable.** | Unused in HA. A "region exit" on a street camera = someone moving away from the gate. Not a useful signal for arrival/departure logic. Generates noise. |

**HA note (ipcam02):** Currently dead — firmware V5.8.13 H13U incompatible with hikvision_next Smart Event discovery (BUG-S28). These recommendations apply to a replacement or firmware-fixed camera.

#### Hikvision AcuSense — HA Entity Limitations

**Human vs vehicle separation:** AcuSense cameras DO classify at hardware level. However hikvision_next maps one HA entity per event TYPE, not per detection target. Multiple rules for the same event type (e.g., Rule 1 = Human, Rule 2 = Vehicle, both Region Entrance) still produce one `*_regionentrance` entity in HA.

**Current workaround using different event types:**
- `fielddetection` (Intrusion) → Human only → person loitering in zone
- `regionentrance` / `regionexiting` → Human + Vehicle → arrival/departure
- `linedetection` → Human + Vehicle → general crossing

This gives partial separation: `fielddetection` = person-specific signal; `regionentrance` = any vehicle or person.

**Line crossing direction:** A↔B and A→B both produce the same HA `linedetection` entity — no directional value in HA. Use A→B to reduce false triggers at the camera level (fewer events sent to HA), but don't rely on it for HA directional logic.

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
- `binary_sensor.ipcam03_driveway_entrance_valid` — vehicle/person confirmed in driveway intrusion zone (fielddetection). **S11c 2026-05-22:** changed from regionentrance → fielddetection. regionentrance disabled in camera UI. No longer a Stage 1 trigger (replaced by main_gate_sensor). Still feeds confirmed_human in threat_level sensor.
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
| `sensor.security_event_classification` | template sensor | security_logic.yaml | idle, arrival, departure, family_movement, service_person, **perimeter_front**, visitor, perimeter_threat, intruder, critical_intrusion | security_event_router automation (S3) |
| `binary_sensor.security_inside_garage_motion` | template sensor | security_zones.yaml | on/off | security_event_classification |
| `binary_sensor.security_inside_main_motion` | template sensor | security_zones.yaml | on/off | security_event_classification |
| `binary_sensor.security_inside_bedrooms_motion` | template sensor | security_zones.yaml | on/off | security_event_classification |
| `binary_sensor.inside_garage_armed` | template sensor | security_zones.yaml | on/off | security_event_classification |
| `binary_sensor.inside_main_armed` | template sensor | security_zones.yaml | on/off | security_event_classification |
| `binary_sensor.inside_bedrooms_armed` | template sensor | security_zones.yaml | on/off | security_event_classification |
| `binary_sensor.family_arriving` | template sensor | presence_confidence.yaml | on/off | security_event_classification |
| `binary_sensor.family_departing` | template sensor | presence_confidence.yaml | on/off | security_event_classification |
| `binary_sensor.all_family_home` | template sensor | presence_confidence.yaml | on/off | security_event_classification |
| `sensor.security_threat_level` | template sensor | security_logic.yaml | low, elevated, warning, critical | security_event_router, lighting reset, security_event_end |
| `sensor.security_threat_score` | template sensor | security_logic.yaml | 0–100 | security_threat_level |
| `sensor.security_movement_confidence` | template sensor | security_logic.yaml | none, low, medium, high | security_grounds_motion condition, security_correlation |
| `sensor.security_movement_path` | template sensor | security_logic.yaml | street, driveway, front_door, side_entry, rear_left, rear_center, rear_right, none | lighting_security.yaml, security_event_router |
| `sensor.security_correlation` | template sensor (no unique_id!) | security_logic.yaml | family_arrival, visitor, service_visit, intruder, house_intrusion, intruder_high, ignore, none | security_grounds_motion, security_rear_grounds_motion, security_house_motion |
| `sensor.security_intruder_level` | template sensor (no unique_id!) | security_logic.yaml | critical, warning, info, none | Possibly UI only — partially redundant to threat_level |
| `sensor.security_mode` | template sensor | security_core.yaml | away, night, home | notify_security_events.yaml |
| `sensor.security_trust_mode` | template sensor | security_core.yaml | scheduled_staff, detected_staff, normal | UI only |
| `sensor.security_lighting_intent` | template sensor | security_core.yaml | ignore, full, area, perimeter | lighting_security.yaml |
| `sensor.security_last_active_camera` | template sensor | cameras_processing.yaml | ipcam02_street_down, ipcam01_street_up, ipcam03_driveway, ipcam04_pool_bar, ipcam05_rear_boundary, cam04_carport, cam05_garage, cam07_kitchen, cam12_pond | UI |

### Debounce Motion Sensors (per-camera)

| Entity | Type | File | Delay Off | Note |
|--------|------|------|-----------|------|
| `binary_sensor.cam04_car_port_front_motion_valid` | template BS | cameras_processing.yaml | 20s | |
| `binary_sensor.cam05_inside_garage_motion_valid` | template BS | cameras_processing.yaml | 25s | Renamed 2026-05-17 from cam05_front_driveway_motion_valid |
| `binary_sensor.cam07_front_kitchen_motion_valid` | template BS | cameras_processing.yaml | 25s | |
| `binary_sensor.cam09_back_bedroom_motion_valid` | template BS | cameras_processing.yaml | 25s | |
| `binary_sensor.cam12_back_pond_motion_valid` | template BS | cameras_processing.yaml | 30s | |
| `binary_sensor.cam14_lounge_motion_valid` | template BS | cameras_processing.yaml | 15s | |
| `binary_sensor.cam15_passage_motion_valid` | template BS | cameras_processing.yaml | 15s | |
| `binary_sensor.cam01_street_driveway_motion_valid` | ~~template BS~~ | cameras_processing.yaml | — | **DEPRECATED 2026-05-07 — commented out; entity registry orphan** |
| `binary_sensor.cam06_front_entrance_motion_valid` | ~~template BS~~ | cameras_processing.yaml | — | **DEPRECATED 2026-05-08 — commented out; entity registry orphan** |
| `binary_sensor.cam10_pool_bar_motion_valid` | ~~template BS~~ | cameras_processing.yaml | — | **DEPRECATED 2026-05-07 — commented out; entity registry orphan** |

### Zone Aggregation Sensors

| Entity | Type | File | Sources | Note |
|--------|------|------|---------|------|
| `binary_sensor.security_perimeter_front_motion` | template BS | security_zones.yaml | ipcam01 OR ipcam02 | Updated 2026-05-08 |
| `binary_sensor.security_perimeter_rear_motion` | template BS | security_zones.yaml | ipcam05_back_boundary | **Fixed 2026-05-08** — was ALWAYS OFF (cam03 never existed) |
| `binary_sensor.security_perimeter_motion` | template BS | security_zones.yaml | perimeter_front OR perimeter_rear | Covers all 3 street/boundary cameras |
| `binary_sensor.security_grounds_motion` | template BS | security_zones.yaml | expand(grounds_front + grounds_rear groups) | grounds_front: cam04/cam07/ipcam03; grounds_rear: cam09/ipcam04/cam12. cam05 removed 2026-05-17 — inside_garage only. |
| `binary_sensor.security_external_motion` | template BS | security_zones.yaml | perimeter OR grounds | |
| `binary_sensor.security_inside_garage_motion` | template BS | security_zones.yaml | cam05_inside_garage_motion_valid | **Added S2 2026-05-17.** Raw (no arming gate). cam05 = garage interior camera. DSC garage zone sensor hooks in S6+. |
| `binary_sensor.security_inside_main_motion` | template BS | security_zones.yaml | cam14_lounge_motion_valid | **Added S2 2026-05-17.** Raw. DSC: lounge/entrance/sunroom/kitchen zone sensors. |
| `binary_sensor.security_inside_bedrooms_motion` | template BS | security_zones.yaml | cam15_passage_motion_valid | **Added S2 2026-05-17.** Raw. DSC: bedroom zone sensors. |
| `binary_sensor.security_inside_house_motion` | template BS | security_zones.yaml | garage OR main OR bedrooms | **Updated S2:** now raw union of three zone sensors. Backward-compatible. Arming gate moved out to classifier. `inside_cameras_armed` / `_passage_armed` booleans remain for snapshot automation logic only. |
| `binary_sensor.inside_garage_armed` | template BS | security_zones.yaml | `inside_garage_armed_manual` | **Added S2.** DSC-ready stub. |
| `binary_sensor.inside_main_armed` | template BS | security_zones.yaml | `inside_main_armed_manual` | **Added S2.** DSC-ready stub. |
| `binary_sensor.inside_bedrooms_armed` | template BS | security_zones.yaml | `inside_bedrooms_armed_manual` | **Added S2. DESIGNED OFF in stay-mode** — bathroom trips suppressed at night. |

### Per-Camera Timestamp Sensors

| Entity | Unique ID | Note |
|--------|-----------|------|
| `sensor.cam04_carport_last_event` | cam04_last_event | Timestamp device_class |
| `sensor.cam05_garage_last_event` | cam05_last_event | Timestamp device_class |
| `sensor.cam07_kitchen_side_last_event` | cam07_last_event | Timestamp device_class |
| `sensor.cam09_rear_bedroom_last_event` | cam09_last_event | Timestamp device_class |
| `sensor.cam12_back_pond_last_event` | cam12_last_event | Timestamp device_class |
| `sensor.cam01_street_last_event` | cam01_last_event | **ORPHAN** — cam01 deprecated; entity registry only |
| `sensor.cam06_entrance_last_event` | cam06_last_event | **ORPHAN** — cam06 removed 2026-05-08 |
| `sensor.cam10_pool_bar_last_event` | cam10_last_event | **ORPHAN** — cam10 deprecated |

**Note: No cam14, cam15 last event sensors defined.**

### Last-Seen Age Sensors (trigger-based, 1-min update)

| Entity | Note |
|--------|------|
| `sensor.cam04_last_seen_seconds` | |
| `sensor.cam05_last_seen_seconds` | |
| `sensor.cam07_last_seen_seconds` | |
| `sensor.cam09_last_seen_seconds` | |
| `sensor.cam12_last_seen_seconds` | |
| `sensor.cam14_last_seen_seconds` | |
| `sensor.cam15_last_seen_seconds` | |
| `sensor.cam01_last_seen_seconds` | **ORPHAN** — cam01 deprecated; entity registry only |
| `sensor.cam06_last_seen_seconds` | **ORPHAN** — cam06 removed |
| `sensor.cam10_last_seen_seconds` | **ORPHAN** — cam10 deprecated |

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
| `input_text.cam04_car_port_front_images` | text | 255 | Per-cam image buffer |
| `input_text.cam05_inside_garage_images` | text | 255 | Per-cam image buffer (renamed 2026-05-17 from cam05_front_driveway_images) |
| `input_text.cam07_front_kitchen_images` | text | 255 | Per-cam image buffer |
| `input_text.cam09_back_bedroom_images` | text | 255 | Per-cam image buffer |
| `input_text.cam12_back_pond_images` | text | 255 | Per-cam image buffer |
| `input_text.cam14_lounge_images` | text | 255 | Per-cam image buffer |
| `input_text.cam15_passage_images` | text | 255 | Per-cam image buffer |
| `input_text.ipcam01_street_driveway_up_images` | text | 255 | Per-cam image buffer |
| `input_text.ipcam02_street_driveway_down_images` | text | 255 | Per-cam image buffer |
| `input_text.ipcam03_driveway_images` | text | 255 | Per-cam image buffer |
| `input_text.ipcam04_pool_bar_images` | text | 255 | Per-cam image buffer |
| `input_text.ipcam05_back_boundary_images` | text | 255 | Per-cam image buffer |
| `input_text.cam04_car_port_front_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam05_inside_garage_history` | text | 255 | Per-cam snapshot history (renamed 2026-05-17 from cam05_front_driveway_history) |
| `input_text.cam07_front_kitchen_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam09_back_bedroom_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam12_back_pond_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam14_lounge_history` | text | 255 | Per-cam snapshot history |
| `input_text.cam15_passage_history` | text | 255 | Per-cam snapshot history |
| `input_text.ipcam01_street_driveway_up_history` | text | 255 | Per-cam snapshot history |
| `input_text.ipcam02_street_driveway_down_history` | text | 255 | Per-cam snapshot history |
| `input_text.security_image_perimeter_front` | text | 255 | **Added S8 2026-05-19.** Last snapshot from ipcam01/02 — router reads this for visitor/perimeter_threat events |
| `input_text.security_image_perimeter_rear` | text | 255 | **Added S8 2026-05-19.** Last snapshot from ipcam05 (rear boundary) |
| `input_text.security_image_grounds_front` | text | 255 | **Added S8 2026-05-19.** Last snapshot from ipcam03/cam04/cam07 — router reads for arrival/departure events |
| `input_text.security_image_grounds_rear` | text | 255 | **Added S8 2026-05-19.** Last snapshot from ipcam04/cam09/cam12 |
| `input_text.security_image_inside_garage` | text | 255 | **Added S8 2026-05-19.** Last snapshot from cam05 (garage interior) |
| `input_text.security_image_inside_main` | text | 255 | **Added S8 2026-05-19.** Last snapshot from cam14 (lounge) |
| `input_text.security_image_inside_bedrooms` | text | 255 | **Added S8 2026-05-19.** Last snapshot from cam15 (passage) |

### Condition / Context Sensors (security_core.yaml)

| Entity | Purpose | Note |
|--------|---------|------|
| `binary_sensor.boundary_permissive_window` | True during maid/guest window | ✅ Fixed S1.3 2026-05-17 — now uses `low_trust_present OR guest_mode OR boundary_permissive_override` |
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

SOLE PATH after S3 (2026-05-17) + S7 (2026-05-18) + S8 (2026-05-19):
  security_event_router (trigger: sensor.security_event_classification state change)
    → dispatches by classifier output:
        arrival/departure     → logbook only (stage1/stage2 own notifications)
        service_person        → script.notify_security_event (10-min cooldown via last_visitor_event)
        visitor               → script.notify_security_event (critical, 60s cooldown — BUG-S27 fix)
        perimeter_threat      → script.notify_security_event (warning)
        intruder              → script.notify_security_event (critical)
        critical_intrusion    → script.notify_security_event (critical)
    → severity: 'information'/'warning'/'critical' (NOT 'info')
    → image: zone-aware lookup via zone_label attribute (S8 BUG-S22/S23 fix):
        'perimeter front' → input_text.security_image_perimeter_front
        'perimeter rear'  → input_text.security_image_perimeter_rear
        'grounds'         → input_text.security_image_grounds_front
        'gate'            → input_text.security_image_grounds_front
        'garage'          → input_text.security_image_inside_garage
        'main house'      → input_text.security_image_inside_main
        'bedroom passage' → input_text.security_image_inside_bedrooms
        fallback          → input_text.security_last_motion_image (global)

Two-stage arrival/departure (S11b — updated 2026-05-22):
  security_gate_vehicle_stage1 (arrival) → fires on binary_sensor.main_gate_sensor ON
    condition: ipcam01 fired within 120s (vehicle from street = arrival)
  security_gate_vehicle_stage1 (departure) → fires on ipcam03_driveway_exit_valid
    condition: gate opened without recent ipcam01 (car reversing = departure)
  security_arrival_stage2_confirm  → 3.5min delay, AP delta confirms who arrived
  security_departure_stage2_confirm → 3.5min delay, AP delta confirms who left

  Note: ipcam03 Region Entrance disabled (2026-05-22). entrance_valid trigger removed.
  ipcam03 line crossing set to driveway→gate direction (departure only).

Deleted in S3: security_grounds_motion, security_rear_grounds_motion,
security_house_motion, security_visitor, security_arrival_detected.
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
**Priority: HIGH | Status: ✅ RESOLVED 2026-05-17 (S3)**

**Symptom:** When grounds motion fires with `correlation = intruder`, three automations
fired simultaneously: `security_grounds_motion`, `security_rear_grounds_motion`, and
`security_house_motion`. All three sent separate notifications for the same event.

**Root cause:** All three automations triggered on `sensor.security_correlation to: "intruder"`.

**Band-Aid applied 2026-04-16:** Dedup gate via `binary_sensor.security_intruder_active`
(`delay_off: 30s`) + zone conditions on each automation. This eliminated the simultaneous
triple-fire but the three automations were still parallel notification paths.

**Architectural fix applied 2026-05-17 (S3):** All three automations deleted. Notification
now solely through `security_event_router` which reads `sensor.security_event_classification`
(S2 presence-first classifier). Two-stage arrival/departure automations replace
`security_arrival_detected`. Visitor branch absorbed into router.

*Automations deleted: security_grounds_motion, security_rear_grounds_motion,
security_house_motion, security_visitor, security_arrival_detected.*

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
**Priority: MEDIUM | Status: Partially resolved 2026-05-17 (S3)**

`security_visitor` and `security_arrival_detected` deleted in S3 — no longer applicable.
`security_arrival_stage1_vehicle` and `_departure_stage1_vehicle` use `from: "off"` ✓.
Remaining open: `security_capture_each_camera_motion`, `security_track_movement_path`,
`security_event_start` — low risk, deferred to S4.

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

### BUG-S14 — No presence-first logic in classifier (new events classified as threats)
**Priority: HIGH | Status: ✅ FIXED 2026-05-17 (S2)**

Family arriving at their own gate classified as intruder because presence-check happened
after threat-check. Rebuilt `sensor.security_event_classification` as a 9-rung
presence-first ladder. Gate + AP arrival = `arrival` before any threat rung is reached.

---

### BUG-S15 — No two-stage arrival model (AP transition not used for arrival detection)
**Priority: HIGH | Status: 🔧 Partially fixed S2 / S3 completes**

`family_arriving` binary sensor now feeds the classifier. Full two-stage model (gate
event + AP transition + ipcam03 entrance confirmation) wired in S3 router.

---

### BUG-S16 — `security_inside_house_motion` has no zone split (bedroom/main/garage)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-17 (S2)**

Three zone sensors added: `security_inside_garage_motion`, `security_inside_main_motion`,
`security_inside_bedrooms_motion`. Zone arming gates: `binary_sensor.inside_*_armed`
backed by `input_boolean.inside_*_armed_manual`. Bedrooms armed in away only (bathroom
trips suppressed at night stay). `security_inside_house_motion` updated to be a backward-
compatible raw union of the three.

---

### BUG-S17 — Garage camera not in an inside zone (cam05 classified as grounds)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-17 (S2)**

cam05 physically inside garage since 2026-05-07 but classified as `grounds_front`.
`security_inside_garage_motion` now also reads `cam05_front_driveway_motion_valid`.
cam05 remains in `grounds_front` group for backward compatibility; classifier zones
disambiguate via the arming gate.

---

### BUG-S18 — False CRITICAL INTRUDER when family home at night
**Priority: HIGH | Status: ✅ FIXED 2026-05-18 (S7)**  
See Sprint 7 in Section 9.

---

### BUG-S19 — Staff on site spam (cam09 — no cooldown)
**Priority: HIGH | Status: ✅ FIXED 2026-05-18 (S7)**  
See Sprint 7 in Section 9.

---

### BUG-S20 — Arrival spam (3-5 duplicate "Family arriving" notifications)
**Priority: HIGH | Status: ✅ FIXED 2026-05-18 (S7)**  
See Sprint 7 in Section 9.

---

### BUG-S39 — RUNG 2.5 false critical at 5am (no time gate, zero cooldown)
**Priority: CRITICAL | Status: ✅ FIXED 2026-05-23 (S13)**

**Symptom:** At 05:03am on 2026-05-23, family got up early (5am). Received 1 blank CRITICAL INTRUDER notification followed by 5 more all showing the same stale cam14_lounge image (05:03:58 timestamp). Notification details: `home: all | zones: Main+Beds | gate: closed | arriving: no | departing: no | staff: no | conf: none | cam: cam14_lounge`.

**Root causes (three combined):**
1. **No time gate on RUNG 2.5:** Condition fires at 5am because `is_late_night = (h >= 23 or h < 6)` is still true at 05:03. Family got up but RUNG 2.5 has no floor — it fires right down to 05:59:59.
2. **AP location delay:** `all_family_in_bedrooms` uses AP association (15s RAW + up to 5min confidence delay). Family walking to lounge triggers cam14 before APs roam → bedrooms still true at moment of trigger.
3. **Zero cooldown on critical_intrusion in router:** `security_event_router` has explicit bypass for critical_intrusion (no 300s cooldown). ext_recent stays ON for 5min. cam14 fires repeatedly → 5 queued router dispatches all fire with no gate.
4. **Blank first image:** `security_capture_best_snapshot` has 2s delay. Router fires immediately after classifier changes → first image lookup hits empty/stale slot before snapshot writes.

**Fix plan:**
- Add `and (now().hour >= 23 or now().hour < 5)` to RUNG 2.5 (hard 5am floor) in security_logic.yaml
- Add 90s cooldown to critical_intrusion branch in security_event_router (separate input_datetime.last_critical_event)
- Add 3s delay before image slot lookup in router critical_intrusion branch

---

### BUG-S37 — ipcam03_driveway_exit_valid missing linedetection (departures missed)
**Priority: HIGH | Status: ✅ FIXED 2026-05-23 (S13)**

**Symptom:** Multiple missed departure detections. Gate opens for departure but Stage 1 never fires. AP-based departure confirmation also not triggered because Stage 1 (ipcam03_driveway_exit_valid) never goes ON.

**Root cause:** In S11b, ipcam03 Line Crossing was configured in the camera to driveway→gate direction (departure direction only). The `ipcam03_driveway_exit_valid` binary sensor in cameras_processing.yaml (line 156) only watches `binary_sensor.ipcam03_driveway_regionexiting`:
```yaml
state: "{{ is_state('binary_sensor.ipcam03_driveway_regionexiting','on') }}"
```
The `ipcam03_driveway_linedetection` entity fires for the departure direction line crossing but is NOT included in exit_valid. Result: departure line crossing signal is completely ignored by Stage 1.

**Fix plan:** cameras_processing.yaml — change exit_valid state to:
```yaml
state: "{{ is_state('binary_sensor.ipcam03_driveway_regionexiting','on') or is_state('binary_sensor.ipcam03_driveway_linedetection','on') }}"
```

---

### BUG-S43 — security_lighting_required references non-existent entity (boundary lights never trigger on weather)
**Priority: HIGH | Status: ✅ FIXED 2026-05-23 (S13)**

**Symptom:** Security boundary lights did not turn on during poor-visibility evening (2026-05-23). Primary trigger (night_early: sun elevation < 2°) should have fired regardless, but the bug prevents the weather/low-light fallback path from ever working.

**Root cause:** In security_core.yaml (line 78), `security_lighting_required` uses:
```yaml
{% set low = is_state('binary_sensor.security_visibility_low','on') %}
```
Entity `binary_sensor.security_visibility_low` does NOT exist. The correct entity (defined 5 lines above in the same file) is `binary_sensor.security_weather_low_light`. The template always evaluates `low = false` → weather condition never contributes to lighting_required.

**Secondary issue:** `boundary_security_on` automation (lighting_boundary.yaml) has no `continue_on_error` on switch.turn_on calls and mode:single — if Sonoff integration is slow to respond, the action fails silently with no retry and no alert.

**Fix plan:**
- security_core.yaml: change `security_visibility_low` → `security_weather_low_light`
- lighting_boundary.yaml: add `continue_on_error: true` to switch.turn_on sequence; add retry automation after 2min + warning notification if lights still off

---

### BUG-S40 — Staff/gardener visitor spam (30s cooldown insufficient for morning garden work)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-23 (S13)**

**Symptom:** Saturday morning gardener worked outside front gate for ~2hrs (08:00–10:00). Repeated visitor notifications every ~30s. Screenshot confirmed: `staff: yes | conf: medium | cam: ipcam01_street_driveway_up` with gardener and plants visible at gate.

**Root cause:** S10 correctly removed `not staff` from RUNG 5 (visitor must fire even when maid on site, so maid-at-gate is visible). But Saturday gardener does sustained work outside front gate — continuous ipcam01 motion triggers → visitor fires → 30s cooldown expires → fires again. Staff flag is shown in notification but cooldown is flat 30s regardless of staff_on_site state.

**Fix plan:** security_automations.yaml visitor branch: extend cooldown to 1800s (30min) when `binary_sensor.staff_on_site` is ON at time of visitor fire (use separate input_datetime or check elapsed since last_visitor_event).

---

### BUG-S41 — Stage 2 arrival notification shows carport NVR image (cam04 overwrites driveway slot)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-23 (S13)**

**Symptom:** Stage 2 arrival confirmations consistently show a dark NVR carport image (cam04) instead of the driveway approach image (ipcam03). Screenshot confirmed: Arrival Stage 1 and "Arrival confirmed Ryan" both showed cam04 carport frame.

**Root cause (two factors):**
1. **Shared slot overwritten:** Stage 1 fires on gate open → car drives up driveway → cam04 (carport) fires as car enters → writes `input_text.security_image_grounds_front`. Stage 2 fires 3.5 minutes later and reads this slot — which now holds the carport frame, not the driveway approach.
2. **cam04 ranks above ipcam03:** `security_trigger_camera` priority list: cam14 > cam15 > **cam04** > **ipcam03** > cam07 > ... cam04 (NVR, no AI, dark at night) outranks ipcam03 (IP, AcuSense, daylight quality), so the trigger camera recorded for the event is also cam04.

**Fix plan:**
- security_automations.yaml Stage 1: lock current `security_image_grounds_front` value into new `input_text.security_image_arrival_locked` at Stage 1 fire time
- Stage 2: read `security_image_arrival_locked` instead of live `security_image_grounds_front`
- security_logic.yaml trigger camera priority: promote ipcam03 above cam04

---

### BUG-S38 — Stage 1 arrival ipcam01 120s window too narrow (gate opens missed)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-23 (S13)**

**Symptom:** Arrival at ~18:59 on 2026-05-23 — screenshots show two gate opens. First gate open missed (Stage 1 not triggered); only second gate open detected. Earlier in the day, departure at ~10:20 also missed.

**Root cause:** Stage 1 condition requires `ipcam01_recent` (ipcam01 fired within 120s of gate open) to confirm vehicle came from street. If ipcam01 fires but the gate is delayed in opening (slow driver, fumbling with remote), or ipcam01 fires briefly and resets, the 120s window is missed and Stage 1 treats the gate open as not vehicle-from-street → no arrival.

**Fix plan:** security_automations.yaml Stage 1: extend ipcam01_recent window from 120s to 180s. Also add AP-based fallback path: if gate opens + nobody_home (departure context) or family_arriving (AP-detected arrival is coming), fire Stage 1 even without ipcam01 confirmation.

---

### BUG-S42 — Delivery person opening gate = false "Arrival — vehicle entering"
**Priority: LOW | Status: ✅ PARTIAL FIX 2026-05-23 (S13) — softer message; structural limitation remains**

**Symptom:** Delivery person rings at gate → user remotely opens gate → delivery walks through → Stage 1 fires "Arrival — vehicle entering" and Stage 2 "Arrival confirmed — Unknown home".

**Root cause:** Stage 1 detects gate open + ipcam01_recent → classifies as vehicle from street → arrival. A pedestrian (delivery, visitor on foot) also triggers ipcam01 and opens the gate. There is no pedestrian vs. vehicle distinction available from NVR/IP cameras at this point in the pipeline.

**Structural limitation:** HA cannot distinguish person-entering from car-entering using the current camera layout. True fix requires vehicle-only detection at street camera (Frigate with vehicle class, or dedicated vehicle detector).

**Partial fix plan:**
- Soften Stage 1 message: change "Arrival — vehicle entering" to "Gate event — someone entering" (less alarm-y for deliveries)
- Stage 2: if nobody newly arrived (delta = nobody), change "Unknown home" → "Visitor/delivery — nobody added to home" (better context)

---

### BUG-S36 — Departure misclassified as arrival (entrance_valid fires for departing car)
**Priority: HIGH | Status: ✅ FIXED 2026-05-21 (S10)**

**Symptom:** At 06:55 on 2026-05-21, family departed but Telegram showed "Arrival — vehicle entering" and "Arrival confirmed — Unknown home at 06:59." Gate event detected correctly, but Stage 1 classified as arrival and Stage 2 confirmed Unknown (nobody newly home).

**Root cause:** `ipcam03_driveway_entrance_valid` fires when a departing car backs through the gate-mouth region (entrance detection zone positioned at top of frame, gate side). Old logic: entrance_valid → always arrival. A departing car reversing through the gate triggers entrance_valid exactly as an arriving car does.

**Fix:** Stage 1 `cam_dir` now inlines ipcam01 (street-up) recency check:
- `entrance_valid` + ipcam01 fired within 120s → arrival (car came from street)
- `entrance_valid` + ipcam01 NOT recent → departure (car was already inside, backing out)
- `exit_valid` → always departure

Also added mutual 120s suppression condition to Stage 1: the second sensor of a traversal (arrival: exit fires after entrance; departure: entrance fires after exit) is blocked, preventing double Stage 1 notifications per vehicle movement.

---

### BUG-S29 — ipcam03 regionentrance and regionexiting zones overlap (direction confusion)
**Priority: HIGH | Status: ✅ HA workaround applied 2026-05-20 (S9) | Camera fix: OPEN**

**Symptom:** Vehicle departing triggers "Arrival — vehicle entering" notification. Vehicle arriving triggers both arrival and departure Stage 1 simultaneously, or just the wrong one. Both `ipcam03_driveway_entrance_valid` and `ipcam03_driveway_exit_valid` fire for any vehicle movement in the driveway regardless of actual direction.

**Root cause:** Camera zone configuration — the Region Entrance and Region Exiting detection zones on ipcam03 are positioned in overlapping areas (both mid-driveway near gate). Any vehicle traversing the driveway crosses both zones. Confirmed from iVMS camera web UI (10.10.1.12) 2026-05-20.

**HA workaround (applied S9):** Merged Stage 1 automation (`security_gate_vehicle_stage1`) replaces separate arrival/departure Stage 1s. Waits 90s after ipcam03 trigger, then reads `family_arriving`/`family_departing` (AP snapshot-delta) to determine direction. Only the correct branch fires. `mode: single` prevents dual-fire when both sensors trigger together.

**Camera fix (pending — access required to iVMS):**
- Region Entrance: reposition to gate-mouth only (narrow strip at top of frame, right where gate opens)
- Region Exiting: reposition to mid-driveway (lower 40% of frame, between garage and gate)
- Zero overlap between the two zones — see camera table in Section 1 for geometry diagram
- Line Crossing: change Direction from A↔B to A→B (one direction only)
- Intrusion (Field): add Vehicle to detection targets (currently Human only on ipcam03)
- On ipcam01: Region Exiting — consider disabling (unused in HA); Line Crossing A↔B → A→B

---

### BUG-S21 — ipcam05 (rear boundary) triggers VISITOR classification
**Priority: HIGH | Status: ✅ FIXED 2026-05-19 (S8)**

**Symptom:** When ipcam05 (back boundary wall, outside property) detected motion at night,
the classifier returned `visitor` and sent a critical "Visitor at gate" alert. ipcam05 is
outside the rear wall — there is no gate there. It is never visitor territory.

**Root cause:** RUNG 5 (visitor) used `perim` (combined perimeter_front OR perimeter_rear).
Any perimeter camera firing = visitor if gate closed and not arriving. ipcam05 feeds
`security_perimeter_rear_motion`, which feeds `perim`.

**Fix:** Split `perim` into `perim_front` (ipcam01/02) and `perim_rear` (ipcam05) in the
classifier. RUNG 5 now only fires on `perim_front`. `perim_rear` falls through to RUNG 6
(perimeter_threat). `zone_label` attribute updated to return `'perimeter front'` or
`'perimeter rear'` separately.

---

### BUG-S22 / BUG-S23 — Wrong-camera image in security notifications
**Priority: HIGH | Status: ✅ FIXED 2026-05-19 (S8)**

**Symptom:** "Visitor at gate" alert showed carport image. "Intruder on property" alert
showed an unrelated camera's image from a few seconds earlier.

**Root cause:** Single global `input_text.security_last_motion_image` overwritten by
`security_capture_best_snapshot` whenever any camera fires at medium/high confidence.
Router reads this at action-start time — if another camera fired between classifier
evaluation and router execution, the image is stale/wrong-zone.

**Fix:** 7 per-zone `input_text.security_image_*` helpers added. `security_capture_best_snapshot`
writes the zone-appropriate slot (based on `sensor.security_trigger_camera` entity_id).
Router `img` variable does a zone-map lookup on `zone_label` attribute, falling back to
global `security_last_motion_image` if the zone slot is empty.

---

### BUG-S24 / BUG-S25 — Stage 2 "who" is a snapshot not a delta
**Priority: HIGH | Status: ✅ FIXED 2026-05-19 (S8)**

**Symptom:** "Arrival confirmed — Ryan, Vicky, Luke" when only Luke arrived (Ryan and Vicky
were already home). Stage 2 departure showed people who had been Away all day, not who
just left.

**Root cause:** Stage 2 `who` template iterated all 4 people and collected everyone
currently in a home zone. No before/after comparison — no delta logic.

**Fix:** Stage 1 arrival now writes `input_text.arrival_who_was_home` (comma-joined names
of everyone home at Stage 1 fire time). Stage 1 departure writes `input_text.departure_who_was_home`.
Stage 2 computes the delta: arrival = home_now AND NOT in before-snapshot; departure =
away_now AND WAS in before-snapshot. These helpers live in `presence_helpers.yaml` and
are kept separate from the rolling `who_was_home_snapshot` so the 60s automation doesn't
overwrite the Stage 1 snapshot during the 3.5-minute Stage 2 delay.

---

### BUG-S26 — family_arriving fires on intra-home AP roaming
**Priority: HIGH | Status: ✅ FIXED 2026-05-19 (S8)**

**Symptom:** Ryan walks from Lounge to Kitchen. `sensor.ryan_ap_location` `last_changed`
updates. `binary_sensor.family_arriving = ON` for 600 seconds. During that window, any
gate opening (delivery, dog walk) classifies as `arrival` not `visitor`.

**Root cause:** `family_arriving` logic: `st in home_zones AND changed <= 600s`. This fires
for any home-zone AP change, not just transitions from Away → home.

**Fix:** `family_arriving` rewritten to snapshot-delta: ON only if person is in a home zone
AND their label is NOT in `input_text.who_was_home_snapshot`. The snapshot is updated every
60 seconds by `presence_snapshot_who_home` automation. Once the person is in the snapshot
(after 60s), `family_arriving` turns OFF. `family_departing` updated same way.

---

### BUG-S27 — Visitor branch has no cooldown (duplicate notifications)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-19 (S8)**

**Symptom:** Sustained presence at the gate (person standing at intercom for 2+ min) produces
3-4 visitor notifications as the ipcam01 debounce cycle (30s delay_off) pulses the
classifier `idle → visitor → idle → visitor`.

**Root cause:** Router visitor branch had no cooldown check — fired on every classifier
transition to `visitor`.

**Fix:** 60-second cooldown added using `input_datetime.last_visitor_event`. Short enough
for a genuine second visitor within the hour; long enough to suppress the debounce pulse.

---

### BUG-S44 — cam14 / cam15 NVR false motion from go2rtc reconnect replay
**Priority: HIGH | Status: ✅ FIXED 2026-05-26 (S14)**

**Symptom:** cam14 (lounge) and cam15 (passage) fire motion events in HA that have NO
corresponding entry in the Hikvision NVR app. Two patterns observed:
1. "Old image" in notification — snapshot shows an image from hours earlier.
2. "Repeated" alerts — multiple critical_intrusion events within minutes.

**Root cause:** go2rtc NVR timeout instability (documented in contract). When go2rtc
reconnects after a timeout, HA's hikvision integration briefly replays the last-known
state of `cam14_lounge_motiondetection` / `cam15_passage_motiondetection` as "on".
This is an instantaneous state replay (<1s) — the NVR never raised a new motion event,
so nothing appears in the Hikvision app.

The "old image" confirms this: `security_capture_best_snapshot` requires `medium/high`
confidence before writing `security_image_inside_main` — but cam14 alone is confidence
"none" (inside cameras excluded from the confidence score). So the zone slot holds the
last real image from a previous event hours earlier.

The "repeated" pattern: cam12 (pond) fires legitimately → provides `ext_recent = ON` →
go2rtc reconnects → cam15 replays "on" immediately → both satisfy RUNG 2.5 / RUNG 8
conditions → critical_intrusion.

**Fix:** Added `delay_on: "00:00:03"` to `cam14_lounge_motion_valid` and
`cam15_passage_motion_valid` in `cameras_processing.yaml`. A go2rtc state replay is
instantaneous (< 1s) and will not survive the 3s hold-on filter. Genuine human motion
(person walking through lounge or passage) stays ON for multiple seconds. This
eliminates the false-positive without any risk of missing real events.

---

### BUG-S45 — ipcam04 pool camera alarm fires independently of HA (camera-side linkage)
**Priority: HIGH | Status: 🔧 PARTIAL FIX — requires camera-side config change**

**Symptom:** Turning on `input_boolean.security_dogs_out` did not stop the pool camera
alarm from sounding. HA's `security_pool_alarm_trigger` correctly conditions on
`security_dogs_out = off`, so the alarm should have been suppressed.

**Root cause (two parts):**
1. **Camera-side alarm linkage:** ipcam04 almost certainly has Smart Event → Intrusion
Detection → Linkage Method → "Alarm Output" checked in the camera web UI
(http://10.10.1.13). This means the CAMERA ITSELF triggers the alarm output whenever
AcuSense detects a person/vehicle in the field detection zone — completely independently
of HA. HA's `dogs_out` flag has zero effect on this camera-side linkage.
2. **No deactivation command:** HA's `rest_command.ipcam04_trigger_alarm` only activates
the alarm output. Once activated (either by HA or by the camera's own linkage), there was
no HA command to CANCEL it before the camera's configured alarm duration expired.

**Fix applied in S14 (2026-05-26):**
- `rest_command.ipcam04_deactivate_alarm` added (security_helpers.yaml) — sends
  `outputState: inactive` to immediately cancel the alarm output.
- `security_dogs_out_cancel_alarm` automation added (security_automations.yaml) — fires
  when `security_dogs_out` turns ON, calls deactivate immediately to silence any active alarm.

**Remaining action required (camera web UI):**
To give HA full control, disable the camera's own alarm output linkage:
- Open http://10.10.1.13 → Configuration → Event → Smart Event → Intrusion Detection
- In "Linkage Method", uncheck "Alarm Output"
- Repeat for any other Smart Event types (Line Crossing etc.) that have Alarm Output linked.
- After this change: the alarm can ONLY be triggered by HA's `rest_command.ipcam04_trigger_alarm`.

---

### BUG-S46 — cam15_passage triggers critical_intrusion during family arrival (AP transition)
**Priority: MEDIUM | Status: ✅ FIXED 2026-05-26 (S14)**

**Symptom:** Critical security event showing cam15_passage as trigger camera, occurring
during the day / early evening when family is (or appears to be) home. Often coincides
with cam12_back_pond motion firing.

**Root cause:** RUNG 8 was missing a `not arriving` guard. During arrival (AP transitioning
from Away → home, phone connects within seconds of entering property), `anyone_connected_home`
can briefly be OFF (2-min delay_off window). At that moment:
- `inside_armed_active` = ON (arming schedule activated on departure, not yet disarmed)
- `not anyhome` = true (AP not yet connected)
- cam12 provides `ext_recent`
- cam15 go2rtc replay fires → RUNG 8 → `critical_intrusion`

Adding `and not arriving` to RUNG 8 prevents this: `family_arriving = on` when the person
is transitioning to home (snapshot-delta), providing the correct suppression window.

**Fix:** Added `and not arriving` to RUNG 8 condition in `security_logic.yaml`.

---

### BUG-S47 — Stale image in critical_intrusion and visitor notifications
**Priority: HIGH | Status: ✅ FIXED 2026-05-27 (S15)**

**Symptom:** Critical intrusion notifications (RUNG 2.5) and visitor notifications show
images timestamped hours or days before the notification. Examples from screenshots:
- 01:44 critical_intrusion notification shows cam14_lounge image timestamped 04:08 previous morning
- 03:04 critical_intrusion shows cam14_lounge image timestamped 02:14 (50min stale)
- 07:00 visitor notification shows ipcam01 image timestamped 21:32 previous night

**Root cause (two separate paths):**

**Path 1 — Inside camera (RUNG 2.5):**
`security_capture_best_snapshot` gates on `confidence in ['medium','high']`. Inside cameras
(cam14, cam15, cam05) are excluded from confidence scoring. cam14 alone = confidence "none".
cam12 (rear NVR, LOW confidence) + cam14 = still "low". So `security_image_inside_main` is
NEVER updated during RUNG 2.5 events. The critical_intrusion router reads a slot that was
last written during a much earlier medium/high confidence outdoor event — hours or days stale.
`security_capture_each_camera_motion` was capturing per-camera snapshots and writing history
but NOT writing the per-zone slot (`input_text.security_image_inside_main`).

**Path 2 — Visitor (timing race):**
The router fires immediately when `security_event_classification` → `visitor`. The top-level
`variables:` block reads `input_text.security_image_perimeter_front` at that instant — before
`security_capture_best_snapshot` (which has a 2s delay + snapshot time) has written a fresh
image. The router sends the notification with the stale slot value.

**Fix — Path 1:**
Added a zone-slot write step at the end of `security_capture_each_camera_motion` for inside
cameras (cam14 → `security_image_inside_main`, cam15 → `security_image_inside_bedrooms`,
cam05 → `security_image_inside_garage`). This runs after the 1s snapshot delay, so the slot
is written ~2s before the router's 3s delay expires. Slot is now guaranteed fresh on any valid
inside camera motion, regardless of confidence.

**Fix — Path 2:**
Added `delay: "00:00:04"` before image read in visitor branch, plus a local `visitor_img`
re-read (same pattern as `critical_intrusion` branch). The 4s wait gives
`security_capture_best_snapshot` (2s delay + snapshot ≈ 3–4s) time to write the fresh slot.
`security_automations.yaml` modified.

---

### BUG-S28 — ipcam02 dead signal (firmware incompatibility)
**Priority: MEDIUM | Status: OPEN — awaiting installer**

**Symptom:** `ipcam02_street_driveway_down_motion_valid` always OFF. `security_perimeter_front_motion`
effectively ipcam01-only. All ipcam02 fielddetection/linedetection/regionentrance entities
do not exist in HA.

**Root cause:** ipcam02 firmware V5.8.13 branch H13U incompatible with hikvision_next Smart
Event ISAPI discovery. Only `scenechangedetection` is discovered. See Section 10.9 for full
troubleshooting history.

**Impact:** Perimeter front has only one camera (ipcam01 street_up). ipcam02 (street_down)
contributes zero signal. No code fix possible — requires firmware rollback to V5.7.19 G5 or
camera replacement. Installer contact initiated 2026-05-11.

**Mitigation:** OR in the perimeter_front zone sensor is harmless (always-off sensor has no
effect). No code change needed until ipcam02 is repaired.

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
[✅] Add cam14/cam15 to security_trigger_camera priority list (top) — DONE 2026-05-18
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

SPRINT 15 — Stale image fixes + perimeter_front rung split (2026-05-27)
```
[✅] BUG-S47 (Path 1): security_image_inside_main never updated during RUNG 2.5 events.
      security_capture_best_snapshot gates on medium/high confidence; cam14 alone = none,
      cam12+cam14 = low — slot was never written. Added zone-slot write step to
      security_capture_each_camera_motion for cam14/cam15/cam05, ensuring the slot is
      always fresh when a valid inside motion occurs. security_automations.yaml modified.

[✅] BUG-S47 (Path 2): Visitor notification reads stale perimeter_front slot (timing race).
      Router fires simultaneously with security_capture_best_snapshot trigger; slot hasn't
      been written yet. Added 4s delay + visitor_img re-read in visitor router branch,
      matching the pattern already used by critical_intrusion branch. Slot is now read
      after the snapshot has been captured and written. security_automations.yaml modified.

[✅] RUNG 5 split — perimeter_front vs visitor using AcuSense entrance_valid:
      Original RUNG 5 classified ALL ipcam01/02 activity as "visitor". Split into:
      RUNG 5a (visitor): entrance_valid (ipcam01 regionentrance) active → person in gate
        approach zone → "Visitor at gate" critical alert (genuine intent to enter).
      RUNG 5b (perimeter_front): entrance_valid NOT active → street/passing activity →
        "Activity on front perimeter" critical alert (only at night or nobody home).
      Daytime street activity with family home now falls silently to family_movement.
      entrance_valid = binary_sensor.ipcam01_street_driveway_up_entrance_valid (already
      calibrated as "Higher validity" gate approach zone in ipcam01 Smart Events).
      security_logic.yaml + security_automations.yaml modified.
```

SPRINT 14 — go2rtc replay filter + RUNG 8 arrival guard + alarm deactivation (2026-05-26)
```
[✅] BUG-S44: cam14/cam15 NVR false motion from go2rtc reconnect replays.
      Added delay_on: 3s to cam14_lounge_motion_valid and cam15_passage_motion_valid.
      go2rtc state replays are instantaneous (<1s) and filtered by the hold-on window.
      Genuine human motion stays ON for multiple seconds and passes through unchanged.
      cameras_processing.yaml modified.

[✅] BUG-S46: RUNG 8 missing 'not arriving' guard — critical_intrusion during arrival.
      Added 'and not arriving' to RUNG 8 condition in security_logic.yaml.
      Prevents cam15 false alert when family member is walking through house while AP
      has not yet fully connected (2-min delay_off still in effect from departure).

[✅] BUG-S45 (partial): Pool alarm not cancelable from HA when dogs_out turns ON.
      Added rest_command.ipcam04_deactivate_alarm (outputState: inactive) to
      security_helpers.yaml.
      Added security_dogs_out_cancel_alarm automation to security_automations.yaml —
      triggers when security_dogs_out turns ON, immediately calls deactivate.
      REMAINING ACTION: In ipcam04 camera web UI (http://10.10.1.13), remove "Alarm Output"
      from Smart Event Intrusion Detection linkage method. Camera-side linkage fires
      independently of HA and cannot be suppressed by dogs_out until disabled.

[ℹ️] Camera alarm output not visible in Hikvision app: EXPECTED BEHAVIOUR.
      ISAPI alarm output trigger (/ISAPI/System/IO/outputs/1/trigger) activates the
      physical alarm but does NOT create a "Motion Detection Alarm" or Smart Event.
      Hikvision app shows Motion Detection / Smart Events only — not alarm output activations.
      To verify HA trigger is working: ipcam04 camera web UI → System → Maintenance → Log
      → look for "Alarm Output" entries timestamped when HA triggered the alarm.
```

SPRINT 13 — BUG-S39/S37/S43/S40/S41/S38/S42 batch fix (2026-05-23)
```
[✅] BUG-S39: RUNG 2.5 false critical at 5am — added hard 5am floor (now().hour < 5).
      90s cooldown added to critical_intrusion router branch.
      3s delay added before critical_intrusion image lookup.

[✅] BUG-S37: ipcam03 exit_valid missing linedetection — departure misses.
      exit_valid state updated to include linedetection OR regionexiting.

[✅] BUG-S43: security_lighting_required visibility entity name wrong.
      security_visibility_low → security_weather_low_light in security_core.yaml.

[✅] BUG-S40: Gardener visitor spam — 30s cooldown extended to 1800s (30min) when
      binary_sensor.staff_on_site is ON at time of visitor fire.

[✅] BUG-S41: Stage 2 arrival shows carport image (cam04 overwrites grounds_front slot).
      input_text.security_image_arrival_locked added. Stage 1 locks the driveway image
      at T+5s before car reaches carport. Stage 2 reads locked slot, not live grounds_front.

[✅] BUG-S38: Stage 1 arrival ipcam01 120s window too narrow.
      Extended from 120s to 180s in Stage 1 condition.

[✅] BUG-S42: Delivery person classified as "Arrival — vehicle entering".
      Stage 1 message softened to "Gate opened — vehicle entering". Stage 2 Unknown
      changed to "Visitor/delivery — nobody newly detected home".
```

SPRINT 11c — entrance_valid source changed to fielddetection (2026-05-22)
```
[✅] ipcam03_driveway_entrance_valid: regionentrance → fielddetection (cameras_processing.yaml).
      regionentrance disabled in ipcam03 camera UI — entity was always OFF.
      fielddetection (Intrusion zone) restores the "vehicle/person confirmed in driveway"
      signal for confirmed_human in security_threat_level sensor.
      entrance_valid is no longer a Stage 1 trigger (main_gate_sensor replaced it in S11b).
      No functional change to arrival/departure routing — only confirmed_human restored.

[ℹ️] ipcam04 Target Validity: confirmed already on High (no change needed).
[ℹ️] cam05 (inside garage): NVR motion sensitivity raised 20→40 (NVR web UI).
      Was triggering NVR recording but not sending ISAPI events to HA. Sensitivity 20
      was below the ISAPI reporting threshold for pixel-diff in a dark garage interior.
```

SPRINT 11b — Stage 1 rewired for new ipcam03 camera config (2026-05-22)
```
Camera changes applied by user (ipcam03 driveway, 2026-05-22):
  - Region Entrance Detection: DISABLED (was firing for departures backing through gate-mouth)
  - Line Crossing: A→B = driveway→gate direction only (departure direction)
  - Region Exiting: unchanged (lower driveway zone, departure signal)

Effect on HA: ipcam03_driveway_entrance_valid always OFF (regionentrance disabled).
Stage 1 entrance_valid trigger is now inert.

[✅] Stage 1 trigger: replaced ipcam03_driveway_entrance_valid with main_gate_sensor
      (from: "off", to: "on"). Gate opens after ipcam01 detects street approach = arrival.

[✅] Stage 1 condition: replaced S10 mutual 120s entrance/exit suppression with:
      gate trigger: proceeds only if ipcam01_recent (< 120s). If ipcam01 not recent,
        gate open = departure initiation → skip, exit_valid will handle it.
      exit_valid trigger: suppressed if gate_age < 120s AND ipcam01_recent.
        This means arrival car crossing exit zone on way to garage → blocked.
        Departure (gate opened without recent ipcam01) passes through.

[✅] Stage 1 cam_dir: simplified.
      main_gate_sensor trigger → arrival (condition already guaranteed ipcam01_recent).
      exit_valid trigger → departure (condition already suppressed arrival traversals).
      ap_dir fallback retained for unknown-trigger edge cases.
      Removed both_active check (no longer relevant — two distinct signals for arrival/departure).

New arrival flow:
  ipcam01 fires → visitor notification (immediate, S11) → family opens gate →
  gate_sensor on → Stage 1 arrival → Stage 2 confirms who (3.5min AP delta).

New departure flow:
  Car reverses → ipcam03 exit zone fires → exit_valid → Stage 1 departure →
  Stage 2 confirms who.
```

SPRINT 11 — False intruder fixes + visitor immediate + outdoor corroboration (2026-05-22)
```
[✅] anyone_connected_home: added delay_off: 2min (presence_core.yaml).
      Was: dropped to OFF the instant last AP disconnected → departing car in grounds
      triggered intruder before the gate even closed. 2-min grace covers full departure
      sequence, phone-sleep AP drop, and inter-AP roaming handoff disconnections.

[✅] RUNG 7 (intruder): added 'and not departing'.
      Departing car briefly visible in grounds after gate closes + AP disconnect was
      hitting RUNG 7 → CRITICAL intruder. family_departing=ON suppresses this window.

[✅] RUNG 8d (inside-only fallback): added 'and not departing'.
      Same departure window can briefly see inside sensors. Suppressed.

[✅] security_external_motion_recent: new binary_sensor in security_zones.yaml.
      delay_off: 5min. ON whenever any perimeter or grounds camera is active.
      Represents "outdoor camera fired in the last 5 minutes" — path corroboration
      for inside camera alerts. A genuine intruder MUST cross an outdoor zone first.

[✅] RUNG 2.5 (stay-mode lounge intrusion): added 'and ext_recent'.
      cam14 (NVR, no AI) fires constantly at night from headlights through kitchen window.
      Now requires outdoor camera fired within 5min before lounge fires = credible path.
      Without outdoor context = headlight/shadow = falls to RUNG 3 (family_movement).

[✅] RUNG 8 (critical_intrusion nobody home): added 'and (ext_recent or ip_cam or high_conf)'.
      cam14/cam15 NVR firing alone when nobody home = shadow/cat → family_movement (silent).
      ext_recent: outdoor camera fired in last 5min (intruder came from somewhere).
      ip_cam: AcuSense camera = AI-confirmed person/vehicle (credible even without outdoor).
      high_conf: multi-zone corroboration. Falls to RUNG 8d (silent) if none of these met.
      Kitchen-window blind spot covered: genuine intruder via back kitchen door would still
      trigger cam09 (rear bedroom area) or cam07 (side entry) → ext_recent ON.

[✅] zone_label: 'grounds' split into 'grounds front' / 'grounds rear' based on trigger camera.
      Rear cameras (ipcam04, cam09, cam12) → 'grounds rear' → router maps to
      security_image_grounds_rear (was always mapping to grounds_front = carport image).

[✅] Router zone_img map: added 'grounds rear' → security_image_grounds_rear entry.
      'grounds front' added as explicit key (was only 'grounds' before).

[✅] Router reason + zone: changed from live state_attr() to trigger.to_state.attributes.
      With mode:queued, router fires seconds after classification; by then all debounce
      sensors clear → live reason showed 'zones: none, conf: none'. Snapshot at trigger
      time preserves the values at the moment of classification.

[✅] Visitor branch: removed 45s grace period. Notification fires immediately.
      Cooldown reduced 60s → 30s (handles ipcam01+ipcam02 dual-fire for same vehicle).
      User priority: see gate approach immediately, then arrival Stage 1 follows if family.
      Two notifications for family arrival (visitor + arrival) is intentional and informative.
```

SPRINT 10 — Direction fix + staff muting + perimeter always active (2026-05-21)
```
[✅] BUG-S36: Departure misclassified as arrival (confirmed by Telegram 06:55 2026-05-21).
      Root: ipcam03_driveway_entrance_valid fires when a departing car backs through the
      gate-mouth zone (entrance detection strip at top of frame). Current logic treated
      ANY entrance_valid as arrival — wrong when car is already on property departing.
      Fix: entrance_valid = arrival ONLY if ipcam01 (street-up) fired within 120s.
      If ipcam01 NOT recent, the car was already inside the property = departure backing
      through the gate. exit_valid still always = departure. cam_dir variable in Stage 1
      now inlines ipcam01 recency check; ap_dir fallback retained for unknown-trigger edge cases.

[✅] Mutual suppression added to Stage 1 condition block:
      If entrance_valid triggers BUT exit_valid last_changed < 120s → suppress.
      If exit_valid triggers BUT entrance_valid last_changed < 120s → suppress.
      Prevents the arriving vehicle crossing the exit zone (on way to garage) from
      generating a spurious second departure Stage 1 event ~60s after the arrival.
      120s window comfortably covers full driveway traversal in either direction.

[✅] direction variable simplified: cam_dir if cam_dir != 'unknown' else ap_dir.
      Previously: cam_dir if NOT both_active else ap_dir.
      With ipcam01-based cam_dir, both_active ambiguity handled upstream; cam_dir is
      always arrival or departure (never unknown) when triggered by entrance/exit sensors.

[✅] RUNG 4 (service_person): removed 'perim' from zone check.
      Was: (perim or grounds or inside_any) and staff → caught ALL motion including street.
      Fix: (grounds or inside_any) and staff → perimeter falls through to RUNG 5/6.
      Effect: ipcam01/02 at street fires → visitor (RUNG 5) or perimeter_threat (RUNG 6),
      even when maid/gardener is on site. Gate/street is always active perimeter.

[✅] RUNG 5 (visitor): removed 'and not staff' condition.
      Was: perim_front and not gate and (allhome or not arriving) and not staff.
      Fix: perim_front and not gate and (allhome or not arriving).
      Effect: a genuine visitor at the front gate triggers visitor alert even when
      staff is on site. Previously fell through to family_movement (silent). Wrong.

[✅] Router service_person branch: always silent logbook (no push notification).
      Was: "👷 Staff on site" push notification with 10-min cooldown.
      Fix: logbook.log only. No push notification ever for service_person.
      Rationale: staff movement in grounds/inside is expected, not actionable.
      Perimeter/visitor/intruder paths still route through normal alert channels.

[✅] Stage 2 departure: suppress "Unknown departed" notification when staff on site.
      Was: "Unknown left at HH:MM" push notification even when maid was the one leaving.
      Fix: if who == 'Unknown' AND binary_sensor.staff_on_site == 'on' → logbook only.
      Real family departures will have named people; Unknown + staff = maid leaving.

[✅] Zone display format in reason attribute: human-readable names replace compact code.
      Was: zones: -G--- (P=perim, G=grounds, g=garage, m=main, b=beds, - = inactive).
      Fix: zones: Grounds (only active zones shown, joined with '+' e.g. 'Grounds+Main').
      'none' shown when no zone active.
```

SPRINT 9h — dogs_inside, auto/manual arming split, dogs_out 10min (2026-05-20)
```
[✅] Auto/manual arming boolean split:
      _manual booleans (inside_*_armed_manual) are dashboard toggles only — arming
      schedule NEVER writes them. Three new auto booleans added:
        input_boolean.inside_main_armed     — set by arming schedule (lounge/garage block)
        input_boolean.inside_garage_armed   — set by arming schedule (lounge/garage block)
        input_boolean.inside_bedrooms_armed — set by arming schedule (passage block)
      binary_sensor.inside_*_armed now = auto OR manual (either can arm the zone).
      Arming condition: nobody_home AND NOT dogs_inside (or bedtime for lounge/garage).

[✅] dogs_inside boolean added (input_boolean.dogs_inside):
      Dashboard toggle. When ON: inside cameras NOT armed by arming schedule
      → RUNG 8 cannot fire → no critical_intrusion for cam14/cam15.
      RUNG 8d: dogs_inside suppresses family_movement fallback (AcuSense + high_conf
      still escalates to intruder — outdoor cameras corroborating = real threat).
      dogs_out (outdoor pool suppression): unchanged — separate boolean, separate purpose.

[✅] dogs_out auto-off timer: 20min → 10min.

[✅] Classifier updated: dogs_inside variable added.
      RUNG 8: added `and not dogs_inside`.
      RUNG 8d: added `and not dogs_inside` to intruder escalation (AcuSense still wins).
```

SPRINT 9g — auto-arm S2 classifier booleans for empty-house inside detection (2026-05-20)
```
[✅] Inside cameras fire critical_intrusion → critical push notification when nobody home.
     Superseded by S9h (arming booleans renamed to auto/manual split).
      Root: arming schedule only set inside_cameras_armed + inside_cameras_passage_armed
      (snapshot capture guards). RUNG 8 in classifier reads inside_main_armed_manual,
      inside_garage_armed_manual, inside_bedrooms_armed_manual (S2 booleans) — these were
      NEVER auto-set, so RUNG 8 never fired for empty-house inside motion.
      Fix: arming schedule now sets ALL six booleans in sync:
        Lounge block (nobody home OR bedtime): arms inside_cameras_armed +
          inside_main_armed_manual + inside_garage_armed_manual
        Passage block (nobody home ONLY): arms inside_cameras_passage_armed +
          inside_bedrooms_armed_manual
      Result: nobody home → all S2 booleans ON → cam14/cam15 fire → RUNG 8 →
      critical_intrusion → router → critical push + Telegram + HA dashboard alert.
      Family returns → arming schedule turns all six OFF → cam14/cam15 → RUNG 3 →
      family_movement (silent).
      Dog caveat: dogs left inside with nobody home will trigger cam14/cam15 →
      critical_intrusion. Use guest_mode to suppress when dogs are home alone.
```

SPRINT 9f — threat_level rule 1 false critical (2026-05-20)
```
[✅] BUG-S35: sensor.security_threat_level rule 1 fired critical for cam15 (passage)
      at night when family IS home. Old: `inside and (nobody OR night)`. The OR night
      clause made any inside camera firing at night = critical regardless of occupancy.
      This fed binary_sensor.security_alert_active → HA dashboard showed 1 Critical Alert
      while push notifications (classifier-based) correctly said family_movement.
      Two pipelines out of sync: classifier said family_movement (silent), threat_level
      said critical (dashboard alert).
      Fix: rule 1 split into:
        1.  inside + nobody home → critical (property unoccupied)
        1b. inside + not nobody + night + all_family_in_bedrooms + inside_cameras_armed
            → critical (stay-mode, mirrors RUNG 2.5)
      Family at home + night + NOT all settled in bedrooms → rule 1 and 1b both fail
      → falls to low/elevated → no false dashboard critical alert.
      Stay-mode (bedtime, lounge fires) → rule 1b fires → critical, consistent with
      RUNG 2.5 push notification.
```

SPRINT 9e — stay-mode lounge intrusion + cam14/15 health alert fix (2026-05-20)
```
[✅] RUNG 2.5 added — stay-mode lounge intrusion:
      When: im=on (lounge fires) AND anyhome=on (family home, stay mode) AND night
            AND all_family_in_bedrooms=on (everyone settled in beds, AP-confirmed)
            AND inside_cameras_armed=on (auto-arming active at 23:00+).
      Result: critical_intrusion → critical push notification + Telegram.
      Placed BEFORE RUNG 3 so family_movement doesn't swallow this case.
      Trade-off: NVR cam14 has no AI — headlights/shadows may trigger.
      Mitigation: requires AP-confirmed bedrooms settled (all 4 family in Bedroom Zone).
      Re-added at user request 2026-05-20 (was removed in BUG-S18 fix).

[✅] Camera health false alert for cam14/cam15 fixed:
      Old: staleness fired for cam14/cam15 when sec_on AND daytime AND last_seen > 8h,
           regardless of occupancy. Triggered during day when nobody home (expected: no motion).
      Fix: added `and (i not in [8,9] or someone_home)` guard in all three template blocks
           (state, devices attribute, summary attribute). Indoor cameras (cam14=index 8,
           cam15=index 9) only flag as stale when someone_home=True. When nobody's home,
           zero motion in passage/lounge is correct — not a fault.

[✅] Passage arming confirmed correct (no change):
      inside_cameras_passage_armed armed when nobody_home ONLY.
      Will NOT arm at bedtime when family is home (bathroom trip protection).
      HA alerts seen at night = camera health issue (now fixed above), not arming issue.
```

SPRINT 9d — reload filter + inside-only fallback rung (2026-05-20)
```
[✅] BUG-S33: cam15 (passage, inside_bedrooms) classified as perimeter_threat on
      template reload when nobody home and zone not armed.
      Root: classifier final else→perimeter_threat was reached because:
      inside_any=True, anyhome=False, inside_bedrooms_armed=False → RUNG 8 fails
      (zone not armed) → fell to else: perimeter_threat.
      Fix 1: RUNG 8d added — inside_any + nobody home + not armed →
      family_movement (silent) unless IP cam or high confidence → intruder.
      Passage camera is NVR, low conf → family_movement. No notification fired.
      Fix 2: Router condition filters unknown→X transitions on reload/restart.
      classifier goes unknown→X every time template entities reload. Adding
      `trigger.from_state.state not in ['unknown','unavailable']` prevents the
      router from firing on the initial state population after a reload.

[✅] BUG-S34: Double visitor notification — cooldown check ran before last_visitor_event
      was set, allowing two simultaneous visitor events both to pass the 60s gate.
      Fix: last_visitor_event now set at START of visitor sequence (before the 45s delay),
      not at the end. Second visitor within 60s is now blocked at condition check time.
```

SPRINT 9c — gate-only idle rung, arrival snapshot fix, visitor grace 45s (2026-05-20)
```
[✅] BUG-S30: Gate-open-alone classified as perimeter_threat.
      Root: gate=on exits idle check; if no RUNG 1-8b matches and anyhome=off,
      classifier falls through to perimeter_threat. Happens when family presses
      remote from street (AP not connected yet, no cameras fired yet).
      Fix: RUNG 8c added — gate AND no perim AND no grounds AND no inside → idle.
      Gate alone (no camera signal) is never a threat, regardless of AP state.

[✅] BUG-S31: Arrival Stage 2 "Unknown" — phone connects to WiFi before ipcam03 fires.
      Root: Stage 1 before-snapshot (arrival_who_was_home) captured at T=0 AP state.
      ipcam03 entrance_valid has 2s delay_on; by the time it fires and Stage 1 runs,
      the phone is already showing as connected to home AP → included in snapshot →
      excluded from Stage 2 delta → "Unknown".
      Fix: Stage 1 arrival now writes who_was_home_snapshot (60s rolling, taken before
      car was in WiFi range) to arrival_who_was_home, not the instantaneous T=0 AP state.
      Fallback to who_home_now if snapshot is empty.

[✅] BUG-S32: Visitor grace period insufficient (20s) — slow gate openers (intercom,
      slow motor, fumbling for remote) take 25-45s to open gate after ipcam01 fires.
      Visitor notification fired before gate opened → false visitor on family arrival.
      Fix: grace period extended 20s→45s. Suppression condition strengthened:
      suppresses if gate currently open OR gate.last_changed < 90s OR
      ipcam03_entrance_valid.last_changed < 120s (vehicle confirmed entered driveway).
```

SPRINT 9b — Stage 1 delay removed + per-ipcam latest files + visitor grace (2026-05-20)
```
[✅] Stage 1 delay 90s→5s: ipcam03 zones now non-overlapping; direction comes from
      trigger entity (entrance_valid=arrival, exit_valid=departure) — no AP settle needed.
      Both-sensor ambiguity fallback still uses family_arriving/departing AP state.

[✅] Per-ipcam latest files: security_capture_each_camera_motion now saves
      /config/www/ipcamXX_latest.jpg for ipcam01-05 (not NVR cameras).
      NVR cameras recorded natively via NVR; ipcams are not on NVR, so this fills the gap.
      Served as /local/ipcamXX_latest.jpg for dashboard use.

[✅] Visitor grace period 20s: router visitor branch waits 20s then checks gate.
      If gate opened during grace period → suppress visitor notification (family arriving).
      Prevents "Visitor at gate" + "Arrival confirmed" false double-notification when
      ipcam01 fires as family car approaches before gate opens.
      Real visitors (gate stays closed for 20s+) still get immediate notification.

[✅] Camera config changes documented: ipcam01 (exit disabled, line A→B),
      ipcam03 (entrance at gate mouth, exiting at driveway bottom, intrusion Human/1s).
      Street-down camera recommendations added to Section 1.
      HA limitation on human vs vehicle separation documented in Section 1.
```

SPRINT 9 — BUG-S29 camera zone overlap + per-zone files (2026-05-20)
```
[✅] BUG-S29: ipcam03 regionentrance + regionexiting zones overlap — both fire
      for any vehicle movement in driveway regardless of direction.
      HA fix: merged security_gate_vehicle_stage1 automation replaces the two
      separate security_arrival_stage1_vehicle + security_departure_stage1_vehicle.
      New automation: triggers on either sensor, captures AP state at T=0,
      waits 90s for AP to settle and family_arriving/departing snapshot-delta to
      update, then reads direction from those sensors and fires one branch only.
      mode:single prevents dual-fire if both entrance+exit trigger together.
      Camera fix still recommended: reposition zones to non-overlapping areas
      (entrance at gate mouth, exiting at mid-driveway). See Section 1 camera table.

[✅] BUG-S22/S23 REAL FIX: per-zone snapshot files on disk.
      Previous fix (2026-05-19) wrote zone paths all pointing to security_latest.jpg
      — a later camera could overwrite the file content even though the path stored
      in the per-zone input_text was different.
      New fix: security_capture_best_snapshot saves a third snapshot to a
      zone-specific file (security_grounds_front_latest.jpg etc). Per-zone
      input_text stores the zone-specific path. Router reads a file that cannot
      be overwritten by a different zone's camera.
      Zone files: security_perimeter_front/rear_latest.jpg,
      security_grounds_front/rear_latest.jpg, security_inside_garage/main/bedrooms_latest.jpg

[✅] ipcam01 + ipcam03 camera zone configuration documented in Section 1
      with specific per-rule recommendations. Camera changes are physical
      (requires access to Hikvision web UI) and are the permanent fix for BUG-S29.
```

SPRINT 8 — BUG-S21–S27 structural fixes (2026-05-19)
```
[✅] BUG-S21: ipcam05 (rear boundary) firing as VISITOR — RUNG 5 now restricted to
      perim_front only. perim variable split into perim_front + perim_rear in
      security_event_classification. ipcam05 only feeds perim_rear; RUNG 6
      (perimeter_threat) handles rear boundary activity instead. BUG-S28 documented:
      ipcam02 dead — firmware V5.8.13 H13U incompatible with hikvision_next
      Smart Event discovery; perimeter_front = ipcam01 only until firmware fix.

[✅] BUG-S22/S23: Wrong-camera image in visitor/intruder alerts — global
      security_last_motion_image overwritten by any camera, not zone-specific.
      Fix: 7 per-zone input_text helpers added (security_image_perimeter_front etc.).
      security_capture_best_snapshot now writes the zone-appropriate slot in addition
      to the global security_latest.jpg. security_event_router reads zone-matched slot
      based on classifier zone_label attribute; fallback to global if slot empty.

[✅] BUG-S24/S25: Stage 2 "who" logic was a snapshot, not a delta — listed everyone
      currently home/away, not the people who newly arrived/departed.
      Fix: Stage 1 arrival writes input_text.arrival_who_was_home before-snapshot.
      Stage 1 departure writes input_text.departure_who_was_home before-snapshot.
      Stage 2 arrival computes: home_now AND NOT in arrival_who_was_home = new arrivals.
      Stage 2 departure computes: away_now AND WAS in departure_who_was_home = who left.

[✅] BUG-S26: family_arriving fired on intra-home AP roaming (Lounge→Kitchen triggers
      AP location last_changed → family_arriving=ON for 600s). Poisoned RUNG 1 with
      false arrival signals. Fix: family_arriving rewritten to snapshot-delta logic
      (compares current home set vs input_text.who_was_home_snapshot updated every 60s).
      Also: family_departing updated with same approach. Rolling snapshot automation
      presence_snapshot_who_home added to presence_boundary.yaml (1-min time_pattern +
      homeassistant:start trigger).

[✅] BUG-S27: Router visitor branch had no cooldown — fired on every classifier state
      change to visitor during a sustained gate approach (ipcam01 30s debounce cycle
      caused idle→visitor pulses). Fix: 60s cooldown added using last_visitor_event.

[✅] Stage 1 arrival/departure: now pass image: security_image_grounds_front (ipcam03
      driveway) instead of global security_last_motion_image. Same for Stage 2.

[✅] zone_label attribute updated: now returns 'perimeter front' vs 'perimeter rear'
      instead of generic 'perimeter'. Router zone_map updated to match new label strings.
```

SPRINT 7 — Notification spam + false intruder fixes (2026-05-18)
```
[✅] BUG-S18: False CRITICAL INTRUDER — INSIDE when family home at night
      Root cause: RUNG 3 excluded `inside_armed_active`, so cam14 firing after 23:00
      (lounge armed, family in bedrooms) bypassed family_movement and hit RUNG 8.
      NVR cam14 has no AI — headlights through kitchen window trigger cam04+cam14
      simultaneously, producing false critical_intrusion with `home=all`.
      Fix: RUNG 3 removes `not inside_armed_active` — when `anyhome=true`, ALL
      inside/grounds motion → family_movement regardless of arming state.
      RUNG 4 extended to cover `inside_any` for staff (maid legitimately inside).
      RUNG 8 now requires `not anyhome AND not staff`.
      Tradeoff: armed lounge camera while family sleeps no longer escalates to
      critical_intrusion. Accept until AcuSense inside cameras replace NVR.

[✅] BUG-S19: Staff on site spam — cam09 firing continuously with no cooldown
      Root cause: service_person branch in security_event_router had no cooldown.
      Every cam09 debounce cycle (25s) during maid work → separate notification.
      Fix: 10-minute cooldown added using last_visitor_event datetime.
      On fire: sets last_visitor_event. Next service_person skipped until 600s elapsed.

[✅] BUG-S20: Arrival spam — 3-5 "Family arriving home" notifications per arrival
      Root cause: security_event_router fired for each camera trigger during an
      arrival event (idle→arrival transition per camera). Stage1/stage2 already
      handle arrival notifications more accurately.
      Fix: arrival and departure branches in event_router now log-only (no notify).
      Stage1 fires "vehicle entering" on ipcam03_entrance_valid.
      Stage2 confirms who arrived 3.5min later via AP location.
      habiesie_arrival_detected still fired by security_arrival_stage1_vehicle (not router).

[✅] OPT-8.4: cam14/cam15 added to security_trigger_camera priority list at top.
      Fixes trig=none when only inside cameras fire (RUNG 8 critical_intrusion
      notification had no image). Inside cameras are highest priority — if they
      fire, that IS the event of interest.

[✅] FMT-01: reason attribute reformatted — equals → colons, flat string → 3 lines.
      Old: `zones=PG--- gate=open home=some arriving=yes departing=no staff=yes conf=medium trig=camera.ipcam01...`
      New: `zones: PG--- | gate: open | home: some\narriving: yes | departing: no | staff: yes\nconf: medium | cam: ipcam01_street_driveway_up`
      iOS/Android notification renders as 3 separate lines.
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
*Updated 2026-05-27 (S15): BUG-S47 stale image fix — inside camera zone slot write in security_capture_each_camera_motion;*
*  visitor branch 4s delay + visitor_img re-read to fix perimeter_front slot timing race;*
*  RUNG 5 split into perimeter_front (street activity, night/away only) + visitor (entrance_valid, genuine gate approach);*
*  security_logic.yaml + security_automations.yaml modified.*
*Updated 2026-05-26 (S14): BUG-S44 go2rtc replay filter (delay_on cam14/cam15); BUG-S46 RUNG 8 arriving guard;*
*  BUG-S45 partial: ipcam04 alarm deactivate REST command + dogs_out cancel automation;*
*  camera alarm Hikvision app visibility explained (expected behaviour, ISAPI output ≠ motion event).*
*Updated 2026-05-21 (S10): BUG-S36 departure/arrival direction fix (ipcam01 recency in Stage 1 cam_dir);*
*  mutual 120s suppression on Stage 1; RUNG 4 perim removed (perimeter always active); RUNG 5 not-staff*
*  removed; service_person router always silent; departure Stage 2 Unknown suppressed when staff on site;*
*  zone display format human-readable.*  
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
*Arrival/exit/visitor flow redesign — see Section 11 (IMPLEMENTED S2/S3 2026-05-17, classifier hardened S7 2026-05-18).*
*Next: Arrival/visitor redesign (Section 11); restore disabled overview cards; ipcam02 installer*

---

## Section 11: Arrival / Exit / Visitor Flow — Presence-First Classifier

> **Status:** ✅ IMPLEMENTED 2026-05-17 (S2 + S3). Core classifier rebuilt. Router wiring complete (S3 2026-05-17). Classifier hardened 2026-05-18 (S7 — RUNG 3/4/8 false-intruder fix).
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
