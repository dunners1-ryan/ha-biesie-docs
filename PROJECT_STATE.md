# HABiesie — Master Project State
> **Paste at the start of every new Claude session.**
> Last full audit: 2026-04-13 (Claude Code deep audit — all domains verified against live files)
> Update after every meaningful change. Commit: `./gitupdate.sh "docs: update PROJECT_STATE"`

---

## 🏠 System Identity

| Field | Value |
|---|---|
| Name | HABiesie |
| Location | Johannesburg, South Africa |
| Timezone | Africa/Johannesburg |
| Platform | Home Assistant OS (Supervisor enabled) |
| Network | 10.10.1.x LAN |
| Config path | `/config/` |
| Snapshot/image path | `/config/www/` → served as `/local/` |
| Backup script | `./gitupdate.sh "message"` |
| Validation | HA Developer Tools → YAML → Check Configuration |

---

## 🧱 Architecture Rules (NEVER VIOLATE)

1. **Package-based modular design** — all config in `packages/` via `!include_dir_named packages`
2. **Layering order within each package:**
   - `*_helpers.yaml` — input booleans, input numbers, counters, timers
   - `*_templates.yaml` — template sensors (derived values only)
   - `*_state.yaml` — state tracking and aggregation
   - `*_automations.yaml` — business logic
   - `*_core.yaml` — main integration config
3. **No new automations in `automations.yaml`** — legacy UI file, 3836 lines, DO NOT TOUCH
4. **Diagnostic sensors excluded from recorder** — maintain for all new diagnostic entities
5. **Trusted proxy** `172.30.0.0/16` for Docker reverse proxy HTTPS
6. **All notifications through central scripts** — never call `notify.*` directly
7. **Zone arming separates from zone motion** — aggregation sensors emit raw zone motion (ground truth). Arming gates are separate template sensors. Consumers (classifier, alerts, dashboard) read both. Do not merge motion + arming into a single "armed-and-firing" sensor (that collapses ground truth).

---

## 📦 Actual Package Structure (Verified 2026-04-16)

```
packages/
  power/          # 25 files — dual inverter, solar, battery, prepaid, energy, automations, statistics
  water/          # 20 files — tank lifecycle, safety aborts, audit
  alerts/         # 15 files — see ALERTS_CONTRACT.md for actual file list (alerts_batteries.yaml added 2026-05-27)
  lighting/       # 14 files — presence-aware and time-based scenes
  notifications/  # 12 files — routing, quiet hours, severity (includes water/power/presence/security/system scripts)
  security/       # 7 files  — cameras_core, cameras_processing, security_helpers,
                 #             security_core, security_logic, security_zones,
                 #             security_automations
  presence/       # 6 files  — presence_helpers, presence_core, presence_confidence,
                 #             presence_boundary, presence_validation, presence_trust (migrated from context/ 2026-04-30)
  context/        # 2 files  — context_global, context_night (context_presence + context_schedules deleted 2026-04-30/28)
  core/           # 3 files  — core_helpers (startup guard), ha_monitoring, sensor_smoothing
  network/        # 3 files  — network_helpers (WAN health, UniFi speed, latency, jitter, NOC status);
                 #             network_ups (EcoFlow River Pro UPS monitoring — added 2026-05-27);
                 #             network_nas (Synology NAS graceful shutdown + WoL restore — added 2026-05-28)
  alerts/ already counted above; also covers: network, temperature, system_health, media, doors, water, security, presence, power
  backup/         # 1 file   — github.yaml (5AM git backup, script, status sensor)
  integrations/   # 1 file   — sonoff.yaml (Sonoff/EWElink config)
  office/         # 1 file   — office_helpers.yaml (printer cartridge monitoring)
  weather/        # 2 files  — weather_core.yaml, weather_helpers.yaml (OWM health tracking)
  sensors/        # 1 file   — filter.yaml (sensor smoothing utilities)
  load_shedding/  # 2 files  — load_shedding_templates.yaml (migrated from power/ 2026-04-21)
                 #             load_shedding_automations.yaml (3 warning automations migrated 2026-04-29)
  admin/          # 1 file   — tablets.yaml (screen brightness management for dashboard tablets — added 2026-05-27)
```

**Important:** Several `*_CONTEXT.md` documents list incorrect filenames. The `*_CONTRACT.md`
files are authoritative for actual file inventory.

---

## ⚡ Hardware Summary

### Inverters
- **Dual Master/Slave Sunsynk** — Master: grid/losses/BMS, Slave: PV/load/battery
- Aggregated in `power_core.yaml` into unified sensors
- Solar forecast: Solcast, cached in `solcast_solar/`

### Cameras (verified 2026-05-17 — 7 NVR + 5 IP active)
- NVR: Hikvision DS-7116HGHI-F1 (16-channel hybrid DVR, analog 1080p, no AI)
- Active NVR (7): cam04, cam05_inside_garage, cam07, cam09, cam12, cam14, cam15
- Active IP AcuSense (5): ipcam01, ipcam02, ipcam03, ipcam04, ipcam05 — fully wired 2026-05-08
- NVR roadmap: existing NVR analog cameras will be progressively replaced by IP cameras. Empty NVR channels (formerly cam02/03/11/13/16-future) were removed 2026-05-17. No new analog cameras will be added to this NVR.
- Deprecated 2026-05-07: cam01_street_driveway (→ipcam01+02), cam10_pool_bar (→ipcam04)
- Uninstalled 2026-05-08: cam06_front_entrance (→ipcam03)
- All deprecated/uninstalled entity definitions purged from packages/security/ 2026-05-17
- cam05 renamed: cam05_front_driveway → cam05_inside_garage (2026-05-17)
- Entity prefix: `ipcam` for all IP cameras (ipcam01–ipcam05) — standardised 2026-05-08
- Streaming: go2rtc (timeout instability with NVR)
- See SECURITY_CONTRACT.md Section 3 for the full zone map
- Frigate: **planned, NOT implemented**
- Integration: `hikvision_next` (custom, unstable)

### Water
- Borehole pump: `switch.borehole_pump`
- Depth sensor: Tuya (`sensor.water_tank_depth_validated`)
- Pressure pump: `switch.water_pressure_pump`

### Network
- Gateway: UniFi Dream Machine | APs: 5x UniFi | WAN Router: ASUS ROG

---

## 🔒 Locked Entity Names (DO NOT RENAME)

### Camera Entities
```
camera.camXX_location_description             e.g. camera.cam04_car_port_front
binary_sensor.camXX_location_motiondetection  NVR raw signal
# 2026-05-17: cam05 renamed cam05_front_driveway → cam05_inside_garage
# NVR hardware entity cam05_front_driveway_motiondetection name unchanged (integration-provided)
# Active NVR: cam04/05_inside_garage/07/09/12/14/15 + IP: ipcam01-05
# DO NOT reintroduce: cam01_street_driveway, cam06_front_entrance, cam10_pool_bar
binary_sensor.camXX_location_motion_valid     debounced output
```

### Core Security Sensors
```
sensor.security_trigger_camera
sensor.security_event_classification
sensor.security_correlation            ← BUG: missing unique_id
sensor.security_threat_level
sensor.security_threat_score
sensor.security_movement_confidence
sensor.security_movement_path
sensor.security_mode
sensor.security_lighting_intent
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
input_boolean.inside_cameras_armed              ← auto-managed by arming automation
input_boolean.inside_cameras_schedule_override  ← dashboard override, force-arms cam14/cam15
```

### Presence Persons
```
sensor.ryan_ap_location      sensor.vicky_ap_location
sensor.luke_ap_location      sensor.tayla_ap_location
person.ryan_dunnington        person.vicky_dunnington
person.luke_dunnington        person.tayla_dunnington
device_tracker.ryan_iphone_tracker   ← NOTE: _tracker suffix is correct
```

### Trust Model — CRITICAL RULE
```
# ALWAYS USE THESE (derived, auto-set by schedules):
binary_sensor.staff_on_site
binary_sensor.low_trust_present

# NEVER USE THESE in automations (manual, never auto-set):
input_boolean.staff_on_site      ← BUG IV-01 FIXED 2026-04-14 (4 security automations corrected)
input_boolean.low_trust_present
```

### Door Alert Inputs (BUG-A06 — added 2026-04-16)
```
input_number.tier3_door_evening_start          # 21.0 h — evening mode start for lounge/bar
input_number.entry_door_night_escalation_minutes # 5 min — tier-2 night escalation
sensor.door_alert_context                       # SINGLE SOURCE — replaces doors_open_alert_severity
# sensor.doors_open_alert_severity DELETED
```

### Water Entities (expanded 2026-04-16)
```
sensor.water_refill_blocked_reason              # "none" or human-readable block reason
binary_sensor.water_borehole_fault_alert_active # ON when faults_today >= 3
binary_sensor.water_borehole_critical_fault_active # ON when faults_today >= 5
counter.water_borehole_faults_today             # daily fault counter — YAML only (storage duplicate removed 2026-04-21)
counter.water_borehole_faults_this_week         # weekly fault counter — YAML only (storage duplicate removed 2026-04-21)
alert.water_borehole_fault                      # level 2 fault alert
alert.water_borehole_critical_fault             # level 3 fault alert
```

### Presence Alert Entities (B1 — added 2026-04-16)
```
binary_sensor.unknown_ap_alert_active           # delay_on 5 min, delay_off 2 min
binary_sensor.occupancy_anomaly_alert_active    # delay_on 15 min — info only
sensor.presence_alert_context                   # warning/critical by AP duration
alert.presence_alert                            # STD_Alerts, repeat 60/120 min
input_boolean.presence_alert_notify             # pipeline toggle
```

### Security Dedup Gate (triple-fire fix — added 2026-04-16)
```
binary_sensor.security_intruder_active          # ON when correlation = intruder/intruder_high
                                                # delay_off: 30s — prevents re-fire on flap
```

### Alert Architecture Sensors
```
sensor.global_alert_context
sensor.alert_device_entities      ← SINGLE SOURCE OF TRUTH
sensor.active_alert_entities
sensor.total_critical_alert_devices
sensor.total_warning_alert_devices
sensor.total_info_alert_devices
sensor.total_active_alerts
sensor.total_acknowledged_alerts
sensor.total_unacknowledged_alerts
```

### Dashboard Battery Alert Entities (added 2026-05-27)
```
# alerts_batteries.yaml
input_boolean.dash_battery_alert_notify           ← pipeline suppress toggle
input_number.dash_battery_warning_threshold       ← 30% low alert trigger
input_number.dash_battery_critical_threshold      ← 15% critical severity
input_number.dash_battery_overcharge_threshold    ← 95% overcharge trigger
input_number.honor10_dash_battery_capacity_wh     ← 39.1 Wh (Honor Pad 10, HEY3-W00 — 10100 mAh × 3.87V)
input_number.honorx7_dash_battery_capacity_wh     ← 27.2 Wh (Honor Pad X7, JMS-W09 — 7020 mAh × 3.87V)
sensor.honor10_dash_battery_time_remaining_est    ← minutes remaining (-1 = charging/unavailable)
sensor.honorx7_dash_battery_time_remaining_est    ← minutes remaining (-1 = charging/unavailable)
binary_sensor.honor10_dash_battery_low            ← delay_on 1 min, < warning_threshold AND not charging
binary_sensor.honorx7_dash_battery_low            ← same for X7
binary_sensor.honor10_dash_battery_overcharge_active ← delay_on 2 h, ≥95% while charging
binary_sensor.honorx7_dash_battery_overcharge_active ← same for X7
binary_sensor.dash_battery_alert_active           ← master trigger (any low OR overcharge)
sensor.dash_battery_alert_context                 ← canonical context sensor for aggregator
alert.dash_battery_alert                          ← 30/60 min repeat, STD_Alerts

# tablets.yaml (admin package) — screen brightness
input_number.dash_brightness_day                  ← 150 (0–255 scale)
input_number.dash_brightness_night                ← 20
input_number.dash_brightness_away                 ← 40
automation.tablets_brightness_night_dim           ← night_mode ON → night brightness
automation.tablets_brightness_morning_restore     ← night_mode OFF → day (or away if nobody home)
automation.tablets_brightness_away_dim            ← nobody_home ON → away brightness (if not night)
automation.tablets_brightness_return_restore      ← nobody_home OFF → day brightness (if not night)

# Entities to ENABLE in HA Settings → Entities (currently disabled by integration):
#   binary_sensor.honor10_dash_interactive  — screen on/off (useful for future proximity logic)
#   binary_sensor.honorx7_dash_interactive  — screen on/off
#   binary_sensor.honorx7_dash_doze_mode    — device doze/sleep state
```

### NAS Protection Entities (added 2026-05-28)
```
# network_nas.yaml — Synology graceful shutdown + WoL restore
input_boolean.ups_nas_auto_shutdown_enabled    ← cancel gate; toggle OFF to abort pending shutdown
input_boolean.ups_nas_was_shutdown             ← flag set at shutdown; gates WoL on HA restart
input_number.ups_nas_shutdown_warn_minutes     ← Stage 1 threshold (default 15 min)
input_number.ups_nas_shutdown_grace_minutes    ← grace between warning and forced shutdown (default 5 min)
input_number.ups_pi_shutdown_wait_minutes      ← wait after NAS shutdown before Pi shutdown (default 4 min)
binary_sensor.ups_nas_shutdown_imminent        ← ON when on battery AND runtime < warn threshold

# External entities (not HA-defined — Synology DSM + WoL integrations)
button.guardians_shutdown                      ← Synology DSM shutdown
button.wol_synology_nas_00_11_32_ad_af_a5      ← WoL magic packet to LAN 1 NIC (MAC 00:11:32:ad:af:a5)
                                                  broadcast_address: 192.168.1.255 (fixed 2026-05-28 — was wrong unicast IP)
```

### Power Core Sensors
```
sensor.inverter_battery_soc      sensor.inverter_load_power
sensor.inverter_grid_power       sensor.inverter_pv_power
sensor.inverter_battery_power    sensor.grid_energy_import_today
sensor.inverter_today_production group.inverter_grid
group.inverter_battery_state     sensor.prepaid_units_left_safe
```

### Power Runtime & Battery Sensors (battery_runtime.yaml)
```
sensor.ss_battery_capacity             ← program-aware target SOC
sensor.ss_soc_battery_time_left        ← runtime seconds to target
sensor.ss_soc_battery_time_left_friendly ← human-readable runtime
sensor.markdown_ss_discharge_time      ← ETA clock (HH:MM)
sensor.battery_runtime_severity        ← charging/ok/warning/critical
sensor.battery_minutes_remaining_safe  ← numeric minutes (automation-safe)
sensor.battery_runtime_confidence      ← high/medium/low/none
binary_sensor.battery_runtime_unreliable ← ON when data is bad
# NOTE: entity prefix is ss_* (NOT ssa_* — those were a dead legacy integration)
```

### Power Diagnostic Sensors (renamed 2026-04-21)
```
sensor.inverter_device_1_since_last_update  ← was inverter_1_device_since_last_update
sensor.inverter_device_2_since_last_update  ← unchanged
# unique_id updated — HA will create new entity; old entity_id becomes orphaned in registry
```

### Load Control (load_control.yaml — added 2026-04-21)
```
input_boolean.load_control_geyser_enabled    ← OFF turns off + blocks geyser
input_boolean.load_control_pool_enabled      ← OFF turns off + blocks pool pump
input_boolean.load_control_borehole_enabled  ← OFF turns off + blocks borehole (WARNING level)
```
Devices: switch.geyser_heat_pump_switch, switch.pool_pump_switch, switch.borehole_pump
# Renamed 2026-04-22: _switch_1 → _switch for both geyser and pool pump (HA entity registry + all YAML + dashboards)
✅ DONE 2026-04-29: load_control_borehole_enabled added to binary_sensor.water_refill_allowed state condition (water_templates.yaml:299)

### Energy Orchestrator (power_helpers.yaml + energy_state.yaml — added E1 2026-06-14)
```
# State sensor — 6-state priority ladder
sensor.energy_orchestrator_state         ← loadshedding_critical/loadshedding/critical/conserve/surplus/normal
sensor.orchestrator_decision_reason      ← human-readable string explaining current state

# Master gate
input_boolean.orchestrator_enabled       ← when OFF, orchestrator always returns 'normal' (initial: true)

# SOC threshold ladder (all UI-adjustable, all read by template sensors via states()|float(default))
input_number.orchestrator_emergency_soc_threshold        ← default 15%
input_number.orchestrator_critical_soc_threshold         ← default 25%
input_number.orchestrator_conserve_soc_threshold         ← default 50%
input_number.orchestrator_pre_shed_soc_threshold         ← default 80%
input_number.orchestrator_conserve_degraded_soc_threshold ← default 60% (degraded solar branch)
input_number.orchestrator_surplus_export_threshold       ← default 1000W
input_number.orchestrator_target_soc_by_sunset           ← default 90% (future use)
input_number.orchestrator_load_first_soc_threshold       ← default 65% (future use)
input_number.orchestrator_pre_shed_hours_warning         ← default 3h

# Pool winter scheduling
input_number.pool_winter_start_hour                      ← default 10h (future use by pool automation)
```

