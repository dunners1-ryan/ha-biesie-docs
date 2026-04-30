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

---

## 📦 Actual Package Structure (Verified 2026-04-16)

```
packages/
  power/          # 25 files — dual inverter, solar, battery, prepaid, energy, automations, statistics
  water/          # 20 files — tank lifecycle, safety aborts, audit
  alerts/         # 14 files — see ALERTS_CONTRACT.md for actual file list (alerts_camera_health.yaml added 2026-04-29)
  lighting/       # 14 files — presence-aware and time-based scenes
  notifications/  # 12 files — routing, quiet hours, severity (includes water/power/presence/security/system scripts)
  security/       # 7 files  — cameras_core, cameras_processing, security_helpers,
                 #             security_core, security_logic, security_zones,
                 #             security_automations
  presence/       # 6 files  — presence_helpers, presence_core, presence_confidence,
                 #             presence_boundary, presence_validation, presence_trust (migrated from context/ 2026-04-30)
  context/        # 2 files  — context_global, context_night (context_presence + context_schedules deleted 2026-04-30/28)
  core/           # 3 files  — core_helpers (startup guard), ha_monitoring, sensor_smoothing
  network/        # 1 file   — network_helpers (WAN health, UniFi speed, latency, jitter, NOC status)
  alerts/ already counted above; also covers: network, temperature, system_health, media, doors, water, security, presence, power
  backup/         # 1 file   — github.yaml (5AM git backup, script, status sensor)
  integrations/   # 1 file   — sonoff.yaml (Sonoff/EWElink config)
  office/         # 1 file   — office_helpers.yaml (printer cartridge monitoring)
  weather/        # 2 files  — weather_core.yaml, weather_helpers.yaml (OWM health tracking)
  sensors/        # 1 file   — filter.yaml (sensor smoothing utilities)
  load_shedding/  # 2 files  — load_shedding_templates.yaml (migrated from power/ 2026-04-21)
                 #             load_shedding_automations.yaml (3 warning automations migrated 2026-04-29)
                 #             ⚠️ New package dir — requires HA restart if not yet restarted since 2026-04-21
  admin/          # 0 files  — EMPTY directory (reserved for future admin automations)
```

**Important:** Several `*_CONTEXT.md` documents list incorrect filenames. The `*_CONTRACT.md`
files are authoritative for actual file inventory.

---

## ⚡ Hardware Summary

### Inverters
- **Dual Master/Slave Sunsynk** — Master: grid/losses/BMS, Slave: PV/load/battery
- Aggregated in `power_core.yaml` into unified sensors
- Solar forecast: Solcast, cached in `solcast_solar/`