### Inverter Sync (power_helpers.yaml + power_automations.yaml — added E1 2026-06-14)
```
input_boolean.inverter_sync_status       ← ON=in sync; OFF=mismatch; written by inverter_sync_check
automation.inverter_sync_check           ← triggers on select.inverter_1_energy_pattern change; 90s delay
script.force_inverter_sync               ← dashboard-callable; copies INV1 energy_pattern → INV2
# ⚠️ Partial: only energy_pattern compared. work_mode/program_N_charging/inv2_soc entities unconfirmed.
```

### Three-Zone Inside Arming (S2 2026-05-17; S9h 2026-05-20)
```
binary_sensor.security_inside_garage_motion   ← raw garage zone motion (cam05 + DSC hook)
binary_sensor.security_inside_main_motion     ← raw main-house zone motion (cam14 + DSC hook)
binary_sensor.security_inside_bedrooms_motion ← raw bedroom passage zone motion (cam15 + DSC hook)
binary_sensor.security_inside_house_motion    ← backward-compat union of the three (updated S2)

# Arming gates — binary_sensor.inside_*_armed = auto OR manual (either arms the zone)
binary_sensor.inside_garage_armed    ← auto OR manual
binary_sensor.inside_main_armed      ← auto OR manual
binary_sensor.inside_bedrooms_armed  ← auto OR manual (bedrooms: nobody home only)

# Auto booleans — written by security_inside_cameras_arming automation
input_boolean.inside_garage_armed    ← auto: ON when nobody home (or bedtime for lounge/garage)
input_boolean.inside_main_armed      ← auto: ON when nobody home (or bedtime)
input_boolean.inside_bedrooms_armed  ← auto: ON when nobody home AND NOT dogs_inside

# Manual booleans — dashboard override only, never auto-set
input_boolean.inside_garage_armed_manual    ← manual override / DSC stub (S6+)
input_boolean.inside_main_armed_manual      ← manual override / DSC stub
input_boolean.inside_bedrooms_armed_manual  ← manual override / DSC stub
```

### Dogs / Occupancy Suppression Booleans (S9h — 2026-05-20)
```
input_boolean.security_dogs_out  ← suppress rear/pool alarm during garden walk (10min auto-off)
input_boolean.dogs_inside         ← suppress inside camera alerts when dogs home alone
                                    (manual — set before leaving, clear on return)
```

### Security Outdoor Corroboration (S11 — 2026-05-22)
```
binary_sensor.security_external_motion_recent  ← ON for 5min after any perimeter/grounds
                                                   camera fires. Used by RUNG 8 to require
                                                   outdoor path corroboration.
                                                   Defined in security_logic.yaml.
binary_sensor.security_front_approach_recent   ← ON for 5min after a FRONT-APPROACH camera
                                                   fires (ipcam01/02, ipcam03, cam04, cam07,
                                                   cam09, ipcam05). Excludes cam12+ipcam04
                                                   (pond/pool — rear NVR cameras that fire
                                                   for animals). Used by RUNG 2.5 instead of
                                                   ext_recent. Added S16 2026-06-14.
```

### Security Classifier Presence Signals (S2 — 2026-05-17; S8 logic updated 2026-05-19)
```
binary_sensor.family_arriving    ← someone in home zone NOT in 60s snapshot (snapshot-delta, S8)
                                   CHANGED from recency-only (BUG-S26 fix)
binary_sensor.family_departing   ← someone Away who WAS in 60s snapshot (snapshot-delta, S8)
binary_sensor.all_family_home    ← ALL family APs in home zones (visitor disambiguation — BUG-P13)
```

### Per-Zone Snapshot Image Helpers (added S8 — 2026-05-19)
```
input_text.security_image_perimeter_front  ← last snapshot from ipcam01/02 (street cameras)
input_text.security_image_perimeter_rear   ← last snapshot from ipcam05 (rear boundary)
input_text.security_image_grounds_front    ← last snapshot from ipcam03/cam04/cam07 (front grounds)
input_text.security_image_grounds_rear     ← last snapshot from ipcam04/cam09/cam12 (rear grounds)
input_text.security_image_inside_garage    ← last snapshot from cam05 (garage interior)
input_text.security_image_inside_main      ← last snapshot from cam14 (lounge)
input_text.security_image_inside_bedrooms  ← last snapshot from cam15 (passage)
input_text.security_image_arrival_locked   ← locked at Stage 1 T+5s; Stage 2 reads this (BUG-S41 fix)
# Router reads zone-matched slot; fallback to security_last_motion_image if slot empty
```

### Presence Snapshot Helpers (added S8 — 2026-05-19)
```
input_text.who_was_home_snapshot  ← rolling 60s "Ryan,Vicky" list; feeds family_arriving/departing
input_text.arrival_who_was_home   ← point-in-time snapshot at Stage 1 arrival; Stage 2 delta reads
input_text.departure_who_was_home ← point-in-time snapshot at Stage 1 departure; Stage 2 delta reads
input_datetime.last_critical_event ← 90s cooldown gate for critical_intrusion branch (BUG-S39 fix)
```

### UI-Managed Utility Meters (DO NOT recreate in YAML)
```
sensor.solar_to_battery_energy_today  ← utility_meter, created 2026-03-20
sensor.grid_to_battery_energy_today   ← utility_meter, created 2026-03-20
```

### Water Core Entities
```
sensor.water_state
sensor.water_tank_level            ← friendly name "Water Tank Level %" — entity_id has % stripped
sensor.water_tank_depth_validated
switch.borehole_pump
input_boolean.water_refill_cycle_active
input_boolean.water_refill_aborted_due_to_safety   ← can be stuck ON after Tuya reconnect false-aborts
binary_sensor.water_refill_allowed
input_boolean.water_alert_notify   ← suppress water alert pipeline
input_boolean.water_refill_force_override ← bypass solar window/SOC/last-sun gates; safety hard-stops still apply (added 2026-06-14)
```

### Water Demand Planning Entities (added 2026-05-25)
```
sensor.water_target_depth_tomorrow             ← stop depth for tonight's refill; reads tomorrow's select
  attributes: demand_type, tomorrow_day
sensor.water_demand_today                      ← today's demand type string (dashboard)
sensor.water_refill_blocked_reason             ← updated: now compares against water_target_depth_tomorrow

# Day demand selectors — one per day, options: Normal | Wash/Clean | Irrigation | Pool
input_select.water_demand_monday
input_select.water_demand_tuesday
input_select.water_demand_wednesday
input_select.water_demand_thursday
input_select.water_demand_friday
input_select.water_demand_saturday
input_select.water_demand_sunday

# Target depth input_numbers (entity_ids unchanged; initials + names changed):
input_number.water_target_depth_normal         ← 1.00m (was 1.55m "Normal Day")
input_number.water_target_depth_partial        ← 1.20m "Wash/Clean" (was 1.65m "Partial Irrigation")
input_number.water_target_depth_irrigation     ← 1.25m NEW (Irrigation days)
input_number.water_target_depth_full           ← 1.60m "Pool Day" (was 1.85m "Full Irrigation")

# Season preset scripts
script.water_demand_set_summer_profile
script.water_demand_set_winter_profile

# Safety hard-stop automation (added 2026-05-25 — water_safety.yaml)
# automation id: water_safety_battery_hard_stop
# Triggers on SOC < water_battery_soc_hard_stop; no safety-state exemption

# Daily target stop automation (added 2026-05-25 — water_tank_refill_control.yaml)
# automation id: water_stop_at_daily_target
# Stops pump when depth reaches sensor.water_target_depth_tomorrow

# REMOVED 2026-05-25 — 21 demand planning input_booleans (were defined but never read by any automation):
# irrigation_full_monday/tuesday/wednesday/thursday/friday/saturday/sunday
# irrigation_partial_monday/tuesday/wednesday/thursday/friday/saturday/sunday
# washing_heavy_monday, washing_heavy_thursday
# house_cleaning_monday, house_cleaning_thursday
# washing_partial_friday, washing_partial_saturday, washing_partial_sunday
```

---

## 🔴 DO NOT TOUCH Without Full Review

- Security event router automation
- Notification central scripts (`script.notify_*_event`)
- Water lifecycle capture start/end automations
- Presence resolver boundary logic
- Power core aggregation sensors

---

## 📋 Domain States (Post-Audit 2026-04-16)