### Cameras
- NVR: Hikvision DS-7116HGHI-F1 (16ch hybrid DVR, analog 1080p, no AI)
- cam03_rear_perimeter: **DOES NOT EXIST** — planned only
- Streaming: go2rtc (timeout instability with NVR)
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
camera.camXX_location_description             e.g. camera.cam01_street_driveway
binary_sensor.camXX_location_motiondetection  NVR raw signal
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
| 🛡️ Security | Camera faults active | Trust chain fixed. Triple-fire FIXED 2026-04-16. alert.camera_health live 2026-04-29. Active faults: cam01/cam04/cam12 — NVR "NO VIDEO" on those channels; fix in Hikvision NVR UI (draw motion zones). cam12 entity IDs updated to cam12_back_pond 2026-04-29 (was cam11_back_pond — pre-dates NVR channel renumbering). |
| ⚡ Power | Stable | Session G (2026-04-28): pool_pump_solar_control two bugs fixed — (1) no 16:00 hard-stop: pump ran until 18:05 when manually stopped; fixed by adding "16:00:00" trigger (id: end_of_day) + Branch 0 unconditional turn-off, removing before:"16:00:00" from global condition, moving it to Branch 5 turn-on only; (2) pool_pump_last_on never updated: mode:single caused Branch 1 to be silently dropped when Branch 5 turned pump on, leaving pool_pump_continuous_run_minutes stale at ~18h; fixed by changing to mode:queued. Session F (2026-04-22): power_statistics.yaml created — inverter_production_7d_mean/30d_mean, house_load_24h_mean (statistics platform), solar_vs_forecast_ratio_7d + solar_weather_correlation (template). All 5 excluded from recorder. pv3/pv4 broken voltage entries removed from 3 dashboards (operations/overview/testing). Broken suppress-counter entity cards removed from dashboard_operations. prepaid_balance_confidence | min(30) bug fixed (must wrap in list). Dashboard card last_changed guards added for pool_pump_control_status, pool_target_run_hours_today, pool_pump_continuous_run_minutes. pyscript sync_power_groups.py: hass.* proxy is fully restricted — hass.data, hass.services, homeassistant.helpers imports, and task.executor(pyscript_fn) all blocked. Final working solution uses state.names(domain="sensor") + service.call() builtins only; label-based flexible/critical groups left empty (entity registry inaccessible from pyscript). Session D (2026-04-22): pool_pump_solar_control added to power_automations.yaml; 4 old pool pump blocks removed from automations.yaml; pool helpers added to power_helpers.yaml; pool runtime sensors (history_stats + 2 templates) added to power_state.yaml; POWER_CONTRACT pool section added. Session A (2026-04-21): grid_to_house_power fixed; energy flow sensors corrected; battery_energy_available + average_night_consumption statistics sensor added; solar_forecast.yaml syntax error + 3× sensor.inverter_power fixed; grid_risk.yaml + alerts_system_health.yaml sensor.inverter_power fixed; power_core.yaml inverter_today_energy_import deprecation documented; pyscript event.fire fixed. Session B (2026-04-21): power_automations.yaml created; grid_status_monitoring + inverter_pwer_monitoring commented out (superseded by alerts_power.yaml); inverter_alert_battery_soc_critical migrated with ssa_*→ss_* fixes; alerts_power.yaml 3× inverter_power fixed; load_shedding migrated to new packages/load_shedding/ dir (⚠️ restart required); B4 sensor rename; B5 group ref fixed; B6 prog1_capacity fixed. Session C (2026-04-22): sync_power_groups.py startup trigger added + import rewritten (hass.data.get); battery_runtime.yaml 6× int default fixed; battery_energy_available device_class removed; number.inverter_battery_capacity storage template int default fixed. Pool pump automations researched (C3 — 4 blocks, lines 2222–3174, research only). |
| 💳 Prepaid | Clean | Drift history tracking added 2026-04-23. Confidence layer done. |
| 🚰 Water | Expanded | "full" dead code fixed — water_tank_full_notification now triggers on binary_sensor.water_tank_full_depth. Fault escalation implemented (L1 notify / L2 warning alert / L3 critical alert). sensor.water_refill_blocked_reason added. Borehole fault counters defined in YAML. |
| 🔔 Alerts | All domains live | alert.camera_health added 2026-04-29 (alerts/ = 14 files). All other pipelines live. |
| 🔔 Notifications | Scripts correct | All C-series bypasses resolved. BUG-N02 counter entity correct. |
| 🧭 Presence | Alert pipeline live | Unknown AP alert + occupancy anomaly implemented. Trust chain intact. |
| 💡 Lighting | Stable | All L01–L10 fixed and verified 2026-04-29. BUG-L09 closed: lighting_entertainment.yaml + lighting_energy_saving.yaml both populated. M1/M2/M3 implemented. SOC-based energy saving trigger remains future work (power session). |
| 🌐 Network | All bugs fixed | BUG-NET01 fixed 2026-04-28: unifi_cpu_5m_max availability → has_value(unifi_gateway_cpu_utilization). BUG-NET02 fixed 2026-04-28: unifi_memory_5m_max availability was self-referencing → has_value(unifi_gateway_memory_utilization). BUG-NET03 fixed 2026-04-28: packet loss removed from wan_health_score (ping_sum_5min is latency sum not pass count); score now uses latency + jitter only. BUG-NET04 fixed 2026-04-21. All verified live in network_helpers.yaml. |
| 🏗️ Context | All fixed | BUG-CTX01 fixed 2026-04-30: context_presence.yaml → presence/presence_trust.yaml. BUG-CTX02 fixed 2026-04-28: context_schedules.yaml deleted, bedtime_mode in lighting_helpers. BUG-CTX03 fixed 2026-04-30: home_context now derives from security_nobody_home + night_confirmed, no longer imports sensor.security_mode from security/. |
| 🔧 Infra | All fixed 2026-04-28 | BUG-CORE01 fixed (ha_events_per_second removed). BUG-INF01 fixed (printer_cartridge_low dangling }}). BUG-BKP01 fixed (github.yaml routed through notify_system_event). BUG-WEA01 confirmed already fixed. |

---

## 🚀 Verified Priority Work Queue

### Group A — Trust Model Chain (Fixes security + lighting + door alerts)
```
✅ A1. Restore input_datetime.low_trust_start/end — SUPERSEDED / NOT NEEDED.
✅ A2. boundary_permissive_window fixed — reads input_boolean.maid_on_site + weekday()==0
✅ A3. Replace input_boolean.low_trust_present → binary_sensor.low_trust_present
✅ A4. Same fix in lighting_departure.yaml
```
**Group A complete. Trust chain fully functional.**

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
| `docs/domains/CONTEXT_CONTRACT.md` | Context — night mode, trust model, 3 arch issues | ✅ Authoritative (2026-04-16) |
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