| Domain | Status | Notes (as of 2026-04-16) |
|---|---|---|
| 🛡️ Security | S16 + dogs notification 2026-06-14 | **S16 (2026-06-14):** BUG-S48 — arrival image shows cam07 kitchen instead of ipcam03 driveway; fixed: Stage 1 arrival lock now reads `ipcam03_driveway_history` (per-camera, ipcam03-only) preferentially over shared `security_image_grounds_front` slot (cam04+cam07 share it). BUG-S49 — departure confirmed time was T+3.5min not actual departure; fixed: `depart_time` captured at Stage 1 T=0, passed in `habiesie_departure_detected` event data as `left_at`, Stage 2 captures before its delay and uses in message. BUG-S50 — RUNG 2.5 false critical at night with `home:all`; root: cam12 (back pond NVR) fires for frogs/moonlight → ext_recent ON for 5min → cam14 catches any light change → critical_intrusion; fixed: new `binary_sensor.security_front_approach_recent` (5min delay_off, excludes cam12+ipcam04) added to security_logic.yaml; RUNG 2.5 uses this instead of ext_recent; ext_recent retained for RUNG 8. BUG-S51 — inside alerts when Luke home; root: Luke AP drops briefly → anyone_connected_home OFF (2min) → passage arms → cam12+cam15 → RUNG 8; fixed: anyone_connected_home delay_off increased 2min→5min in presence_core.yaml. cam14 zone reduction (monitor light area) applied camera-side by user — helped reduce genuine NVR motion from screen light changes (not fixable in code). **2026-05-28 (ipcam02 BUG-S28 re-attempt):** Camera fully restarted, reconfigured with Smart Events (Intrusion/Line Crossing/Region Entrance), deleted from HA, and re-added. Still only `scenechangedetection` discovered — AcuSense sensors (fielddetection/linedetection/regionentrance) not appearing. BUG-S28 remains open. `ipcam02_street_driveway_down_motion_valid` now includes scenechangedetection as fallback trigger (committed 2026-05-27). **S15 (2026-05-27):** BUG-S47 — stale image in critical_intrusion/visitor notifications fixed: RUNG 2.5 snapshot now 4s delayed after motion capture; visitor reads zone-matched image slot (not global fallback). RUNG 5 three-way split: `perimeter_front` (ipcam01/02 without gate approach), `visitor` (entrance_valid OR gate_activity), `gate_activity` (gate opened + street motion without entrance_valid). ipcam03 AcuSense dog misclassification noted (dogs trigger intrusion zone → false visitor). **S14 (2026-05-26):** BUG-S44 — go2rtc reconnect replays cam14/cam15 as false motion; fixed: delay_on 3s added to cam14_lounge_motion_valid + cam15_passage_motion_valid. BUG-S46 — RUNG 8 fires critical_intrusion during family arrival (AP transition race); fixed: `not arriving` guard added. BUG-S45 partial — ipcam04 alarm fires camera-side independently of HA; rest_command.ipcam04_deactivate_alarm added; dogs_out_cancel_alarm automation added. **S11 (2026-05-22):** anyone_connected_home delay_off 2min added (false intruder on departure). RUNG 7 + 8d: `not departing` guard. `binary_sensor.security_external_motion_recent` (5min delay_off) — outdoor corroboration sensor. RUNG 2.5 + RUNG 8: require `ext_recent OR ip_cam OR high_conf` — NVR inside cameras alone no longer escalate to critical without outdoor precedent. zone_label: 'grounds front'/'grounds rear' split (trigger camera based). Router reason/zone: from trigger.to_state.attributes (not stale live re-evaluation). Visitor: 45s grace removed → immediate notification, 30s cooldown. **S10 (2026-05-21):** BUG-S36 — departure misclassified as arrival fixed (ipcam01 recency in Stage 1 cam_dir: entrance_valid = arrival only if street camera fired ≤120s ago). Mutual 120s suppression added to Stage 1 (second sensor of a traversal blocked). RUNG 4 (service_person): `perim` removed — perimeter always active regardless of staff schedule. RUNG 5 (visitor): `not staff` removed — visitor fires at gate even when maid on site. Router service_person: always silent logbook (no push notification). Stage 2 departure: Unknown suppressed when staff_on_site. Zone display: human-readable names. **S9h (2026-05-20):** dogs_inside boolean (suppresses interior alerts when dogs home alone). Auto/manual arming split — arming schedule writes `inside_*_armed` (auto); `_manual` variants dashboard-only. binary_sensor.inside_*_armed = auto OR manual. dogs_out timer 20→10min. **S9g (2026-05-20):** Auto-arm S2 classifier booleans with capture booleans in arming schedule. Nobody home → `inside_main_armed_manual` + `inside_bedrooms_armed_manual` + `inside_garage_armed_manual` all ON → RUNG 8 fires critical_intrusion for inside cameras → critical push notification. Dog caveat: use guest_mode when dogs home alone. **S9f (2026-05-20):** threat_level rule 1 false critical fixed — `inside AND (nobody OR night)` changed to `inside AND nobody` (rule 1) plus `inside + stay-mode` (rule 1b). Cam15 at night with family home no longer triggers critical HA dashboard alert. **S9e (2026-05-20):** Stay-mode lounge intrusion (RUNG 2.5) — after full bedtime, lounge fires → critical_intrusion → push notification. Camera health false alert for cam14/cam15 fixed — staleness only flagged when someone is home (indoor cameras legitimately quiet when empty). Passage arming confirmed correct (nobody_home only). **S9d (2026-05-20):** Reload filter on event router (unknown→X transition suppressed). Inside-only fallback rung (cam15 reload flicker → family_movement not perimeter_threat). Double visitor cooldown race fixed (last_visitor_event set before delay). **S9c (2026-05-20):** Gate-open-alone false perimeter_threat fixed (RUNG 8c: gate+no-motion→idle). Arrival Stage 2 "Unknown" fixed (arrival_who_was_home now uses 60s rolling snapshot, not T=0 AP state which included the arriving phone). Visitor grace period 20s→45s + stronger suppression (gate-age + ipcam03 entrance check). **S9b (2026-05-20):** Stage 1 delay 90s→5s (camera zones non-overlapping, direction from trigger entity). Per-ipcam latest snapshot files added (ipcam01-05, `/local/ipcamXX_latest.jpg`). Visitor grace period 20s (gate-open check prevents false visitor before family arrival). ipcam01/ipcam03 camera configs documented and updated. Street-down camera recommendations documented. HA human/vehicle separation limitation documented. **S8 (2026-05-19): BUG-S21–S27 structural fixes.** ipcam05 (rear boundary) was triggering visitor classification (BUG-S21 — RUNG 5 now front-perimeter only). Wrong-camera image in notifications fixed via 7 per-zone image helpers + zone-aware router lookup (BUG-S22/S23). Stage 2 arrival/departure now uses delta logic: Stage 1 snapshots who was home before the event; Stage 2 computes who newly arrived/departed (BUG-S24/S25). family_arriving/departing rewritten from recency-only to snapshot-delta — AP roaming no longer triggers false arrival (BUG-S26). Visitor branch cooldown 60s added (BUG-S27). zone_label attribute now distinguishes perimeter front vs perimeter rear. New entities: 7 `input_text.security_image_*` per-zone helpers (security_helpers.yaml); `input_text.who_was_home_snapshot`, `arrival_who_was_home`, `departure_who_was_home` (presence_helpers.yaml); automation `presence_snapshot_who_home` (presence_boundary.yaml). ipcam02 confirmed dead (BUG-S28 — firmware V5.8.13 H13U incompatible with hikvision_next Smart Events; perimeter_front = ipcam01 only in practice). Pool alarm tiered 2026-05-10. **Notifications silent since 2026-04-17 — FIXED 2026-05-08.** Root cause 1: `security_event_router` had no `elevated` branch — single-camera daytime events (score 25–74) were silently dropped. Root cause 2: cam07 excluded from confidence front tier — `security_intruder_active` never fired for cam07-only events. Both fixed. Camera fleet restructured: cam06 removed (uninstalled); cam01/cam10 deprecated; cam05 moved to inside garage (grounds_front zone); all 5 IP cameras fully wired (ipcam01–ipcam05); perimeter_rear now active for first time (ipcam05). AcuSense entrance/exit debounce sensors added. Visitor trigger corrected to street cameras only (ipcam01/02). Arrival uses ipcam03_entrance_valid as primary. Entity naming standardised: all IP cameras now use `ipcam` prefix (was mixed camip/ipcam). 2026-05-10: `security_pool_alarm_trigger` tiered threat gate added — arms from 20:00 (not waiting for night_mode ~22:30); critical always fires; warning suppressed when family asleep (night_mode ON + home). |
| ⚡ Power | Updated 2026-06-14 E5 | **Session E1 (2026-06-14):** Energy Orchestrator core implementation. sensor.energy_orchestrator_state rebuilt: old 5-state basic sensor replaced with 6-state priority ladder (loadshedding_critical/loadshedding/critical/conserve/surplus/normal). sensor.orchestrator_decision_reason added. 12 new input helpers (orchestrator_enabled + 9 SOC/threshold input_numbers + pool_winter_start_hour + inverter_sync_status). binary_sensor.water_refill_allowed updated: orchestrator gate added — blocks refill when state in [critical, loadshedding, loadshedding_critical]. Inverter sync: automation.inverter_sync_check + script.force_inverter_sync added (partial — energy_pattern only; other entities unconfirmed). Spec correction: battery_state_health has no 'warning' state — conserve branch uses 'low' only. Degraded SOC floor parameterised: input_number.orchestrator_conserve_degraded_soc_threshold (60%). ha core check passed. Template reload + helper reload completed — functional check: state=normal, SOC 64%, battery healthy. | **2026-05-27:** BUG fix — pyscript sync_power_groups.py was wiping group.flexible_power_loads and group.critical_power_loads to [] on every startup, overriding YAML definitions. Removed the two wipe lines; pyscript now only maintains real_power_loads (auto-detected). YAML in power_templates.yaml is now the sole owner of flexible/critical groups. Plug change: Water Cooler Plug → LG Combo Washer Plug (2026-05-25); sensor.lg_combo_washer_plug_power added to known_power_loads, flexible_power_loads, house_laundry_power_sensors. Issue 16 in POWER_CONTRACT resolved. **Session H (2026-05-25):** Eskom outage triggered BMS protection — pool pump drew ~3.9kW at 07:25 (grid offline, SOC ~38%), causing BMS shutdown and house trip at 07:45. Fixed: (1) pool pump now blocked during grid outage when SOC < `grid_offline_soc_min_pool` (60%, Branch 2a) — bypasses minimum run time; (2) pool pump now blocked in last-sun-slot (14:00–16:00) when SOC < `sensor.last_sun_soc_target` (80%/90% by season, Branch 2b); (3) same two conditions added as gates to Branch 5 turn-on; (4) new triggers: `group.inverter_grid` state change + `sensor.inverter_battery_soc` below threshold. Borehole: `binary_sensor.water_refill_allowed` updated with last-sun-slot gate (SOC below target → block, but bypassed if tank ≤20%). New shared helpers added to power_helpers.yaml (9 new input_numbers). New template sensors: `sensor.last_sun_soc_target` (80%/90% season-aware), `sensor.geyser_target_run_hours_today` (2/3.5h season-aware). Geyser: helpers added but automation NOT implemented — see POWER_CONTRACT Sprint 5 for design questions. Session G (2026-04-28): pool_pump_solar_control two bugs fixed — (1) no 16:00 hard-stop: pump ran until 18:05 when manually stopped; fixed by adding "16:00:00" trigger (id: end_of_day) + Branch 0 unconditional turn-off, removing before:"16:00:00" from global condition, moving it to Branch 5 turn-on only; (2) pool_pump_last_on never updated: mode:single caused Branch 1 to be silently dropped when Branch 5 turned pump on, leaving pool_pump_continuous_run_minutes stale at ~18h; fixed by changing to mode:queued. Session F (2026-04-22): power_statistics.yaml created — inverter_production_7d_mean/30d_mean, house_load_24h_mean (statistics platform), solar_vs_forecast_ratio_7d + solar_weather_correlation (template). All 5 excluded from recorder. pv3/pv4 broken voltage entries removed from 3 dashboards (operations/overview/testing). Broken suppress-counter entity cards removed from dashboard_operations. prepaid_balance_confidence | min(30) bug fixed (must wrap in list). Dashboard card last_changed guards added for pool_pump_control_status, pool_target_run_hours_today, pool_pump_continuous_run_minutes. pyscript sync_power_groups.py: hass.* proxy is fully restricted — hass.data, hass.services, homeassistant.helpers imports, and task.executor(pyscript_fn) all blocked. Final working solution uses state.names(domain="sensor") + service.call() builtins only; label-based flexible/critical groups left empty (entity registry inaccessible from pyscript). Session D (2026-04-22): pool_pump_solar_control added to power_automations.yaml; 4 old pool pump blocks removed from automations.yaml; pool helpers added to power_helpers.yaml; pool runtime sensors (history_stats + 2 templates) added to power_state.yaml; POWER_CONTRACT pool section added. Session A (2026-04-21): grid_to_house_power fixed; energy flow sensors corrected; battery_energy_available + average_night_consumption statistics sensor added; solar_forecast.yaml syntax error + 3× sensor.inverter_power fixed; grid_risk.yaml + alerts_system_health.yaml sensor.inverter_power fixed; power_core.yaml inverter_today_energy_import deprecation documented; pyscript event.fire fixed. Session B (2026-04-21): power_automations.yaml created; grid_status_monitoring + inverter_pwer_monitoring commented out (superseded by alerts_power.yaml); inverter_alert_battery_soc_critical migrated with ssa_*→ss_* fixes; alerts_power.yaml 3× inverter_power fixed; load_shedding migrated to new packages/load_shedding/ dir (⚠️ restart required); B4 sensor rename; B5 group ref fixed; B6 prog1_capacity fixed. Session C (2026-04-22): sync_power_groups.py startup trigger added + import rewritten (hass.data.get); battery_runtime.yaml 6× int default fixed; battery_energy_available device_class removed; number.inverter_battery_capacity storage template int default fixed. Pool pump automations researched (C3 — 4 blocks, lines 2222–3174, research only). |
| 💳 Prepaid | Clean | Drift history tracking added 2026-04-23. Confidence layer done. |
| 🚰 Water | Updated 2026-06-14 | **2026-06-14:** Two blocking bugs fixed + force override added. (1) BUG Issue 14 FIXED — `water_block_refill_when_not_allowed` had no safety-state exemption; safety-state refills looped endlessly (Branch 1 start → block kill → repeat). Fixed: added NOT(safety) + NOT(critical+grid) exemptions to block automation, matching existing mid_run_shutdown exemptions. (2) BUG Issue 15 FIXED — Branch 2 (critical+grid) incorrectly required `water_refill_allowed = on` (SOC ≥ 50%), violating contract (critical+grid = any SOC). At 2% tank fill with SOC=45% and grid on, nothing fired. Fixed: Branch 2 now checks `group.inverter_grid = on` + `load_control_borehole_enabled = on` directly; block automation exempts critical+grid starts. (3) `input_boolean.water_refill_force_override` added — bypasses solar window, SOC, and last-sun-slot gates; safety hard-stops (40% battery floor, 1.95m max depth) still apply; Branch 4.5 fires when override ON. Saturday demand selector changed to Pool for Friday-night prep fill (1.6m target). **2026-05-25 (session H — three commits):** (1) binary_sensor.water_refill_allowed updated — last-sun-slot gate added (borehole blocked if SOC below overnight target during 14:00+ window, unless tank ≤ 20% critical override). (2) water_borehole_mid_run_shutdown automation added — stops pump mid-cycle if water_refill_allowed goes OFF or solar window closes; safety state exempt; does NOT set safety abort flag. (3) water_safety_battery_hard_stop automation added in water_safety.yaml — stops pump + sets safety abort flag when SOC < water_battery_soc_hard_stop (40% temp, lower to 20% after new batteries). Issue 6 FULLY RESOLVED. (4) Demand-based refill targets: 21 per-day demand booleans removed (were unused); replaced by 7 input_select per day (Mon–Sun, options: Normal/Wash/Clean/Irrigation/Pool); sensor.water_target_depth_tomorrow and sensor.water_demand_today added; Case 4 refill trigger now fires when depth < tomorrow's target (not just state=low); water_stop_at_daily_target automation added — pump stops at demand target instead of always 1.95m. (5) Thresholds corrected: critical 0.50→0.25m, minimum_safety 0.60→0.35m, low_trigger 1.00→0.80m, target_depth_normal 1.55→1.00m, target_depth_partial (Wash/Clean) 1.65→1.20m, target_depth_full (Pool Day) 1.85→1.60m; water_target_depth_irrigation added (1.25m). Season preset scripts: water_demand_set_summer_profile / water_demand_set_winter_profile. |
| 🔔 Alerts | Updated 2026-05-27 | alert.camera_health added 2026-04-29 (alerts/ = 14 files). **2026-05-27:** alerts_batteries.yaml added (15th file) — full battery alert pipeline for Honor 10 Dash + Honor X7 Dash. Pipeline: per-device low + overcharge binary sensors → dash_battery_alert_context → alert.dash_battery_alert → aggregator. Screen brightness management added in packages/admin/tablets.yaml (night dim, away dim, morning/arrival restore). ⚠️ Requires HA restart (alert: entity). |
| 🔔 Notifications | Scripts correct | All C-series bypasses resolved. BUG-N02 counter entity correct. |
| 🧭 Presence | Alert pipeline live | Unknown AP alert + occupancy anomaly implemented. Trust chain intact. |
| 💡 Lighting | Stable | All L01–L10 fixed and verified 2026-04-29. BUG-L09 closed: lighting_entertainment.yaml + lighting_energy_saving.yaml both populated. M1/M2/M3 implemented. SOC-based energy saving trigger remains future work (power session). |
| 🌐 Network | Updated 2026-05-28 | **2026-05-28:** sensor.ups_accessories_power (sum of USB1/2/3 + TypeC + DC out) and sensor.ups_visibility_score (accessories / total × 100) added to network_ups.yaml — mirrors load_visibility_score pattern from power domain. **2026-05-27:** network_ups.yaml added — EcoFlow River Pro UPS monitoring (packages/network/). Sensors: ups_on_battery, ups_runtime_seconds/friendly/eta, ups_status_card, ups_runtime_severity, ups_load_percent/status, ups_load_markdown. Helpers: 6 input_numbers + 1 input_boolean. Automations: AC Always On enforcement, on-battery notify, grid restore notify, battery warning/critical, load warning. All alerts via script.notify_power_event. | BUG-NET01 fixed 2026-04-28: unifi_cpu_5m_max availability → has_value(unifi_gateway_cpu_utilization). BUG-NET02 fixed 2026-04-28: unifi_memory_5m_max availability was self-referencing → has_value(unifi_gateway_memory_utilization). BUG-NET03 fixed 2026-04-28: packet loss removed from wan_health_score (ping_sum_5min is latency sum not pass count); score now uses latency + jitter only. BUG-NET04 fixed 2026-04-21. All verified live in network_helpers.yaml. |
| 🏗️ Context | All fixed | BUG-CTX01 fixed 2026-04-30: context_presence.yaml → presence/presence_trust.yaml. BUG-CTX02 fixed 2026-04-28: context_schedules.yaml deleted, bedtime_mode in lighting_helpers. BUG-CTX03 fixed 2026-04-30: home_context now derives from security_nobody_home + night_confirmed, no longer imports sensor.security_mode from security/. |
| 🔧 Infra | All fixed 2026-04-28 | BUG-CORE01 fixed (ha_events_per_second removed). BUG-INF01 fixed (printer_cartridge_low dangling }}). BUG-BKP01 fixed (github.yaml routed through notify_system_event). BUG-WEA01 confirmed already fixed. |

---

## 🚀 Verified Priority Work Queue

### Group A — Trust Model Chain (Fixes security + lighting + door alerts)
```
✅ A1. Restore input_datetime.low_trust_start/end — SUPERSEDED / NOT NEEDED.
✅ A2. boundary_permissive_window — CORRECTION: 2026-04-15 session only confirmed
       maid_on_site+weekday==0 logic (partial). Full rebuild to use
       binary_sensor.low_trust_present OR guest_mode completed 2026-05-17 (S1.3).
✅ A3. Replace input_boolean.low_trust_present → binary_sensor.low_trust_present
✅ A4. Same fix in lighting_departure.yaml
```
**Group A genuinely complete as of 2026-05-17. PROJECT_STATE previously recorded A2 as done but the 2026-04-15 fix was incomplete — boundary_permissive_window only fired on Monday maid days, not for gardener/contractor/guest_mode. S1.3 applied the full fix.**

### Group B — Alert Pipeline Gaps
```
✅ B1. Implement alerts_presence.yaml — done 2026-04-16
      binary_sensor.unknown_ap_alert_active (5 min delay)
      binary_sensor.occupancy_anomaly_alert_active (15 min delay, info only)
      sensor.presence_alert_context (warning/critical by AP duration)
      alert.presence_alert — STD_Alerts, repeat 60/120 min
B2. Implement alerts_security.yaml — done 2026-04-14 (already complete)
✅ B3. Add network_alert + media_alert to aggregator trigger list — done 2026-04-14
```

### Group C — Notification Bypasses (~36 min)
```
✅ C1. Add continue_on_error: true to Telegram in notify_water_events.yaml (BUG-N01, done 2026-04-14)
✅ C2. Fix counter: water_borehole_faults_week → water_borehole_faults_this_week (done 2026-04-15, water_notifications.yaml:158)
✅ C3. Add input_boolean.notifications_enabled to notifications_helpers.yaml (BUG-N03, done 2026-04-15)
✅ C4. Migrate alerts_temperature.yaml to script.notify_system_event (BUG-N04, done 2026-04-15)
✅ C5. Migrate presence_notifications.yaml to script.notify_presence_event (BUG-N05, done 2026-04-15)
✅ C6. Resolve dual notification in alerts_device_power.yaml (BUG-A04/N06, already clean — verified 2026-04-15)
✅ C7. Add | default('') guard to escaped_title/escaped_message in all 6 scripts (BUG-N08, done 2026-04-13)
✅ C8. Complete MarkdownV2 escape chain — add . and ! to all 6 scripts (BUG-N09, done 2026-04-13)
```

### Group D — Security Quick Fixes ✅ Done 2026-04-15
```
✅ D1. Fix typo: security_poor_visibility → security_visibility_poor
✅ D2. Fix typo: security_low_light_weather → security_weather_low_light
✅ D3. Fix cam06 last event reads cam09 (copy-paste, 1 line)
✅ D4. Fix cam10 image key: cam10_pool_bar_motion_images → cam10_pool_bar_images
✅ D5. Add unique_id to sensor.security_correlation
✅ D6. Remove presence_test_arrival test automation from production
```

### Group E — Water Trigger Integrity ✅ Done 2026-04-15
```
✅ E1. water_borehole_no_rise_protection: added from: "off" to pump ON trigger
✅ E1. water_reconcile_cycle_state: added from: "on" to pump OFF trigger
✅ E2. water_refill_visibility_guard: added for: "00:00:10" stability window
✅ E3. water_refill_allowed verified: checked on critical+low paths; safety/emergency paths intentionally bypass
```

### Group I — Infra / Network / Context Bugs (Audited 2026-04-16 — see INFRA_CONTRACT.md, NETWORK_CONTRACT.md, CONTEXT_CONTRACT.md)
```
✅ BUG-INF01 [HIGH]:   Fix binary_sensor.printer_cartridge_low — extra }} on trailing line
                        Fixed 2026-04-28: Removed dangling }} from state: > block

✅ BUG-NET03 [HIGH]:   WAN packet loss formula was mathematically wrong (ping_sum_5min is latency sum, not pass count)
                        Fixed 2026-04-28: Removed packet loss from health score — score now uses latency + jitter only

✅ BUG-NET01 [MED]:    sensor.unifi_cpu_5m_max availability references non-existent sensor.unifi_cpu
                        Fixed 2026-04-28: Changed to has_value('sensor.unifi_gateway_cpu_utilization')

✅ BUG-NET02 [MED]:    sensor.unifi_memory_5m_max availability was self-referencing (circular)
                        Fixed 2026-04-28: Changed to has_value('sensor.unifi_gateway_memory_utilization')

✅ BUG-BKP01 [MED]:    backup/github.yaml direct Telegram call bypassed quiet hours
                        Fixed 2026-04-28: Failure routes through script.notify_system_event (warning severity — bypasses
                        quiet hours so 5AM failures still alert). Success notification removed (5AM noise).

✅ BUG-WEA01 [LOW]:    weather_api_stale_alert already had full action block — checkbox not updated

✅ BUG-CORE01 [LOW]:   sensor.ha_events_per_second returned total sensor count, not a rate
                        Fixed 2026-04-28: Removed sensor from ha_monitoring.yaml (real delta-based
                        rate needs sensor.recorder_events which isn't available)

✅ IMP-IDS01 [MED]:    IDS Hyyp alarm stub created 2026-04-29: packages/security/security_alarm.yaml
                        Search of automations.yaml found ZERO IDS/hyyp references — integration not yet wired.
                        Stub documents provisional entity interface + migration instructions for when IDS goes live.

✅ BUG-CTX02 [LOW]:    context_schedules.yaml was stub
                        Fixed 2026-04-28: bedtime_mode moved to lighting_helpers.yaml (7 lighting consumers);
                        bedtime_time (unused) deleted; context_schedules.yaml deleted
```

### Group T — Telegram Enhancement (Decided 2026-04-20 — replaces defunct notify.whatsapp)
```
✅ T1. [HIGH] Security events → camera snapshot on Telegram
        Fixed 2026-04-28: telegram_bot.send_photo added after send_message in
        notify_security_events.yaml, guarded by img is not none. Uses existing img URL
        (https://ha.dunners.tech/local/security_latest.jpg). continue_on_error: true.

✅ T2. [MEDIUM] Inline keyboard acknowledgment for critical alerts
        Fixed 2026-04-28: inline_keyboard added to Telegram critical messages in
        notify_security_events.yaml (/ack_security_alert → alert.security_alert off) and
        notify_power_event.yaml (/ack_power_alert → alert.power_alert off).
        Callback automations added to each file.

✅ T3. [LOW] Suppress Telegram sound for information-level notifications
        Fixed 2026-04-28: disable_notification: "{{ sev == 'information' }}" added to
        Telegram data block in all 6 notify_*_event.yaml scripts.

✅ T4. [LOW] HA mobile app alarm sound for critical alerts
        Fixed 2026-04-28: push.interruption-level: critical (iOS) + channel: alarm +
        ttl: 0 + priority: high (Android) added to all STD_Critical notify calls
        across all 6 notify_*_event.yaml scripts.
```

### Group L — Lighting Bugs (Audited 2026-04-16 — see LIGHTING_CONTRACT.md)
```
✅ BUG-L01 [HIGH]: entrance_down_lights:off in scene_night_away — already present, checkbox not updated
✅ BUG-L02 [HIGH]: main_entrance_light behaviour corrected 2026-04-28:
                   Original bug said "add :off to scene_night_away" but correct behaviour is the opposite —
                   main_entrance_light stays ON overnight as deterrence (same as boundary lights),
                   only turns off in morning routine. Fixed: removed from scene_night_away;
                   explicit switch.turn_off added to morning_wake_lights_on action.
✅ BUG-L01+L02: scene_morning_routine_off already applied after scene_night_away in departure — checkbox not updated
✅ BUG-L03 [HIGH]: arrival anyone_home → anyone_connected_home — already fixed, checkbox not updated
✅ BUG-L04 [MEDIUM]: time + cam14 triggers in morning routine — already present, checkbox not updated
✅ BUG-L05 [MEDIUM]: entrance_down_lights + dining + 30s cancel in kids bedtime — already present, checkbox not updated
✅ BUG-L06 [LOW]: from:"off" on departure trigger — already present, checkbox not updated
✅ BUG-L07 [LOW]: kids_bedtime_enabled had no consumers — removed from lighting_helpers.yaml 2026-04-28
✅ BUG-L08 [LOW]: patio_second_wake_time had no consumers — removed from lighting_helpers.yaml 2026-04-28
✅ BUG-L09 [LOW]: M1 + M3 implemented 2026-04-29. M2 helpers added. SOC trigger deferred to power session.
✅ BUG-L10 [MED]: kids_bedtime_week + kids_bedtime_weekend — added continue_on_error: true to both
      script.notify_lighting_event calls. Notification failure can no longer stop the scene from firing.
      GAP ANALYSIS (2026-04-29):
      --- Entertainment mode ---
      • input_button.entertainment_mode_on exists (defined); input_boolean.entertainment_mode does NOT exist
      • lighting_entertainment.yaml is EMPTY — nothing responds to the button
      • scene.scene_entertainment_mode EXISTS in lighting_scenes.yaml (pool light + patio + entrance_down + dining) — scene is ready
      • CRITICAL: kids_bedtime_week + kids_bedtime_weekend both unconditionally turn off switch.pool_light_switch
        with NO entertainment mode guard — bedtime clobbers the scene mid-party
      PLAN: (1) add input_boolean.entertainment_mode to lighting_helpers.yaml
            (2) add button→boolean automation to lighting_entertainment.yaml (turn on button sets boolean ON; morning routine clears it)
            (3) add condition: entertainment_mode is OFF to both kids bedtime automations before pool_light_switch off
            (4) apply scene.scene_entertainment_mode when boolean turns ON; restore prior scene (or turn off) when it turns OFF
      --- Energy saving mode ---
      • input_button.energy_saving_mode_on + energy_saving_mode_off exist; input_boolean.energy_saving_mode does NOT exist
      • lighting_energy_saving.yaml is EMPTY
      • load_control.yaml already has load_control_geyser_enabled / pool_enabled / borehole_enabled with turn-off automations
      • energy_orchestrator_state sensor likely has states that should gate this (see POWER_CONTRACT.md)
      PLAN: (a) input_boolean.energy_saving_mode belongs in power_helpers.yaml (not lighting)
            (b) Power system sets it ON when battery SOC drops below a configurable threshold (power_automations.yaml)
              → also triggered by energy orchestrator hitting a critical state
            (c) Manual buttons (on/off) act as overrides (lighting_energy_saving.yaml handles them)
            (d) Lighting side checks binary: energy_saving_mode is ON → suppress non-essential lights (bar, patio, feature lights)
            (e) Morning routine or SOC recovery clears the boolean
      OWNER: power_automations.yaml owns the SOC-threshold trigger; lighting_energy_saving.yaml owns the lighting response
```

### Group M — Mode Features (Entertainment + Energy Saving) [PLANNED 2026-04-29]
```
[ ] M1. Entertainment mode — lighting side (lighting package)
        NOTE: input_boolean.entertaining_mode ALREADY EXISTS in context_presence.yaml.
              Button is input_button.entertainment_mode_on (name mismatch: "entertainment" vs "entertaining").
              DO NOT create a new input_boolean — wire the button to the EXISTING entertaining_mode boolean.
        a. Populate lighting_entertainment.yaml:
           - button→boolean: input_button.entertainment_mode_on → input_boolean.entertaining_mode ON
           - scene apply: entertaining_mode turns ON → scene.scene_entertainment_mode
           - scene restore: entertaining_mode turns OFF → revert (or apply scene_evening_routine)
           - morning clear: morning routine → input_boolean.entertaining_mode OFF
        b. Guard both kids_bedtime_week + kids_bedtime_weekend:
           - Add condition: input_boolean.entertaining_mode is OFF before pool_light_switch step
           NOTE: scene.scene_entertainment_mode already exists — no scene work needed

✅ M1. Entertainment mode — lighting side (lighting package) [2026-04-29]
        CORRECTION: Uses input_boolean.entertaining_mode (context_presence.yaml) — same concept,
        same boolean. input_boolean.entertainment_mode duplicate removed from lighting_helpers.yaml.
        a. lighting_entertainment.yaml populated (all wired to input_boolean.entertaining_mode):
           - Button → boolean on (entertainment_mode_button_on)
           - Boolean ON (from: "off") → scene.scene_entertainment_mode (entertainment_mode_scene_on)
           - Boolean OFF (from: "on") → scene.scene_evening_routine_on if civil_night is on (entertainment_mode_scene_off)
           - 06:00 daily clear → entertaining_mode OFF if currently on (entertainment_mode_daily_clear)
        b. kids_bedtime_week + kids_bedtime_weekend: entertaining_mode guard added — if ON, skip
           pool_light_switch in scene; turn off other 4 lights individually instead.

✅ M2. Energy saving mode — power side helpers (power package) [2026-04-29]
        a. input_boolean.energy_saving_mode added to power_helpers.yaml (icon: mdi:lightning-bolt-outline, initial: false)
        b. input_number.energy_saving_soc_threshold (default 25%) added to power_helpers.yaml
        c. input_number.energy_saving_soc_recovery (default 40%) added to power_helpers.yaml
        OPEN: power_automations.yaml SOC trigger + orchestrator trigger not yet wired (future M2 work)
        Manual override buttons (energy_saving_mode_on/off) now wired via lighting_energy_saving.yaml.
        Morning wake routine clears energy_saving_mode (lighting_morning.yaml morning_wake_lights_on).

✅ M3. Energy saving mode — lighting side (lighting package) [2026-04-29]
        a. lighting_energy_saving.yaml populated:
           - Button on/off → boolean wiring automations
           - energy_saving_mode ON (from: "off") → turn off TIER 2 lights:
               switch.pool_light_switch, switch.pool_patio_down_lights, switch.back_house_security_light
           - logbook + notify (continue_on_error: true)
        b. Restoration NOT done here — presence/routine triggers handle it naturally.
        TIER 2 definition: pool light, pool patio, back house security (entertainment/comfort lights).
```

### Group F — Restart-Protection Guards ✅ Done 2026-04-13
Spurious fires on HA restart / template reload — missing `from:` / `not_from:` guards.
```
✅ F1. presence_notifications.yaml — Ryan/Vicky/Luke/Tayla unknown AP triggers: added from: "off"
✅ F2. admin_notifications.yaml — Unknown UniFi AP Detected: added from: "off"
✅ F3. water_notifications.yaml — water_tank_full_notification: added not_from: [unknown, unavailable]
✅ F4. water_notifications.yaml — water_refill_never_reached_full: added from: "off"
✅ F5. lighting_bar_presence.yaml — lighting_bar_presence (bar_occupied): added from: "off"
✅ F6. lighting_arrival_night.yaml — lighting_arrival_night (arrival_detected): added from: "off"
✅ F7. lighting_morning.yaml — morning_wake_wfh_cleanup (bedrooms_occupied): added from: "off"
✅ F8. energy_state.yaml — grid_charging_battery_while_solar_available: added not_from: [unknown, unavailable]
```
Rule: binary sensors / input_booleans → `from: "off"`. Template/state sensors → `not_from: [unknown, unavailable]`.

### Group U — iCloud Re-integration [DEFERRED — blocked on upstream fix]
```
BLOCKER: icloud integration broken on HA 2025.11.0 (Issue #155933 — dict has no attribute user_info).
         Do not start U1–U3 until integration is confirmed working on current HA version.
         Monitor: https://github.com/home-assistant/core/issues/155933

U1. [DEFERRED] Re-add iCloud integration via UI (Settings → Integrations → Apple iCloud)
    Use primary Apple ID account (non-primary family accounts unsupported upstream).
    Enable "with family" to expose all family device trackers.

U2. [DEFERRED] presence/presence_icloud.yaml — new file
    - Monitor device_tracker.* iCloud entities going unavailable (auth failure detection)
    - Auto-delete /config/.storage/icloud via shell_command on failure detection
    - Trigger HA restart via homeassistant.restart service
    - Send push notification (script.notify_presence_event, warning severity) with
      deep link to /config/integrations so re-auth is one tap away
    - Startup guard: suppress during system_startup window to avoid false triggers
    Note: 2FA code entry itself cannot be automated — human must approve on Apple device.
    This reduces monthly re-auth from manual diagnostic + terminal + restart to
    a single notification → tap → enter code flow.

U3. [DEFERRED] Apple device battery pipeline — two options (decide at implementation time):
    Option A: Extend alerts/alerts_batteries.yaml — add Apple device battery/charging
              sensors alongside existing Honor tablet pipeline (same pattern)
    Option B: New alerts/alerts_apple_devices.yaml — separate file if scope grows
              (location staleness monitoring, find-my state, etc.)
    Entities expected from iCloud: sensor.PERSON_iphone_battery_level,
    binary_sensor.PERSON_iphone_battery_charging — one per family member device.
```

---

## 📚 Document Index

| Document | Purpose | Authority |
|---|---|---|
| `docs/PROJECT_STATE.md` | This file — master context | ✅ Updated 2026-04-16 |
| `docs/CODING_STANDARDS.md` | Code rules and conventions | ✅ Current |
| `docs/SESSION_STARTERS.md` | Copy-paste session prompts | ✅ Current |
| `docs/SYSTEM_CONTRACT.md` | Cross-domain dependency matrix + all bugs | ✅ Audit 2026-04-13 (partial — predates context/network/infra contracts) |
| `docs/domains/SECURITY_CONTRACT.md` | Security — full audit, entity ref, checklist | ✅ Authoritative |
| `docs/domains/POWER_CONTRACT.md` | Power — full audit, measurement model | ✅ Authoritative |
| `docs/domains/WATER_CONTRACT.md` | Water — lifecycle contract verification | ✅ Authoritative |
| `docs/domains/PRESENCE_CONTRACT.md` | Presence — trust model audit | ✅ Authoritative |
| `docs/domains/ALERTS_CONTRACT.md` | Alerts — pipeline audit | ✅ Authoritative |
| `docs/domains/NOTIFICATIONS_CONTRACT.md` | Notifications — bypass audit | ✅ Authoritative |
| `docs/domains/LIGHTING_CONTRACT.md` | Lighting — full audit, scenes, 9 open bugs | ✅ Authoritative (2026-04-16) |
| `docs/domains/NETWORK_CONTRACT.md` | Network/WAN — entity ref, all bugs fixed | ✅ Authoritative (2026-04-16, bugs verified 2026-04-30) |
| `docs/domains/CONTEXT_CONTRACT.md` | Context — night mode; CTX01/02/03 all resolved 2026-04-30 | ✅ Authoritative (updated 2026-04-30) |
| `docs/domains/INFRA_CONTRACT.md` | Infra — core/backup/office/weather/integrations/HACS | ✅ Authoritative (2026-04-16) |

> **The `*_CONTRACT.md` files are authoritative.** They were produced by reading actual config files.
> The older `*_CONTEXT.md` files are quick-reference summaries — use contracts for real work.

---

## ⚠️ Known Time Wasters (Avoid Repeating)

- Using `input_boolean.low_trust_present` in automations — always use `binary_sensor.`
- Storing structured data in `input_text` (255 char limit)
- Triggers without `from:` constraints (fires on unavailable→on)
- Calling `notify.*` directly instead of `script.notify_*_event`
- Editing `automations.yaml` — always use package files
- Building UI before backend is stable
- Missing `continue_on_error: true` on the **calling** `script.notify_*_event` action — a Telegram `ConnectTimeout` is an "Unexpected error" that propagates past `continue_on_error` inside the script and kills the caller. Always add `continue_on_error: true` to the calling action, not just inside the script.

---

## ⚠️ Known Integration Issues (Non-Config)