---

*2026-04-30 session: Doc sweep — verified live against actual files: BUG-NET01/02/03 confirmed fixed in network_helpers.yaml (availability references corrected, packet loss removed from health score formula). alerts_presence.yaml 158 lines fully implemented (not 17-line stub — contract was stale). borehole gate already marked done. Infra domain table updated (all 4 bugs fixed). Network sensor reliability (unifi_cpu_5m_max, unifi_memory_5m_max, wan_health_score) — formula fixes confirmed in code; live value reliability (whether UniFi integration feeds valid data) is a runtime check, not code — verify in Developer Tools → States.*
*2026-04-30 session: average_night_consumption sampling bug fixed — energy_state.yaml: removed sampling_size: 20 (at 5s solarman poll rate it gave ~100s effective window, silently overriding max_age: 24h because HA statistics platform uses whichever limit is hit first). Now uses max_age: hours: 24 only → true 24h rolling mean. ALERTS_CONTRACT updated: alerts_presence.yaml line count corrected 17→~158 (was fully implemented 2026-04-16, line count never updated). Infra domain table updated: all 4 bugs marked fixed (were already confirmed fixed in session log but table still said "4 bugs found"). Water borehole gate confirmed ✅ done (already in Load Control section line 236, was just stale in old TODO text). BUG-CTX01/CTX03 fixed (see below).*
*2026-04-30 session: BUG-CTX01 fixed — context_presence.yaml content moved to packages/presence/presence_trust.yaml; context_presence.yaml deleted. Entity IDs unchanged. BUG-CTX03 fixed — sensor.home_context in context_global.yaml no longer imports sensor.security_mode or sensor.security_trust_mode from security/; now derives from binary_sensor.security_nobody_home (self-contained in context_global) + binary_sensor.night_confirmed; trust attribute simplified to low_trust/normal via binary_sensor.low_trust_present. ha core check passed. context/ package: 4→2 files. presence/ package: 5→6 files. All three context bugs confirmed closed.*
*2026-04-30 session: Telegram backslash escape fix — all 6 notify scripts had 18-character MarkdownV2 escape chains applied (BUG-N09 fix 2026-04-13), but the Telegram bot integration config has `options.parse_mode: markdown` (old Markdown, not MarkdownV2). Old Markdown does not recognise backslash escapes for `.`, `(`, `)`, `+`, `|` etc., so they appeared literally in all messages. Reduced escape chain to 4 characters in all 6 scripts: `\\`, `\*`, `\_`, `` \` `` (only characters special in old Markdown). NOTIFICATIONS_CONTRACT.md BUG-N09 updated with revised history. Rule: do NOT expand escape chains beyond these 4 unless the Telegram integration is switched to parse_mode: markdownv2.*
*2026-04-30 session: WAN outage 04:30–05:00 caused cascade of Telegram ConnectTimeout errors (geyser AM/PM at 04:30, bar via presence at 04:45, git push at 05:00). Root issue: `continue_on_error: true` was on the Telegram send INSIDE notify scripts but NOT on the calling `script.notify_lighting_event` action — ConnectTimeout propagates as "Unexpected error" which bypasses the inner guard. Fixed: 23 call sites across 7 lighting files (lighting_bar_presence +6, lighting_evening +3, lighting_morning +4, lighting_arrival_night +3, lighting_boundary +2, lighting_garage +2, lighting_office_presence +3). gitupdate.sh hardened: now skips commit+push if `git diff --cached --quiet` (nothing staged) → exits 0 cleanly instead of propagating git's exit 128 when there's nothing to push. github.yaml backup failure message: added `| default('no stderr')` guard on backup_result['stderr'] (was None, causing Jinja2 error in the failure notification). Prepaid Repairs error (notify.std_warning in automations.yaml legacy automation) — confirmed already documented in session notes; fix via HA UI only (DO NOT TOUCH automations.yaml rule).*
*2026-04-15 session (Group A audit): A1 confirmed superseded — low_trust_start/end never needed; complete trust chain runs through maid_start/end + gardener_start/end datetimes → schedule automations → maid_on_site/gardener_on_site booleans → binary_sensor.staff_on_site → binary_sensor.low_trust_present. A2 confirmed done — boundary_permissive_window reads maid_on_site + weekday()==0 + override, no longer always false. Group A fully complete.*
*Last updated: 2026-04-30*
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