| Integration | Issue | Severity | Action |
|---|---|---|---|
| `hikvision_next` v1.1.1 | Sets invalid entity IDs with uppercase serial number (`DS-7116HGHI-F1...`). Deprecation warning — breaks in HA 2027.2 | Low | File upstream bug at maciej-or/hikvision_next |
| `bellows` (Zigbee) | `RESET_SOFTWARE: 11` on startup | None | Normal coordinator reset during init — ignore |
| `ids_hyyp` v1.9.0 | Zero automations in automations.yaml — integration not yet wired at HA level | Medium | Stub created (IMP-IDS01 ✅). Migrate entity interface when IDS is live. |
| `localtuya` v5.2.3 | False reconnect event can trigger spurious water tank abort | Medium | See WATER_CONTRACT.md — input_boolean.water_refill_aborted_due_to_safety can get stuck |
| Multiple weather integrations | OWM + OWM History + Met.no + Met Nowcast all installed — canonical source unclear | Low | Document which is used for what in INFRA_CONTRACT.md |
| `icloud` (built-in) | **Removed end of 2025.** Monthly 2FA session expiry requires manual `.storage/icloud` delete + HA restart + code entry. Non-primary Apple ID (family members) unsupported. Broke entirely on HA 2025.11.0 (`dict has no attribute user_info`). Value lost: GPS away-from-home location + Apple device battery/charging state for all family devices. | High | **DEFERRED** — monitor upstream fixes (Issue #155933 for 2025.11 breakage). Re-add only when auth is stable. Recovery automation + Apple device battery pipeline planned: see Group U. |

---

*2026-06-14 session (power Session E5 — Inverter programme automation): Three new automations added to power_automations.yaml. (1) inverter_energy_pattern_control (E5-2): Battery First / Load First crossover. Triggers: time_pattern /5min + orchestrator state change + homeassistant start + time 16:30. Four branches: Branch 1 (orchestrator in [loadshedding, loadshedding_critical, critical, conserve] → Battery First, severity warning/info); Branch 2 (morning, hour < 10, SOC < 65% threshold → Battery First, silent); Branch 3 (orchestrator normal/surplus, SOC ≥ 65%, 09:00–16:00 → Load First); Branch 4 (16:30 evening return → Battery First). All pattern changes call force_inverter_sync after 5s delay. (2) inverter_p4_grid_charge_control (E5-3): Dynamic P4 grid charge at 14:00–17:00 window. Evaluates at 14:00/14:30/15:00: if SOC gap > 5% AND kwh_shortfall > orchestrator_solar_gap_threshold (2 kWh) → enable P4 Grid on both inverters directly; else if P4 is Grid and on track → disable. 17:00 trigger always restores P4 to Disabled. Uses sensor.house_load_24h_mean (24h mean W) for load estimate. (3) solar_forecast.yaml gated by input_boolean.use_legacy_solar_scenes (default: false) — scenes no longer fire; E5 owns inverter control. Rollback: set use_legacy_solar_scenes to on. inverter_sync_check expanded: now triggers on all 6 program_N_charging changes + energy_pattern; compares all 7 entities; details mismatch. force_inverter_sync expanded: copies energy_pattern + all 6 charging selects + all 6 SOC numbers (INV1→INV2). New helpers: inverter_programme_auto_enabled (bool, true), use_legacy_solar_scenes (bool, false), orchestrator_p4_charge_trigger_soc (70%), orchestrator_solar_gap_threshold (2 kWh), battery_capacity_kwh (30 kWh — update to 45 after Tuesday swap). Pre-flight revealed: battery = 30 kWh (not 17.3); P4 window = 14:00–17:00 (confirmed from Developer Tools); all inverter_2_program entities confirmed via scenes.yaml. ha core check passed. Functional check: orchestrator=conserve → energy_pattern changed Load First → Battery First immediately on reload. P4=Disabled (17:00 already closed). POWER_CONTRACT Inverter Sync section updated (removed PARTIAL flag), Sprint 5 marked IMPLEMENTED, E5-2/E5-3 automation logic documented.*

*2026-06-14 session (power Session E4 — Geyser orchestrator retrofit + midday solar gate): geyser_automations.yaml fully rewritten (E2 baseline → E4). geyser_turn_on: 5 branches. Morning non-negotiable — only loadshedding_critical blocks (hot water essential); morning_override bypasses load_control_geyser_enabled; 4 fixed triggers (03:30/04:00/04:30/05:00) match weekday+weekend×winter start time helpers; branch condition encodes season+weekday logic. Midday upgraded from unconditional (E2) to solar-gated: orchestrator in [surplus,normal] + solar_available_surplus > geyser_midday_surplus_threshold (300W) + geyser_at_temperature off + before 15:00. Evening standard 20:00 non-winter / winter 19:30 (30 min earlier); sports night 21:00. geyser_turn_off: 7 branches + default, mode:queued. Morning hard-off added (07:30/08:00/08:30/09:00 weekday/weekend×winter). Midday hard-off at 15:00. Evening standard 21:30/winter 22:00/sports 23:00/winter sports 23:30. Orchestrator emergency off: loadshedding_critical → turn off outside morning window only (morning sacred). AM battery protection preserved (grid off + SOC < prog2 + shedding — fires even in morning window). New sensors added to power_state.yaml: sensor.geyser_control_status (13-state priority display) + binary_sensor.geyser_at_temperature (power proxy: <50W sustained 5min = at temp). New helpers in power_helpers.yaml: geyser_morning_start_weekday (4h), geyser_morning_start_weekend (5h), geyser_morning_end_weekday (7.5h), geyser_morning_end_weekend (8.5h), geyser_winter_start_offset (30min), geyser_midday_surplus_threshold (300W). Entity name correction: canonical solar surplus is sensor.solar_available_surplus (Watts) NOT sensor.solar_surplus_available (boolean string). ha core check passed. Functional check: geyser_control_status=Outside active windows (correct — between 15:00 and evening start at ~19:00), geyser_at_temperature=off (drawing full power), switch=on, helpers loaded. POWER_CONTRACT geyser section rewritten (E4).*

*2026-06-14 session (power Session E3 — Pool pump orchestrator retrofit): pool_pump_solar_control in power_automations.yaml retrofitted to use sensor.energy_orchestrator_state as the primary gate — this is the reference pattern all appliance automations will follow. Turn-on (Branch 5): replaced raw battery_state_health + load_shedding_stage + upcoming-shed SOC checks with `orchestrator in ['surplus', 'normal']` as the single gate; local conditions preserved: grid-offline SOC floor (grid_offline_soc_min_pool 60%), last-sun-slot (last_sun_soc_target 80%/90%), winter morning hold (pool_winter_start_hour 10h). Turn-off: Branch 2 replaced raw load_shedding_stage >= 2 / upcoming-shed logic with `orchestrator in [loadshedding, loadshedding_critical, critical]`; loadshedding_critical overrides minimum run time protection (emergency turn-off); other states respect min run time, notify at warning. Branch 3 replaced raw battery_state_health + headroom < 800W with `orchestrator == conserve OR headroom < 800W`. sensor.pool_pump_control_status (power_state.yaml) fully rewritten to read orchestrator state — 11-state priority display: Disabled / load-shedding blocked / critical battery / target reached / running (surplus/min-time) / conserve / last sun slot / winter morning hold / outside hours / solar insufficient / waiting. Entity name fix: sensor.solar_surplus_available (boolean True/False string in solar_core.yaml) replaced with sensor.solar_available_surplus (canonical Watts metric in power_state.yaml) in lovelace.dashboard_operations line 5212. ha core check passed. Functional check: orchestrator=conserve, pool status=Off—target reached (100 min / 90 min winter target), solar_available_surplus=-4412W (evening, correct), switch=off.*

*2026-06-14 session (power Session E2 — Geyser automation migration): automations.yaml IDs 1742224800650 (turn on) + 1744130174080 (turn off) migrated to packages/power/geyser_automations.yaml. Migration fixes: device_id → switch.geyser_heat_pump_switch (confirmed); sensor.inverter_power → sensor.inverter_load_power; notify.STD_Information → script.notify_power_event; load shedding check re-enabled (was blocked on string comparison against unavailable sensor — now uses state_attr | int(0) < 4). New automations: geyser_turn_on (4 branches: morning standard 04:30 / morning winter 04:00 / midday 12:05–14:05 unconditional / evening standard 20:05 / evening sports 21:05), geyser_turn_off (4 branches: AM battery protection 05:30–08:30 / winter evening 22:00 / standard evening 21:05 / sports night 23:05), geyser_sports_night_scheduler (Tue+Thu 17:00 → on; 00:01 daily → off), geyser_manual_run (timed run via input_select.geyser_manual_run_duration 30/60min). script.geyser_manual_run dashboard entry point. Seasonal trigger design: dual triggers with IDs (04:00+04:30; 21:05+22:00); both fire daily, branch condition filters by trigger.id + sensor.season — single automation, no code duplication. New helpers in power_helpers.yaml: geyser_sports_night (bool), geyser_morning_override (bool), geyser_manual_run_active (bool), geyser_manual_run_duration (select 30/60). ha core check passed. Functional check: switch.geyser_heat_pump_switch=on, geyser_sports_night=off, geyser_manual_run_active=off, geyser_manual_run_duration=30, geyser_enabled=on. All 4 automations confirmed live in Settings → Automations. POWER_CONTRACT Section 2 (file count 25→26) + Section 8 (geyser helpers + automation logic) updated. AUTOMATIONS_AUDIT.md geyser section marked MIGRATED.*

*2026-06-14 session (security — dogs_inside departure notification tap): Two files changed. (1) security_automations.yaml: automation.dogs_inside_from_notification added — handles mobile_app_notification_action DOGS_INSIDE_ON event; turns on input_boolean.dogs_inside + logs to logbook. No auto-off — must be cleared manually on return (arming logic prevents false alerts even if left on). (2) notify_security_events.yaml: departure Stage 1 push notification updated to include inline action button "DOGS INSIDE 🐕" (action DOGS_INSIDE_ON). Suppressed when dogs_inside already ON, guest_mode ON, or staff_on_site ON. Fires at Stage 1 (gate opened / ipcam01 departure trigger) so leaving family member can tap before driving off. SECURITY_CONTRACT sprint 9h+ section added. No new entities — dogs_inside and dogs_out already locked in PROJECT_STATE.md (added S9h 2026-05-20).*

*2026-06-14 session (power Session E1 — Energy Orchestrator core): sensor.energy_orchestrator_state rebuilt from basic 5-state sensor (run_heavy_loads/run_medium_loads/reduce_load/conserve_energy/normal) into 6-state priority ladder: loadshedding_critical / loadshedding / critical / conserve / surplus / normal. Priority logic: loadshedding_critical = load shedding active AND SOC < emergency threshold; loadshedding = active OR upcoming within pre_shed_hours_warning window AND SOC < pre_shed target; critical = battery_state_health == 'critical' OR SOC < critical threshold; conserve = health == 'low' OR SOC < conserve threshold OR solar degraded AND SOC < degraded floor; surplus = solar export > threshold AND battery strong/healthy; normal = default. Gated by input_boolean.orchestrator_enabled (when OFF returns normal always). sensor.orchestrator_decision_reason added — human-readable explanation for dashboard/logbook. 12 new input helpers in power_helpers.yaml: orchestrator_enabled (bool), inverter_sync_status (bool), plus 9 SOC/threshold input_numbers (emergency 15% / critical 25% / conserve 50% / pre_shed 80% / conserve_degraded_soc 60% / surplus_export 1000W / target_soc_by_sunset 90% / load_first 65% / pre_shed_hours 3h) + pool_winter_start_hour (10h). binary_sensor.water_refill_allowed updated: added orchestrator gate — blocks normal refills when state in [critical, loadshedding, loadshedding_critical]; conserve does NOT block. All three emergency branches (safety, critical+grid, critical+limited) in water_tank_refill_control.yaml confirmed separate and unaffected. Inverter sync check: automation.inverter_sync_check (90s delay after energy_pattern change, compares and sets inverter_sync_status, notifies on mismatch) + script.force_inverter_sync (copies INV1 energy_pattern to INV2) added. Currently partial — only select.inverter_1/2_energy_pattern confirmed; work_mode/program_N_charging/inv2_soc entities need verification in Developer Tools. Spec correction: battery_state_health states are strong/healthy/low/critical — no 'warning' state (spec said ['low','warning'] for conserve; corrected to ['low']). Degraded SOC floor parameterised (was hardcoded 60% — moved to input_number.orchestrator_conserve_degraded_soc_threshold). ha core check passed. Functional check: orchestrator = normal (SOC 64%, battery healthy, no surplus, no load shedding — correct). water_refill_allowed orchestrator_state attribute confirmed. POWER_CONTRACT Section 7 (orchestrator states) + Section 8 (helpers + inverter sync) updated. WATER_CONTRACT Section 5 cross-domain table + refill_allowed definition updated.*

*2026-05-28 session (network: Synology NAS graceful shutdown + WoL restore + HA Pi shutdown): network_nas.yaml created (3rd network/ file). Three-stage shutdown pipeline: Stage 1 warn at <15 min remaining (critical notification, grace countdown); Stage 2 grace period (5 min, cancellable by toggling ups_nas_auto_shutdown_enabled OFF or grid restore); Stage 3 NAS shutdown (button.guardians_shutdown) → 4 min wait → hassio.host_shutdown (Pi). binary_sensor.ups_nas_shutdown_imminent = ON when on battery AND runtime < warn_threshold. WoL restore: HA startup trigger → if ups_nas_was_shutdown flag ON AND NAS offline → 3 min delay → button.wol_synology_nas_00_11_32_ad_af_a5 → 5 min wait → confirm/warn → clear flag. WoL broadcast_address bug fixed in .storage/core.config_entries: was 192.168.1.6 (unicast to powered-off IP, never works) → 192.168.1.255 (subnet broadcast). NAS on 192.168.1.x, Pi on 10.10.1.x (cross-subnet WoL via routing). DSM WoL confirmed enabled on LAN 1 + LAN 2 physical NICs; auto-restart on power failure also enabled in DSM. WoL targets physical NIC MAC (00:11:32:AD:AF:A5), not bond interface (Bond 1 = LAN1+LAN2, WoL is per-NIC only). 3 automations + binary sensor + 5 input helpers. Restart required (new helpers). See NETWORK_CONTRACT.md Section 10.*

*2026-05-28 session (dashboard battery alerts + Mobile Devices view): alerts_batteries.yaml created (15th alerts/ file) — canonical battery alert pipeline for Honor 10 Dash (HEY3-W00, 10100 mAh, 39.1 Wh) and Honor X7 Dash (JMS-W09, 7020 mAh, 27.2 Wh). Full pipeline: per-device low binary sensors (delay_on 1 min, < 30% AND not charging) + per-device overcharge binary sensors (delay_on 2 h, ≥ 95% while charging) → binary_sensor.dash_battery_alert_active → sensor.dash_battery_alert_context (critical/warning/normal, devices[] with SOC + runtime) → alert.dash_battery_alert (30/60 min repeat, STD_Alerts) → global aggregator. Runtime estimate sensors: honor10_dash_battery_time_remaining_est / honorx7_dash_battery_time_remaining_est (minutes; -1 when charging; 0.5W doze fallback when power < 0.1W). Screen brightness management added to packages/admin/tablets.yaml (new admin/ package, first file): 4 automations — night_dim (night_mode ON → night brightness), morning_restore (night_mode OFF → day or away), away_dim (nobody_home ON, not night → away brightness), return_restore (nobody_home OFF, not night → day brightness). Three input_number helpers for day/night/away brightness levels (0–255 Android scale). ⚠️ Restart required (alert: entity). Mobile Devices dashboard view (lovelace.dashboard_operations, mobile-devices path): 3-column sections layout built — col 1: summary tiles + alert status card + brightness sliders + alert settings; col 2: Honor Pad 10 detail (battery/state/runtime, health/temp/power, cycles/low/overcharge, screen/interactive/doze, per-device brightness buttons); col 3: Honor Pad X7 (same structure). Heading badges fixed: type: entity (type: template not supported in heading card badges). 16 new locked entities — see Dashboard Battery Alert Entities section. Network: sensor.ups_accessories_power (sum USB1/2/3/TypeC/DC) and sensor.ups_visibility_score (accessories/total × 100) added to network_ups.yaml.*

*2026-05-27 session (security S15 — RUNG 5 three-way split + gate_activity rung): sensor.security_event_classification now has 12 output states (was 10). RUNG 5 split into three: RUNG 5a visitor (entrance_valid + allhome OR nobody home), RUNG 5b gate_activity (entrance_valid + some family home — ambiguous arrival/visitor, neutral notification with gate control), RUNG 5c perimeter_front (no entrance_valid — street/passing activity, always fires, dynamic severity: warning daytime+family home / critical night or away). entrance_valid = binary_sensor.ipcam01_street_driveway_up_entrance_valid (AcuSense regionentrance, Higher validity precision zone at gate). BUG-S47 fixed (two paths): (1) security_image_inside_main slot never written during RUNG 2.5 events — security_capture_best_snapshot skips low/none confidence; inside cameras excluded from confidence scoring; added zone-slot write to security_capture_each_camera_motion for cam14/cam15/cam05. (2) Visitor/perimeter_front notification image stale (timing race — router reads slot before security_capture_best_snapshot writes it); added 4s delay + local image re-read in visitor and perimeter_front branches (same pattern as critical_intrusion 3s delay). BUG-S44 fixed: cam14/cam15 NVR false motion from go2rtc reconnect replay — delay_on: 3s added to both debounce sensors (cameras_processing.yaml). BUG-S45 partial fix: pool alarm deactivation — ipcam04_deactivate_alarm REST command added + security_dogs_out_cancel_alarm automation. BUG-S46 fixed: RUNG 8 missing not arriving guard (cam15 false critical during family AP transition). ipcam03 dog AcuSense misclassification noted (camera fix required — Minimum Target Size in Region Exiting rule). No new helper entities. Files: security_logic.yaml, security_automations.yaml, cameras_processing.yaml, security_helpers.yaml, SECURITY_CONTRACT.md.*

*2026-05-23 session (security S13 — BUG-S37 through BUG-S43 implemented): All 7 bugs fixed. BUG-S37: ipcam03_driveway_exit_valid now includes linedetection (departure line crossing). BUG-S38: ipcam01_recent window 120s→180s. BUG-S39: RUNG 2.5 hard 5am floor added (now().hour < 5); 90s cooldown on critical_intrusion branch (input_datetime.last_critical_event stamped before delay); 3s delay + re-eval image in critical branch (snapshot race fixed). BUG-S40: visitor cooldown 1800s when staff_on_site ON (was flat 30s). BUG-S41: security_image_arrival_locked added (locked at Stage 1 T+5s before cam04 overwrites grounds_front); Stage 2 reads locked slot for both arrival confirmed and Stage 1 notification. BUG-S42: Stage 1 title softened to "Gate opened — vehicle entering"; Stage 2 Unknown → "Nobody newly detected — visitor, delivery, or same-person return." BUG-S43: security_lighting_required entity reference fixed (security_visibility_low → security_weather_low_light); boundary_security_on switch.turn_on calls both get continue_on_error: true. New entities: input_datetime.last_critical_event, input_text.security_image_arrival_locked. Files: cameras_processing.yaml, security_core.yaml, security_logic.yaml, security_helpers.yaml, security_automations.yaml, lighting_boundary.yaml.*
*2026-05-23 session (security S12 — diagnosis + gitupdate.sh fix): Diagnosis-only session. Investigated real failures from 2026-05-23 field events. 7 new bugs identified (BUG-S37 through BUG-S43) — no code shipped, pending user approval. Summary: (1) BUG-S39 CRITICAL — RUNG 2.5 false critical ×6 at 05:03am when family rose early: is_late_night (h<6) still true, all_family_in_bedrooms still true (AP delay), ext_recent true from passing car; zero cooldown on critical_intrusion → 5 queued router dispatches all fired; first snapshot blank (2s race), subsequent 5 stale same-image. (2) BUG-S37 HIGH — ipcam03_driveway_exit_valid watches only regionexiting; S11b set line crossing to departure direction but linedetection signal ignored by exit_valid → missed departures. (3) BUG-S40 — staff visitor spam: S10 removed `not staff` from RUNG 5 (correct for maid-at-gate), but Saturday gardener cleans all morning at front gate → 30s cooldown gives repeated visitor notifications. (4) BUG-S41 — Stage 2 shows carport NVR image: cam04 ranks above ipcam03 in trigger_camera priority; car drives in → cam04 overwrites grounds_front image slot; Stage 2 fires 3.5min later, reads stale carport dark frame. (5) BUG-S38 — ipcam01 120s window too narrow: two gate opens showed first arrival missed because ipcam01 hadn't fired within 120s. (6) BUG-S42 — delivery person opening gate = false "Arrival — vehicle entering" (structural limitation of Stage 1 gate+ipcam01 logic). (7) BUG-S43 — security_lighting_required references non-existent binary_sensor.security_visibility_low (correct name: security_weather_low_light) → weather condition never triggers boundary lights. Also fixed: gitupdate.sh — sync_docs.sh failure was silent (no error check on call). Fixed to surface sync_docs.sh exit code with warning. sync_docs.sh confirmed working: ha-biesie-docs in sync with local docs (SSH deploy key at ~/.ssh/id_ed25519 is correctly scoped to ha-biesie-docs; main config repo uses /config/.ssh/id_rsa via core.sshcommand). Files changed: gitupdate.sh (sync_docs.sh error propagation).*
*2026-05-22 session (security_external_motion_recent moved to security_logic.yaml): Entity was defined in security_zones.yaml but did not create after reload (entity not found). Moved to security_logic.yaml alongside binary_sensor.security_intruder_active which uses the same delay_off pattern and is confirmed working. No functional change — same entity_id, same logic, same delay_off: 5min.*
*2026-05-22 session (security S11c): ipcam03_driveway_entrance_valid: regionentrance → fielddetection. regionentrance disabled in camera UI after user removed Region Entrance zone from ipcam03. fielddetection restores confirmed_human signal for threat_level sensor. entrance_valid no longer used as Stage 1 trigger. NVR cam05 sensitivity raised 20→40 (NVR web UI, not code). ipcam04 Target Validity confirmed already High. File: cameras_processing.yaml.*
*2026-05-22 session (security S11b): Stage 1 rewired for new ipcam03 camera config. User disabled ipcam03 Region Entrance, set line crossing to driveway→gate (departure) direction only. Stage 1 arrival trigger changed from ipcam03_driveway_entrance_valid → binary_sensor.main_gate_sensor (from:off, to:on). Condition: gate trigger proceeds only if ipcam01_recent (<120s) = vehicle from street. exit_valid trigger: suppressed if gate opened <120s ago AND ipcam01_recent (arriving car in exit zone). cam_dir simplified: gate=arrival, exit=departure. File: security_automations.yaml.*
*2026-05-22 session (security S11 — false intruder fixes, outdoor corroboration, visitor immediate): anyone_connected_home: delay_off 2min added — was instantly OFF on AP disconnect → departing car triggered CRITICAL intruder. RUNG 7 + 8d: `not departing` guard added. New binary_sensor.security_external_motion_recent (security_zones.yaml, delay_off 5min) — stays ON for 5min after any outdoor camera fires. RUNG 2.5 (stay-mode lounge at night): added ext_recent requirement — NVR cam14 headlights/shadows no longer escalate without outdoor corroboration. RUNG 8 (nobody home critical): added (ext_recent OR ip_cam OR high_conf) — NVR cam14/15 alone without outdoor context falls silently to RUNG 8d (family_movement). zone_label: grounds split into 'grounds front'/'grounds rear' based on trigger camera (was always 'grounds' → router always used carport image for rear cameras). Router zone_img map: 'grounds rear' → security_image_grounds_rear added. Router reason/zone: changed from live state_attr to trigger.to_state.attributes (snapshot at trigger time, not stale re-evaluation). Visitor branch: 45s grace removed → immediate notification; cooldown 60s→30s. ipcam02 left unchanged (admin access pending, BUG-S28 still open from HA perspective). Files changed: presence_core.yaml, security_zones.yaml, security_logic.yaml, security_automations.yaml. New locked entity: binary_sensor.security_external_motion_recent.*
*2026-05-21 session (security S10 — direction fix, staff muting, perimeter always active): BUG-S36 — departure misclassified as arrival (confirmed Telegram 06:55): departing car backs through gate-mouth entrance zone → entrance_valid fires → Stage 1 classified as arrival → Stage 2 "Unknown home". Root: entrance_valid always treated as arrival. Fix: cam_dir in Stage 1 now checks ipcam01 (street-up) recency — if no street approach within 120s, entrance_valid = departure (car was already inside property backing out). exit_valid always = departure. Mutual 120s suppression condition added: if entrance_valid fires within 120s of exit_valid (or vice versa), the second sensor is blocked — prevents an arriving car crossing the exit zone on its way to garage from generating a spurious second Stage 1. direction variable simplified: cam_dir if cam_dir != 'unknown' else ap_dir (was cam_dir if not both_active else ap_dir). RUNG 4 (service_person): removed `perim` — perimeter (street cameras ipcam01/02) was being swallowed by service_person rung during maid hours, muting visitor/perimeter alerts at gate. Fix: RUNG 4 now only catches `grounds or inside_any`. RUNG 5 (visitor): removed `and not staff` — visitor at gate must fire even when maid is on site. Router service_person branch: changed from 10-min-cooldown push notification to silent logbook always (no push ever for staff movement). Stage 2 departure: suppress "Unknown departed" notification when staff_on_site=on (maid leaving triggers departure via ipcam03, Stage 2 gets "Unknown" since maid not in person list). Zone display format in reason attribute: human-readable names (`Grounds` instead of `-G---`). Files changed: security_logic.yaml (RUNG 4/5 + zones format), security_automations.yaml (Stage 1 condition + cam_dir + service_person router + Stage 2 departure suppress). SECURITY_CONTRACT Sprint 10 added + BUG-S36 documented. No entity additions/renames. Reload required: template entities + automations.*
*2026-05-08 session: Security camera system restructure — cam06 removed (uninstalled); cam01/cam10 deprecated; cam05 moved inside garage 2026-05-07 (was placed in grounds_front 2026-05-08 as false-critical workaround; reverted to inside_garage zone 2026-05-17 once S2 arming gates made the workaround obsolete); all 5 IP cameras fully wired and entity naming standardised to ipcam01–ipcam05 prefix (was mixed camip/ipcam — single-pass rename across 7 config files); perimeter_rear_motion fixed (was always OFF — cam03 never existed, now uses ipcam05_back_boundary); security_event_router elevated branch added (Root Cause 1 — single-camera daytime events were silently dropped); cam07 added to confidence front tier (Root Cause 2 — intruder never fired for cam07-only events); AcuSense entrance/exit debounce sensors added (ipcam03 entrance/exit, ipcam01 entrance, ipcam02 entrance stub); vehicle_approaching correlation fixed (street cameras only — gate is closed structure, driveway = inside boundary); vehicle_in_driveway new correlation added; security_visitor trigger corrected to street cameras only (ipcam01/02 + entrance sensors); security_arrival_detected uses ipcam03_entrance_valid as primary with motiondetection fallback; notification spam fixed (event_router now sole notifier, individual automations retain state/logbook only); critical message fixed (contextual — checks inside motion at fire time vs hardcoded "inside house"); docs updated (SECURITY_CONTRACT.md + PROJECT_STATE.md).*
*2026-05-06 session: Water notification clarity — 4 files changed to fix misleading "aborted due to fault" messaging on normal pump stops. Root cause: water_refill_aborted_due_to_safety was set for (1) tank reaching max depth (NORMAL completion) and (2) permission blocks (override off, low SOC, load control) — neither is a genuine hardware fault. Fix: (1) water_safety.yaml water_stop_refill_at_max_depth no longer sets the abort flag; lifecycle now shows "Completed (Full)" instead of "Aborted (Safety)"; notification changed to "Tank Full" title with clear success language. (2) water_tank_refill_control.yaml water_block_refill_when_not_allowed no longer sets the abort flag; added notification with specific reason detection (master switch off / load control override / battery too low / grid offline). (3) water_refill_capture.yaml start/end notifications: removed [Code: ...] prefixes, added descriptive titles ("Refill Started"/"Refill Complete"), added depth rise delta to end message. (4) water_templates.yaml: sensor.water_refill_outcome_display now shows "Completed (Full)" when end depth >= 1.95m; sensor.water_refill_blocked_reason priority 1 reworded from "pump halted due to fault" to "Last refill stopped by fault protection — check borehole or pump" (now only fires for genuine no-rise stops). Abort flag now has single clear meaning: borehole no-rise protection fired. ha core check passed. Reload: automations + template entities.*
*2026-04-29 session (consolidated): Recovery mode fix — YAML/Jinja2 Rule 6 added to CODING_STANDARDS (notify_power_event + notify_security_events had {% if %} inside YAML keys → split to choose: branches). ha-biesie-docs public docs repo created + sync_docs.sh auto-sync wired into every gitupdate.sh commit. FETCH_URLS.md (24 URLs for Claude chat context seeding). AUTOMATIONS_AUDIT.md created — 9 active automations remain in automations.yaml (7 power session, 2 geyser DO NOT TOUCH). alerts_garden.yaml + GARDEN_CONTRACT.md (pond pump unscheduled-run alert pipeline). M1 entertainment mode: lighting_entertainment.yaml populated, entertaining_mode guard added to both kids_bedtime automations. M2/M3 energy saving: helpers + lighting_energy_saving.yaml populated; morning clear added. 3 automations deleted/migrated from automations.yaml (pond pump, 3 load-shedding warning blocks). IDS stub created (security_alarm.yaml). CTX01/02/03 context cleanup: context_presence.yaml → presence_trust.yaml, context_schedules.yaml deleted, home_context rewritten to remove security/ import.*
*2026-04-30 session: Doc sweep — verified live against actual files: BUG-NET01/02/03 confirmed fixed in network_helpers.yaml (availability references corrected, packet loss removed from health score formula). alerts_presence.yaml 158 lines fully implemented (not 17-line stub — contract was stale). borehole gate already marked done. Infra domain table updated (all 4 bugs fixed). Network sensor reliability (unifi_cpu_5m_max, unifi_memory_5m_max, wan_health_score) — formula fixes confirmed in code; live value reliability (whether UniFi integration feeds valid data) is a runtime check, not code — verify in Developer Tools → States.*
*2026-04-30 session: average_night_consumption sampling bug fixed — energy_state.yaml: removed sampling_size: 20 (at 5s solarman poll rate it gave ~100s effective window, silently overriding max_age: 24h because HA statistics platform uses whichever limit is hit first). Now uses max_age: hours: 24 only → true 24h rolling mean. ALERTS_CONTRACT updated: alerts_presence.yaml line count corrected 17→~158 (was fully implemented 2026-04-16, line count never updated). Infra domain table updated: all 4 bugs marked fixed (were already confirmed fixed in session log but table still said "4 bugs found"). Water borehole gate confirmed ✅ done (already in Load Control section line 236, was just stale in old TODO text). BUG-CTX01/CTX03 fixed (see below).*
*2026-04-30 session: BUG-CTX01 fixed — context_presence.yaml content moved to packages/presence/presence_trust.yaml; context_presence.yaml deleted. Entity IDs unchanged. BUG-CTX03 fixed — sensor.home_context in context_global.yaml no longer imports sensor.security_mode or sensor.security_trust_mode from security/; now derives from binary_sensor.security_nobody_home (self-contained in context_global) + binary_sensor.night_confirmed; trust attribute simplified to low_trust/normal via binary_sensor.low_trust_present. ha core check passed. context/ package: 4→2 files. presence/ package: 5→6 files. All three context bugs confirmed closed.*
*2026-04-30 session: Telegram backslash escape fix — all 6 notify scripts had 18-character MarkdownV2 escape chains applied (BUG-N09 fix 2026-04-13), but the Telegram bot integration config has `options.parse_mode: markdown` (old Markdown, not MarkdownV2). Old Markdown does not recognise backslash escapes for `.`, `(`, `)`, `+`, `|` etc., so they appeared literally in all messages. Reduced escape chain to 4 characters in all 6 scripts: `\\`, `\*`, `\_`, `` \` `` (only characters special in old Markdown). NOTIFICATIONS_CONTRACT.md BUG-N09 updated with revised history. Rule: do NOT expand escape chains beyond these 4 unless the Telegram integration is switched to parse_mode: markdownv2.*
*2026-04-30 session: WAN outage 04:30–05:00 caused cascade of Telegram ConnectTimeout errors (geyser AM/PM at 04:30, bar via presence at 04:45, git push at 05:00). Root issue: `continue_on_error: true` was on the Telegram send INSIDE notify scripts but NOT on the calling `script.notify_lighting_event` action — ConnectTimeout propagates as "Unexpected error" which bypasses the inner guard. Fixed: 23 call sites across 7 lighting files (lighting_bar_presence +6, lighting_evening +3, lighting_morning +4, lighting_arrival_night +3, lighting_boundary +2, lighting_garage +2, lighting_office_presence +3). gitupdate.sh hardened: now skips commit+push if `git diff --cached --quiet` (nothing staged) → exits 0 cleanly instead of propagating git's exit 128 when there's nothing to push. github.yaml backup failure message: added `| default('no stderr')` guard on backup_result['stderr'] (was None, causing Jinja2 error in the failure notification). Prepaid Repairs error (notify.std_warning in automations.yaml legacy automation) — confirmed already documented in session notes; fix via HA UI only (DO NOT TOUCH automations.yaml rule).*
*2026-05-18 session (security classifier hardening — BUG-S18/S19/S20 + FMT-01): BUG-S18 — false CRITICAL INTRUDER at night with `home=all`: root cause was RUNG 3 excluding `inside_armed_active`, so cam14 motion after 23:00 (inside_cameras_armed=on) bypassed family_movement and hit RUNG 8 simultaneously with NVR cam04 headlight triggers outside. Fix: (1) removed `and not inside_armed_active` from RUNG 3 — when family is home, ALL inside motion → family_movement regardless of arming state; (2) added `and not anyhome and not staff` guard to RUNG 8 — critical_intrusion only fires when property is genuinely unoccupied. BUG-S19 — staff on site spam (maid/gardener all-day cam09 motion): added 10-min cooldown to service_person branch in security_event_router using `input_datetime.last_visitor_event`; sets datetime after each notification fires, suppresses if within 600s. BUG-S20 — arrival notification spam (3-5 per arrival event): event_router was firing for every camera trigger during arrival sequence as classifier flipped idle→arrival repeatedly. Fix: arrival and departure branches in security_event_router now log to logbook only; stage1/stage2 automations handle all arrival/departure notifications. OPT-8.4 — cam14_lounge_motion_valid and cam15_passage_motion_valid added to TOP of security_trigger_camera priority list (was causing trig=none when inside cameras fired alone). FMT-01 — reason attribute reformatted: changed from flat `zones=PG---` format (YAML `>` folding collapsed all to one line) to 3-line output using `'\n'` in Jinja2 string concatenation: line 1 = zones/gate/home, line 2 = arriving/departing/staff, line 3 = conf/cam. Also changed all `=` separators to `:`. Extended RUNG 4 (service_person) to include `inside_any` zones — staff legitimately enter the house; inside camera triggers during scheduled staff hours → service_person not critical_intrusion. SECURITY_CONTRACT Sprint 7 section added with all 5 items documented. Files changed: security_logic.yaml (RUNG 3/4/8 + trigger camera priority + reason attribute), security_automations.yaml (service_person cooldown + arrival/departure logbook-only). Reload: template entities + automations.*
*2026-05-17 session (camera fleet cleanup): cam01_street_driveway, cam06_front_entrance, cam10_pool_bar deprecated/uninstalled entities purged from packages/security/ (input_text helpers, history_cleanup entries, pyscript list). cam05 renamed cam05_front_driveway → cam05_inside_garage across cameras_processing, security_zones, security_logic, security_automations, security_helpers, history_cleanup, pyscript. NVR hardware entity cam05_front_driveway_motiondetection unchanged (integration-provided). Active fleet locked: 7 NVR (cam04/05_inside_garage/07/09/12/14/15) + 5 IP (ipcam01-05). SECURITY_CONTRACT Section 3 rewritten as authoritative zone map. Entity registry orphans for cam01/06/10 (image.*, sensor.*, input_text.*) need manual Settings → Entities cleanup. Dashboard: cam01 has 13 refs in lovelace.dashboard_operations — needs user attention.*
*2026-05-17 session (security S3 — single notify path): Triple-fire dedup gate (2026-04-16, via binary_sensor.security_intruder_active 30s delay_off) was a Band-Aid — the trio still sent notifications (correction: the trio was ALREADY modified to NOT send notifications by some later session; at S3 pre-flight, trio was found to only manage security_event_active state + logbook + last_security_event). Architectural fix: security_event_router rebuilt to trigger on sensor.security_event_classification (S2 presence-first classifier), dispatch by classifier output, mode: queued. Deleted: security_grounds_motion, security_rear_grounds_motion, security_house_motion, security_visitor, security_arrival_detected. Added: security_arrival_stage1_vehicle / _stage2_confirm, security_departure_stage1_vehicle / _stage2_confirm (two-stage arrival/departure with AP confirmation at 3.5min). Visitor classification → critical severity (session decision). DSC alarm hook stubs in intruder + critical_intrusion branches. notify_security_event severity confirmed: must use 'information' not 'info'. SECURITY_CONTRACT Issues 2 (triple-fire), 7 (from: constraints partial), 8 (trust filtering via classifier) marked resolved/deferred. Reload: automations.*
*2026-05-17 session (workflow hardening): CODING_STANDARDS Rule 7 strengthened — docs-with-code is now blocking, not advisory. SESSION_CHECKLIST.md created at repo root — visible session-close enforcement mechanism. NVR placeholder slot devices (cam02/03/11/13/16-future) removed; NVR roadmap documented: all future additions will be IP-based, existing NVR fleet progressively replaced.*
*2026-05-17 session (cam05 placement correction): cam05 reverted from grounds_front to inside_garage zone. cam05 watches garage interior, not driveway approach (ipcam03 = driveway). The 2026-05-08 grounds_front placement was a workaround for false critical_intrusion alerts in the old single-zone model. With S2 zone arming gates (inside_garage_armed defaults OFF), the workaround is no longer needed and was causing real classification bugs (garage interior motion routing through intruder path). Removed from security_grounds_front_cameras group in cameras_core.yaml. Docs corrected: SECURITY_CONTRACT cam05 table + garage design block.*
*2026-05-17 session (inter-sprint doc reconciliation — S1+S2): P01/P02 confirmed already fixed 2026-04-15 (PROJECT_STATE was accurate; PRESENCE_CONTRACT bug list was stale). P10 was N/A — two staff vars in different sensors, not duplicates. Startup sync confirmed in presence_trust.yaml not presence_boundary.yaml. BUG-P03 body corrected — actual pre-fix state was maid-Monday-only logic (not the deprecated input_datetime situation described). Section 2 file inventory updated: presence_trust.yaml documented as startup sync owner; orphan helpers (input_boolean.low_trust_present / staff_on_site) noted as harmless. zone aggregation table updated. Architecture Rule 7 added (zone arming separates from zone motion). Locked entity names added for all S2 entities. [NOTE: cam05 entry in this reconciliation was wrong — corrected in same-day follow-up commit above.]*
*2026-05-17 session (S2 — Presence-First Classifier + Three-Zone Inside Split): New sensors added: binary_sensor.security_inside_garage_motion (cam05), security_inside_main_motion (cam14), security_inside_bedrooms_motion (cam15), inside_garage_armed / _main_armed / _bedrooms_armed (DSC-ready stubs backed by input_boolean.*_armed_manual), binary_sensor.family_arriving / family_departing / all_family_home (AP-location recency, 600s window, room names from unifi_ap_room_map). sensor.security_event_classification replaced: 5-state simple → 9-rung presence-first ladder (idle/arrival/departure/family_movement/service_person/visitor/perimeter_threat/intruder/critical_intrusion) with reason + zone_label attributes. security_inside_house_motion updated to raw union of three zone sensors (arming gate moved to classifier). SECURITY_CONTRACT Section 11 status: Design → Implemented. BUG-S14/S15/S16/S17 and BUG-P13 closed. ha core check passed. Reload: template entities + helpers.*
*2026-05-17 session (S1 — Trust Layer Rewire + Startup Sync): Pre-flight verified trust chain swap (P01/P02) already applied — zero reads of input_boolean trust entities in code. S1.3: boundary_permissive_window rebuilt (security_core.yaml) — old maid-Monday-only logic replaced with binary_sensor.low_trust_present OR guest_mode OR boundary_permissive_override; gate_open_too_long_permissive now fires for all staff/contractor/guest scenarios. S1.4: arrival_detected wired — both Pass 1 and Pass 2 arrival branches in presence_boundary_resolver now set input_boolean.arrival_detected; presence_clear_arrival_flag auto-clear automation added (5 min, mode: restart); lighting_arrival_night and security_arrival_detected now fire on real arrivals. S1.6: startup sync extended — gardener_on_site now restored on HA restart if inside gardener window and low_trust_enabled; contractor excluded (manual-only, HA state restore sufficient). BUG-P10 confirmed N/A — two staff vars are in separate sensors, both correct. GROUP A correction: A2 was only partially fixed 2026-04-15; fully corrected today. PRESENCE_CONTRACT updated: P01/P02/P03/P06/P10/P11/P12 all closed. Reload: template entities + automations.*
*2026-04-15 session (Group A audit): A1 confirmed superseded — low_trust_start/end never needed; complete trust chain runs through maid_start/end + gardener_start/end datetimes → schedule automations → maid_on_site/gardener_on_site booleans → binary_sensor.staff_on_site → binary_sensor.low_trust_present. A2 confirmed done — boundary_permissive_window reads maid_on_site + weekday()==0 + override, no longer always false. Group A fully complete.*
*2026-05-10 session: security_pool_alarm_trigger — tiered threat gate. Before: required `security_night_mode` ON + threat in ['warning','critical']. After: arms from 20:00 (night_mode OR hour>=20); critical always fires; warning fires when nobody home OR (evening + not night_mode); warning suppressed when family asleep (night_mode ON + home). Rationale: old logic missed the 20:00–22:30 window when people are still awake outside; family-asleep suppression prevents nuisance wakeups for low-confidence rear motion (dogs/wind). SECURITY_CONTRACT.md updated with pool alarm behaviour.*
*Last updated: 2026-05-08 — Security camera system restructure + entity rename complete (ipcam01–ipcam05 standardised — see SECURITY_CONTRACT.md for full detail)*
*2026-04-28 session (pool pump bugs): pool_pump_solar_control in power_automations.yaml — Bug 1: no 16:00 hard-stop trigger. Pump turned on at 15:00 by Branch 5 (solar surplus), ran until manually turned off at 18:05 (confirmed via DB: switch state change had empty context = physical/manual, not automation). Fix: added "16:00:00" time trigger (id: end_of_day); added Branch 0 (unconditional 16:00 turn-off, no minimum run time guard); removed before:"16:00:00" from global conditions; added before:"16:00:00" to Branch 5 turn-on conditions only. Bug 2: input_datetime.pool_pump_last_on never updated — mode:single caused Branch 1 (records pump start time) to be silently dropped (max_exceeded:silent) when it queued behind Branch 5 that had just turned the pump on. Result: pool_pump_continuous_run_minutes always showed ~1085 min (today_at('00:00:00') = midnight), so 45-min minimum run time guard was always trivially met. Fix: mode:single → mode:queued.*
*2026-04-29 session: HA Repairs "Prepaid Strategic Top-Up uses notify.std_warning" — confirmed transient error, no code change needed. automation.prepaid_strategic_top_up_suggestion is only in prepaid_core.yaml (correctly calls script.notify_power_event); no duplicate in automations.yaml. Error occurred at 22:29:11 during script reload window when notify services were briefly unavailable. API query confirmed all 4 notify groups live: std_information, std_critical, std_alerts, std_warning. HA Repairs notification dismissed. automations.yaml lines 1660+1708 (grid usage warning, legacy) use notify.std_warning directly — intentional, DO NOT TOUCH.*
*2026-04-29 session: Notify script Telegram audit — disable_notification removed from all 6 scripts (notify_light_events, notify_system_event, notify_water_events, notify_presence_events, notify_power_event, notify_security_events). Key was invalid in notify.send_message schema: top-level in light script, nested data.data in the rest. Caused runtime errors: garage lighting automation → notify_lighting_event failed; weather_api_recovery → notify_system_event failed. All 6 scripts confirmed clean against CODING_STANDARDS Rule 6 (all {% if %} blocks inside block scalars, none emitting YAML keys) and Telegram schema (message/title at data root; inline_keyboard under data.data in critical branches only; continue_on_error: true on all send_message calls). EcoFlow sensor.river_pro_ups_main_remain_capacity unit mismatch (mAh vs mA/A for device_class current) — third-party custom component bug in ecoflow_cloud; no HA config change possible, report to github.com/snell-evan-itt/hassio-ecoflow-cloud-US/issues.*
*2026-04-29 session: Garden alert pipeline: alerts_garden.yaml created — binary_sensor.garden_alert_active (pond pump after 11:00, 1min/5min delay), sensor.garden_alert_context, alert.garden_alert (60min repeat, STD_Alerts), garden_alert_ack_turn_off_pond_pump automation (mobile_app_notification_action handler — fixes broken wait_for_trigger-in-notify-data from original). alert.garden_alert added to alerts_summary.yaml aggregator trigger list. Pond pump automation 1742999668407 deleted from automations.yaml (replaced with migration comment). update_device_uptimes_group + device_restart_info_alert_active confirmed already absent from automations.yaml (HA reformatted file on last reload). Both superseded by alerts_network.yaml pipeline. automations.yaml now 1,968 lines, 9 active automations. GARDEN_CONTRACT.md created. ALERTS_CONTRACT.md updated: 12→13 files, garden domain pipeline audit added, aggregator trigger list corrected. ⚠️ RESTART REQUIRED — alert.garden_alert is an alert: entity; alert: entities cannot be reloaded (CODING_STANDARDS Reload vs Restart Rules). Binary sensor + context sensor will load on template reload; alert entity only activates after full HA restart.*
*2026-04-29 session: TASK 1 M1 entertainment mode — lighting_entertainment.yaml populated (4 automations: button→boolean, boolean ON→scene, boolean OFF→evening restore gated on civil_night, 06:00 daily clear). All wired to input_boolean.entertaining_mode (context_presence.yaml) — same concept as entertaining guests; duplicate input_boolean.entertainment_mode removed from lighting_helpers.yaml after user clarification. kids_bedtime_week + kids_bedtime_weekend: entertaining_mode guard added via choose block — if ON, skip pool light; turn off other 4 lights individually. TASK 2 energy saving — M2 helpers fixed (threshold 25%, icon mdi:lightning-bolt-outline); M3 already done in prior micro-session; morning_wake_lights_on now clears energy_saving_mode (lighting_morning.yaml). TASK 3 IDS stub: security_alarm.yaml created — zero IDS refs found in automations.yaml; provisional entity interface documented; IMP-IDS01 closed. TASK 4 water borehole gate: load_control_borehole_enabled added to binary_sensor.water_refill_allowed condition (water_templates.yaml:299); borehole_enabled attribute added. TASK 5 load shedding migration: announce_load_shedding_stage + load_shedding_warning_15 + load_shedding_warning_2hr migrated to packages/load_shedding/load_shedding_automations.yaml (fixes: notify.STD_* → script.notify_system_event, whatsapp removed, dead notify.mobile_app_ap_0223_1001 removed); blocks commented out in automations.yaml with migration banner; ha core check passed. Inverter Scene Switcher deferred to power session. TASK 6 automations.yaml full audit: AUTOMATIONS_AUDIT.md created in docs/. 12 active automations remaining (was 3,836 line file, now 2,915 lines). Power session owns 7 automations; geyser session owns 2; 3 ready to migrate now. Dead entity flags: 8 automations still reference sensor.inverter_power.*
*2026-04-29 session: Camera health alert implemented — alerts_camera_health.yaml created (⚠️ requires HA restart). Monitors video loss (binary_sensor.camXX_videoloss) and motion staleness (sensor.camXX_last_seen_seconds > 8h outdoor / 24h indoor, 08:00-20:00 only, only when security_system_enabled ON). One domain alert.camera_health, per-camera fault detail in sensor attributes and notification. Telegram mirror via automation → script.notify_security_event. alert.camera_health added to alerts_summary.yaml aggregator trigger list. Active faults: cam01 (NO VIDEO), cam04 (NO VIDEO), cam12 (video loss — cam11_back_pond entities; binary_sensor.cam11_back_pond_videoloss ON). cam12 entity naming confirmed: NVR channel is Cam12-Back-Pond but HA config uses cam11_back_pond IDs. Fix for all three: draw motion detection areas + check NVR channel signal in Hikvision NVR UI.*
*2026-04-29 session: BUG-L10 FIXED — kids_bedtime_week + kids_bedtime_weekend: added continue_on_error: true to both script.notify_lighting_event calls (pre-notification and post-scene). Root cause of bedtime failure on 2026-04-28 night: T2 fix introduced illegal {% if %} Jinja2 in notify_power_event.yaml + notify_security_events.yaml; those files caused script load error at reload time; when notify_lighting_event was called (no continue_on_error), HA may have been in a partial recovery state causing the call to fail, stopping the automation before scene.turn_on. NOTE: DB query earlier in session incorrectly showed switch entities as unavailable — this was historical data from April 24-25 outage; recorder does not write new entries while state unchanged after recovery. Sonoff integration IS working (confirmed via UI screenshot). BUG-L10 entry corrected from Sonoff-outage to notification-hardening issue.*
*2026-04-29 session: BUG-L10 found — Sonoff integration offline since ~2026-04-24. DB query confirmed switch.front_house_security_light, back_house_security_light, entrance_down_lights, dining_room_light, pool_light_switch all showing only "unavailable" state since April 24 (no on/off transitions recorded). Kids bedtime automation (kids_bedtime_week/weekend) condition: or check uses state:"on" — all unavailable switches evaluate to false, condition fails after setting bedtime_mode, scene never fires. Physical lights stayed on in last relay state. Root cause: Sonoff (EWElink) cloud integration down — check Settings → Integrations → Sonoff. Both switches are sonoff platform, config_entry 01JHTXM2, device 1001e1f931. LIGHTING_CONTRACT.md updated: L05/L07/L08 marked fixed; scene_kids_bedtime inventory corrected to 5 lights; helper inventory pruned (kids_bedtime_enabled + patio_second_wake_time removed); entertaining_mode entity name clarified (input_boolean.entertaining_mode exists in context_presence.yaml; button name mismatch "entertainment" vs "entertaining"). M1 planning note in GROUP M corrected — no new boolean needed, wire button to existing input_boolean.entertaining_mode.*
*2026-04-29 session: Recovery mode caused by illegal % token in notify_power_event.yaml line 198 and notify_security_events.yaml — both had {% if sev == 'critical' %} blocks used to conditionally include the inline_keyboard YAML key inside a data: mapping. HA's YAML parser sees {% at structural YAML level (where a key would appear) as an illegal % token and enters recovery mode. Fixed both files by splitting the Telegram notify.send_message into a choose block: critical branch includes inline_keyboard, default branch uses disable_notification. Both files verified correct: critical branch → inline_keyboard present, default branch → disable_notification present, no {% %} at key level. CODING_STANDARDS Rule 6 added: never use Jinja2 block tags to conditionally emit YAML keys — use choose: branches instead. Rule also added to pre-commit checklist.*
*2026-04-29 planning: BUG-L09 gap analysis completed. Entertainment mode: scene.scene_entertainment_mode already exists (pool light + patio + entrance_down + dining); missing are input_boolean.entertainment_mode, a button→boolean automation in lighting_entertainment.yaml (currently empty), and an entertainment-mode guard in both kids_bedtime automations which unconditionally turn off switch.pool_light_switch today. Energy saving mode: input_boolean.energy_saving_mode missing entirely; lighting_energy_saving.yaml empty; architecture decision — power system owns the SOC/orchestrator trigger (power_automations.yaml); lighting side only consumes the boolean. Manual override buttons exist (energy_saving_mode_on/off). Implementation captured in Group M (M1/M2/M3). Not yet implemented.*
*2026-04-28 session 3: T1 fixed — telegram_bot.send_photo added to notify_security_events.yaml after Telegram text message, guarded on img not none. T2 fixed — inline_keyboard added for critical severity in security (→ /ack_security_alert → alert.security_alert off) and power (→ /ack_power_alert → alert.power_alert off); callback automations added to respective notify files. T3 fixed — disable_notification: sev==information added to all 6 Telegram send_message calls. T4 fixed — push.interruption-level: critical (iOS) + channel: alarm + ttl/priority (Android) added to all 6 STD_Critical notify calls. BUG-WEA01 confirmed already fixed (checkbox not updated). BUG-CORE01 fixed — ha_events_per_second removed (was returning total sensor count not rate). BUG-CTX02 fixed — bedtime_mode moved to lighting_helpers.yaml, bedtime_time deleted, context_schedules.yaml deleted. BUG-L07/L08 fixed — kids_bedtime_enabled + patio_second_wake_time removed (no consumers). Only BUG-L09 remains open.*
*2026-04-28 session 2: BUG-INF01 fixed — removed dangling }} from binary_sensor.printer_cartridge_low. BUG-NET03 fixed — packet loss variables removed from WAN health score (ping_sum_5min is latency sum not pass count; formula was subtracting ms from integer). BUG-NET01 fixed — unifi_cpu_5m_max availability was referencing non-existent sensor.unifi_cpu → corrected to unifi_gateway_cpu_utilization. BUG-NET02 fixed — unifi_memory_5m_max availability was self-referencing → corrected to unifi_gateway_memory_utilization. BUG-L02 fixed (correctly this time) — main_entrance_light removed from scene_night_away (it stays ON overnight as deterrence with boundary lights), explicit switch.turn_off added to morning_wake_lights_on action so it turns off with the morning routine. BUG-L01/L03/L04/L05/L06 confirmed already fixed in code — checkboxes updated. BUG-BKP01 fixed — github.yaml failure notification routed through script.notify_system_event (warning severity bypasses quiet hours); success notification removed (5AM noise). IMP-IDS01 noted as intentionally deferred.*
*2026-04-28 session: number.inverter_battery_capacity TemplateError fixed — |int with no default throws when either solarman entity is unavailable at startup. Fixed |int → |int(0) for both inverter_1 and inverter_2 capacity in the UI template config entry (entry_id: 01JKJES2XR88FXQ8TZHST4YA46). Note: this fix was documented as done in the 2026-04-22 session C notes but did NOT persist — storage entry still showed modified_at: 2026-01-15 when re-examined. Root cause likely: HA rewrote the storage file after the edit (config entry was reloaded via API which triggers HA to overwrite the file from its in-memory state). Fix re-applied and verified — entity reads 600.0 Ah immediately. POWER_CONTRACT updated with UI-created template number section and re-application warning.*
*2026-04-23 session (performance): Recorder excludes expanded in configuration.yaml — added sensor.wan_*_5min_max, sensor.udm_*_5min_*, sensor.average_night_consumption, sensor.house_outdoor_power_clean, sensor.prepaid_drift_rate_per_day. Root cause: statistics/filter platform sensors recalculate on every source entity update (no minimum interval configurable); unifi_*_stats and *_ping_* were already excluded. Recorder excludes only stop DB writes — recalculation continues (negligible CPU for arithmetic templates). OPEN BUG: sensor.average_night_consumption has sampling_size: 20 bug — at solarman 5s poll rate, effective window is ~100s not 24h. Fix: throttled intermediary sensor (5-min trigger) + sampling_size: 288 → true 24h mean. Requires configuration.yaml reload → ⚠️ full HA restart required.*
*2026-04-22 session (power Session E): E1 — geyser switch rename confirmed done; one missed reference (load1_switch in dashboard_operations line 827) fixed. E2/P4-1 — sensor.prepaid_net_position_this_month is always available (no availability guard, all inputs use float(0)); returns 0 at month start before utility meters accumulate; no broken dependencies. P4-3 — sensor.prepaid_balance_confidence added to prepaid_core.yaml: 0-100% graduated score on drift magnitude + -2%/day penalty >30 days since last meter verification (max 30% penalty). P4-4 — input_number.prepaid_drift_threshold (default 2%) added to prepaid_helpers.yaml; prepaid_auto_reconcile automation added to prepaid_core.yaml: triggers on prepaid_meter_lifetime_import change, condition drift>threshold, calls script.prepaid_realign_offset + script.notify_power_event.*
*2026-04-22 session (power Session D — fixes): sensor.pool_pump_run_hours_today history_stats never loaded — history_stats platform sensors in packages silently fail when the source entity (switch.pool_pump_switch) has no recorder history at load time. Removed history_stats definition from power_state.yaml; must be created as UI Helper (Settings → Helpers → History Stats). sensor.pool_pump_solar_headroom orphaned entity removed from core.entity_registry (stale entry from before rename to solar_available_surplus, was showing cached -295W). POWER_CONTRACT updated to document UI-only history_stats requirement.*
*2026-04-22 session (power Session D — continued): Solar surplus metric corrected — solar_export_potential reads near-zero while battery fills (subtracts battery charging). Replaced with sensor.solar_available_surplus = pv_power - load_power (single shared metric for all optional loads; each appliance checks its own threshold). Pool pump thresholds: turn on >1200W, turn off <800W (hysteresis). sensor.pool_pump_control_status added — human-readable reason string for dashboard. All pool pump automation notifications now route through script.notify_power_event (HA app + Telegram mirror). Default branch fires hourly when solar active + pump off — Telegram trace of why pump didn't run. Entity renames: switch.pool_pump_switch_1 → switch.pool_pump_switch and switch.geyser_heat_pump_switch_1 → switch.geyser_heat_pump_switch across all YAML + 3 dashboard storage files (operations/overview/testing).*
*2026-04-22 session: BUG-A09 fixed — sensor.critical_sensor_health_alert_context devices attribute had inline Jinja2 `# comment` on same line as {{ ns.items[:20] }} inside a YAML > block scalar. YAML does not treat # as a comment inside block scalars; Jinja2 appended the literal text to the list output, making it unparseable as a list. alert_device_entities aggregator check `devs is not string` silently failed — critical sensor health never appeared in global alert summary. Fixed by converting to {# Jinja2 comment #}. Same fix applied to sensor_health_overview_context. Prepaid Strategic Top-Up Suggestion stale HA Repairs error: automation YAML already uses script.notify_power_event (correct) — stale Repairs notification, dismissed via UI. Note: automations.yaml lines 2318+2368 (grid usage warning) still use notify.std_warning (lowercase) — DO NOT TOUCH (legacy automations.yaml).*
*2026-04-22 session (power Session D — pool pump migration): D0 pre-flight: sensor.load_shedding_stage → actual entity is sensor.load_shedding_stage_eskom (attr stage: int); sensor.load_shedding_next_start → use sensor.load_shedding_minutes_remaining; sensor.battery_runtime_estimate → use sensor.ss_soc_battery_time_left. D1: 5 pool helpers added to power_helpers.yaml (pool_minimum_run_minutes/pre_shed_battery_threshold/target_hours_summer/_winter/pool_pump_last_on). D2/D3: pool_pump_run_hours_today (history_stats) + pool_pump_continuous_run_minutes + pool_target_run_hours_today template sensors added to power_state.yaml. D4: pool_pump_solar_control added to power_automations.yaml — solar-surplus-aware, load-shedding-gated, minimum-run-time-protected, season-aware target hours. D5: 4 legacy pool pump automations (IDs 1742227789739/1742294033785/1742294477609/1742295582190) deleted from automations.yaml; migration comment left at deletion point. D6: POWER_CONTRACT pool pump section added.*
*2026-04-22 session (power Session C): sync_power_groups.py: (1) @time_trigger("startup") added so load groups populate on HA start without manual service call; (2) from homeassistant.helpers import entity_registry blocked (not in pyscript ALLOWED_IMPORTS) — replaced er.async_get(hass) with hass.data.get("entity_registry") (HassKey is str subclass, plain string lookup works). battery_runtime.yaml: 6× |int → |int(0) on program_N_soc lookups (unavailable at startup caused TemplateError). energy_state.yaml: removed device_class: energy from battery_energy_available (incompatible with state_class: measurement — device_class: energy requires total/total_increasing; this is a point-in-time Wh value). .storage/core.config_entries: number.inverter_battery_capacity UI-template fixed |int → |int(0) for both inverter capacity entities (cannot edit via YAML — lives in config entries storage). C3 pool pump research: 4 automation blocks found in automations.yaml lines 2222–3174 — IDs 1742227789739/1742294033785/1742294477609/1742295582190 (Turn Off PM / Turn On Early AM / Turn On AM 11-1 / Turn On PM 1-4). All 4 use dead sensor.inverter_power, notify.STD_Information, device_id-based control, and indirect solar mode via input_select.inverter_solar_appliance_mode_helper. Load shedding conditions partially disabled (enabled:false). Research only — no changes.*
*2026-04-22 session: BUG-N11 fixed — notify_power_event.yaml all 3 severity branches used {{ title }} with no default guard; HA logged "title is undefined" on every render. Added | default('Notification') to match notify_water_events.yaml pattern.*
*2026-04-21 session (power + load control): energy_core.yaml — grid_to_house_power now subtracts grid_to_battery_power (was assigning all grid import to house); solar_to_battery_energy_today and grid_to_battery_energy_today template sensors removed — both already exist as UI utility meters (created 2026-03-20, would have created _2 duplicates); grid_used_by_house_today now uses grid_energy_import_today minus grid_to_battery_energy_today (UI meter). CODING_STANDARDS.md Rule 4 added: always check .storage/core.entity_registry before creating any entity in YAML. load_control.yaml created: input_boolean.load_control_geyser_enabled/pool_enabled/borehole_enabled + 3 automations that turn off running devices when disabled. recorder purge_keep_days set to 90. pyscript/power_snapshot.py added.*
*2026-04-21 session: Removed 3 UI-created storage duplicates causing startup errors. counter.water_borehole_faults_today and counter.water_borehole_faults_this_week removed from .storage/counter (both already defined in water_helpers.yaml since 2026-04-16). input_boolean.notifications_enabled removed from .storage/input_boolean (already defined in notifications_helpers.yaml since 2026-04-15 as BUG-N03 fix). Requires HA restart to take effect.*
*2026-04-21 session (cont): BUG-NET04 fixed — sensor.network_device_down_alert_severity and sensor.network_alert_context devices attribute both only filtered state=='off', missing 'unavailable'. group.network_devices (all:true) goes 'off' when any member is unavailable, correctly firing the alert binary sensor, but severity evaluated down=0 → none → context stayed normal → global summary showed nothing. Fixed both selectors to ['off','unavailable']; unavailable devices now labelled "Unavailable" in alert detail.*
*2026-04-21 session: UniFi AP MAC addresses updated in presence_core.yaml for new/replaced hardware (Bar, Office, Garage, Lounge, Bedroom Zone). AP monitoring migrated from ping platform (UI-configured binary_sensor.ap_*_ping) to UniFi controller state (binary_sensor.ap_*_connected wrapping sensor.ap_*_state == 'connected') in network_helpers.yaml. group.network_devices updated. Old ping entries for APs can now be removed via UI. Kitchen AP not yet adopted — excluded.*
*2026-04-20 session: Disabled two enabled notify.whatsapp steps in Load Shedding Inverter Scene Switcher (automations.yaml Stage 4+6 branches). Decided to drop WhatsApp entirely — no viable free replacement. Added Group T (Telegram enhancements): camera snapshot on security events, inline keyboard ack buttons, disable_notification for info level, mobile alarm sound for critical.*
*Previous: Three-batch session — BUG-A06 door unification, water alerts expansion, presence pipeline + triple-fire dedup*
*2026-04-15 session (Group D+E): D1 fixed — security_lighting_allowed entity typos corrected; D2 fixed — cam06 last event now reads cam06 not cam09; D3 fixed — cam10_pool_bar_motion_images renamed to cam10_pool_bar_images; D4 fixed — unique_id added to security_correlation; D5 fixed — presence_test_arrival removed; E1 fixed — from: constraints added to water_borehole_no_rise_protection + water_reconcile_cycle_state; E2 fixed — for: stability window added to water_refill_visibility_guard; E3 verified — water_refill_allowed gate confirmed correct.*
*2026-04-13 session: BUG-N08 fixed (escaped_title/message defaults); BUG-N09 fixed (MarkdownV2 . and ! missing from escape chains — caused Telegram BadRequest on all float values); alerts_power template fix; camera snapshot continue_on_error; automations max_exceeded:silent; Lovelace template errors; prepaid weekday trigger fixed (sunday→sun list); BUG-N10 fixed (Group F — 8 missing from:/not_from: guards across notifications, lighting, power)*
*2026-04-14 session: BUG-N01 fixed (continue_on_error on Telegram in notify_water_events); sev default guard fixed in notify_water_events; prepaid_days_since_verification none guard fixed; Lovelace: config.entity replaced with config.get()/static styles on all alert+power cards; door sensor last_changed guarded against None at boot (6 cards); continue_on_error added to all 10 direct Telegram calls across 5 files (solar_forecast x3, water_reporting x1, water_tank_refill_control x2, github x2, lighting_bar_presence x2) — fixes ServiceNotFound on restart*
*2026-04-15 session: sensor.load_visibility_score ZeroDivisionError fixed — float(1) default didn't guard against actual 0 value; added if total > 0 else 100 guard (power_templates.yaml); sensor.printer_cartridge_state UndefinedError fixed — min on empty sequence when printer is off; added rejectattr+list+none guard (office_helpers.yaml); Lovelace dashboard_operations: 6 door/gate card_mod CSS templates still had unguarded states.binary_sensor.X.last_changed — secondary text was fixed in prior session but style template was a separate code path; fixed with entity-is-not-none fallback to now() on all 6 (main_gate, garage, front_security_gate, front_door, lounge_door, bar_door)*
*2026-04-15 session (Group C): C3 fixed — notifications_helpers.yaml created, notifications_enabled now in YAML (BUG-N03); C4 fixed — 4 temperature route automations migrated from direct notify.STD_* calls to script.notify_system_event with quiet hours + Telegram (BUG-N04); C5 fixed — 4 presence unknown AP automations migrated from direct STD_Information to script.notify_presence_event with quiet hours (BUG-N05); C6 verified clean — route_device_power_alert already removed (BUG-A04). C2 verified — water_notifications.yaml:158 already uses counter.water_borehole_faults_this_week (correct name). A3/A4 verified — security_core.yaml, security_logic.yaml, lighting_departure.yaml already use binary_sensor.low_trust_present and binary_sensor.staff_on_site throughout. All were already applied; docs updated to reflect.*
*2026-04-16 session (lighting audit): 5 bugs fixed — (1) scene_night_away missing entrance_down_lights (added); (2) night departure now applies scene_morning_routine_off as second scene to catch evening routine lights; (3) departure trigger got from:"off" restart guard; (4) arrival scenario-1 binary_sensor.anyone_home→anyone_connected_home (entity didn't exist); (5) morning_wake_lights_on got time+cam14 fallback triggers with trigger-id weekday guard; scene_kids_bedtime got entrance_down_lights+dining_room_light; kids bedtime week+weekend got 30s confirmation notify + cancel button (input_button.kids_bedtime_cancel added to helpers).*
*2026-04-16 session: BUG-A06 fixed — sensor.doors_open_alert_severity deleted; sensor.door_alert_context now unified single source with full tiered logic (tier 1: gate/front security gate; tier 2: front door/garage; tier 3: lounge/bar). Two new input_numbers added: tier3_door_evening_start (21h) + entry_door_night_escalation_minutes (5 min). Water: water_tank_full_notification dead trigger fixed (binary_sensor.water_tank_full_depth). Borehole fault counters defined in YAML (were undefined). Fault escalation pipeline added: L1 notification, L2 alert (>=3), L3 critical (>=5). sensor.water_refill_blocked_reason added (priority-ordered block reason). Presence B1: full alerts_presence.yaml pipeline implemented (unknown AP + occupancy anomaly). Triple-fire dedup: binary_sensor.security_intruder_active + zone conditions added to all three intruder automations. alerts_summary.yaml trigger list updated for water borehole faults + presence_alert.*
*2026-04-21 session B (power entity fixes + migration): Session A: battery_energy_available + average_night_consumption (statistics platform, mean of inverter_load_power, 24h/20 samples) added to energy_state.yaml; battery_night_survival formula fixed (×10 for 10h Joburg night); solar_forecast.yaml syntax error line 61 fixed + 3× sensor.inverter_power→sensor.inverter_load_power; grid_risk.yaml + alerts_system_health.yaml sensor.inverter_power fixed; power_core.yaml inverter_today_energy_import doubling workaround documented with full deprecation comment; pyscript/power_snapshot.py: hass.bus.async_fire→event.fire. Session B: power_automations.yaml created as power domain automation home; grid_status_monitoring + inverter_pwer_monitoring commented out in automations.yaml (fully superseded by alert.power_alert in alerts_power.yaml); inverter_alert_battery_soc_critical migrated as power_battery_soc_critical_alert — ssa_*→ss_* sensor fix + notify.std_critical→script.notify_power_event; alerts_power.yaml: 3× sensor.inverter_power→sensor.inverter_load_power; load_shedding migrated to new packages/load_shedding/ package (⚠️ restart required); load_shedding unique_id typo fixed (load_hedding_inutes_emaining→load_shedding_minutes_remaining); sensor.inverter_device_1_since_last_update renamed (unique_id fix); house_security_power group reference fixed (security_power_sensors→house_security_power_sensors); dashboard prog1_capacity fixed (number.inverter_1_program_1_charging→number.inverter_1_program_1_soc which exists in entity registry).*
*2026-04-15 session (door alerts): Gate notification split — alert.door_alert now skip_first:true (first fire at 5 min sustained open); notify_gate_opened automation sends warning severity immediately on open (bypasses quiet hours, befitting tier 1 perimeter); notify_gate_closed sends information severity on close (suppressed in quiet hours). Removes generic "list all doors" fallback from initial low-severity trigger.*
*2026-04-14 session (security): Inside camera arming schedule added (security_inside_cameras_arming automation — arms cam14/cam15 based on time+occupancy). New booleans: inside_cameras_armed (managed), inside_cameras_schedule_override (dashboard override). Master security_system_enabled now also gates camera capture automations (was only gating notifications). Motion debounce tuned: delay_on 1–2s on all outdoor cameras to filter rain/wind; delay_off increased on rear cameras to 45s. Dashboard cards recommended: security_system_enabled + inside_cameras_schedule_override on security control card. cam01/cam04/cam07 snapshots still blank — root cause: no motion detection areas drawn in Hikvision NVR. Fix is Hikvision UI only, no HA changes needed.*
*2026-04-15 session (security classification audit): warning threshold raised to 75% (from 50%) in sensor.security_threat_level; security_event_router — added 5-min cooldown (checks last_security_event >300s), added to: "on" guard to suppress off-transition fires, now updates last_security_event after each notification; cam04 carport added to sensor.security_movement_path (was returning none for carport-only events). Architecture — all classification state names, entity names, and notification interface preserved intact for new AI camera integration. Known remaining: triple-fire from security_grounds_motion + security_rear_grounds_motion + security_house_motion all triggering on correlation→intruder simultaneously (low priority — 5-min cooldown on event_router is the primary anti-spam gate).*
