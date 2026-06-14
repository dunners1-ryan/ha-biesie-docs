# HABiesie — automations.yaml Full Audit
> Generated: 2026-04-29 (Claude Code session)  
> Source: `/config/automations.yaml` — 2,915 lines post-migration (was 3,836)  
> Scope: All active (non-commented) automations. Commented blocks listed separately.

---

## Methodology

Each active automation was read in full. The following flags are applied:
- `⚠️ DEAD ENTITY` — references `sensor.inverter_power` (dead, use `sensor.inverter_load_power`)
- `⚠️ DIRECT NOTIFY` — calls `notify.STD_*` directly instead of `script.notify_*_event`
- `⚠️ WHATSAPP` — calls `notify.whatsapp` (dropped 2026-04-20; remove on migration)
- `⚠️ DEVICE_ID` — uses device_id not entity_id; HA UI-managed; risky to migrate
- `🚫 GEYSER` — geyser automations; DO NOT TOUCH — geyser system not started
- `⏳ POWER SESSION` — defer to dedicated power session; do not migrate piecemeal

---

## Active Automations

### CORE / NETWORK

| ID | Alias | Status |
|---|---|---|
| `update_device_uptimes_group` | Update Device Uptimes Group | ✅ DELETED 2026-04-29 — superseded by `alerts_network.yaml` (group auto-managed) |
| `device_restart_info_alert_active` | Device Restart Info Alert Active | ✅ DELETED 2026-04-29 — superseded by `binary_sensor.device_restart_alert_active` + `alert.network_alert` pipeline in `alerts_network.yaml` |

---

### LOAD SHEDDING

> **3 automations migrated 2026-04-29** to `packages/load_shedding/load_shedding_automations.yaml`.
> Commented-out blocks retained in `automations.yaml` for stable observation period.

| ID | Alias | Status |
|---|---|---|
| `announce_load_shedding_stage` | Load Shedding Stage Announce | ✅ MIGRATED 2026-04-29 |
| `load_shedding_warning_15` | Load Shedding Warning 15 Minutes | ✅ MIGRATED 2026-04-29 |
| `load_shedding_warning_2hr` | Load Shedding Warning 2 Hours | ✅ MIGRATED 2026-04-29 |

**Remaining (NOT migrated — POWER SESSION):**

| ID | Alias | Lines | Purpose | Complexity | Migration Target |
|---|---|---|---|---|---|
| `1741007573197` | Load Shedding Inverter Scene Switcher | 511–726 | Changes inverter scenes (no_load_shedding / stage_2 / stage_4 / stage_6_priority_charge) based on load shedding stage change, 15-min debounce | complex | `packages/load_shedding/load_shedding_automations.yaml` ⏳ POWER SESSION |

**Flags — Scene Switcher:**
- `⚠️ DEAD ENTITY` → `sensor.inverter_power` in 4 notification message templates. Fix: `sensor.inverter_load_power`.
- `⚠️ DIRECT NOTIFY` → `notify.STD_Critical` on all 4 branches. Fix: `script.notify_power_event`.
- `⚠️ WHATSAPP` → 4 `notify.whatsapp` steps, all `enabled: false`. Remove on migration.

---

### POWER — INVERTER SCENE / SOLAR FORECAST CONTROL

> ⏳ **POWER SESSION — defer all.** These 5 automations are the original inverter
> scene-switching engine, predating the power package. They are active and functional
> but tightly coupled to `input_select.inverter_solar_mode_helper`,
> `input_select.inverter_solar_appliance_mode_helper`, and legacy inverter scenes.
> Do NOT migrate piecemeal — coordinate with power session to rationalise the
> scene architecture first.

| ID | Alias | Lines | Purpose | Complexity |
|---|---|---|---|---|
| `1741870692661` | Solar Forecast Today Update Inverter Scenes 8am-1pm | 813–973 | Switches inverter scene (low_solar_forecast / high_solar_forecast) based on Solcast today forecast hourly 8am–1pm | complex |
| `1741870952191` | Solar Forecast Check Update Appliances 2pm-5pm | 974–1292 | Sets inverter_solar_appliance_mode to Low/Medium Solar based on SOC + hourly forecast, 2pm–5pm every 2 min | complex |
| `1741899615829` | Solar Forecast Remaining Update Inverter Scenes 3-4pm | 1293–1445 | Adjusts inverter mode based on remaining solar + SOC 3–4pm (some actions disabled) | complex |
| `1741900108775` | Solar Forecast Remaining Update Inverter Scenes 4-5pm | 1446–1568 | Adjusts inverter mode based on remaining solar + SOC 4–5pm | complex |
| `1742122936369` | Solar Forecast Check Update Appliances 8am-2pm | 1569–1894 | Sets inverter_solar_appliance_mode (Low/Medium/High Solar) based on SOC + hourly forecast, 8am–2pm | complex |

**Flags — all 5:**
- `⚠️ DEAD ENTITY` → `sensor.inverter_power` in notification message templates across all. Fix: `sensor.inverter_load_power`.
- `⚠️ DIRECT NOTIFY` → `notify.STD_Information` calls. Fix: `script.notify_power_event`.

---

### POWER — GEYSER ✅ MIGRATED 2026-06-14 (Session E2)

| ID | Alias | Migration Target | Status |
|---|---|---|---|
| `1742224800650` | Geyser Heat Pump Turn On AM / PM | `packages/power/geyser_automations.yaml` → `automation.geyser_turn_on` | ✅ MIGRATED |
| `1744130174080` | Geyser Heat Pump Turn Off PM | `packages/power/geyser_automations.yaml` → `automation.geyser_turn_off` | ✅ MIGRATED |

**Fixes applied on migration:**
- `device_id: 04313162c9b0bb48347b8002235c725d` → `switch.geyser_heat_pump_switch` (confirmed)
- `sensor.inverter_power` → `sensor.inverter_load_power`
- `notify.STD_Information` → `script.notify_power_event`
- Load shedding check re-enabled: `state_attr | int(0) < 4` (was disabled — string comparison blocked on unavailable)
- Sports night branch added: 21:05 delayed start (Tue/Thu), hard-off at 23:05
- Winter extension added: morning 04:00 (was 04:30), evening off 22:00 (was 21:05)
- Daily reset scheduler added: `automation.geyser_sports_night_scheduler`
- Manual run added: `automation.geyser_manual_run` + `script.geyser_manual_run`

**New automations in packages/power/geyser_automations.yaml:**
- `automation.geyser_turn_on` — replaces 1742224800650
- `automation.geyser_turn_off` — replaces 1744130174080
- `automation.geyser_sports_night_scheduler` — new (Tue/Thu auto-set + daily clear)
- `automation.geyser_manual_run` — new (timed manual run via script)
- `script.geyser_manual_run` — new dashboard entry point

---

### POWER — GRID USAGE

| ID | Alias | Lines | Purpose | Complexity | Migration Target |
|---|---|---|---|---|---|
| `1742385109757` | Grid Usage Warning per day | 2233–2415 | Notifies at low/medium/high daily grid import thresholds (3 branches) as `sensor.grid_energy_import_today` changes | moderate | `packages/power/power_automations.yaml` ⏳ POWER SESSION |

**Flags:**
- `⚠️ DEAD ENTITY` → `sensor.inverter_power` in message of Branch 2 (medium) and Branch 3 (high). Branch 1 already uses `sensor.inverter_load_power` correctly — copy that pattern.
- `⚠️ DIRECT NOTIFY` → `notify.STD_Warning` (uppercase) in all 3 branches (casing normalized 2026-04-30; was lowercase in Branches 2 and 3). Full fix on migration: `script.notify_power_event`.
- No `from:` guard needed — trigger is `numeric_state` (stateless threshold crossing).

---

### GARDEN / MISC

| ID | Alias | Status |
|---|---|---|
| `1742999668407` | Alert Pond Pump Unscheduled Turn on | ✅ MIGRATED 2026-04-29 → `packages/alerts/alerts_garden.yaml` |

Fixes applied on migration:
- `⚠️ DIRECT NOTIFY` → `notify.mobile_app_iphone16promax_ryan` replaced by `alert.garden_alert → STD_Alerts` + `script.notify_system_event` confirmation.
- **Broken action pattern** → `wait_for_trigger` and `service: switch.turn_off` nested inside notify data block removed. Replaced by proper `mobile_app_notification_action` event automation (`garden_alert_ack_turn_off_pond_pump`).
- Full canonical pipeline implemented: `binary_sensor.garden_alert_active → sensor.garden_alert_context → alert.garden_alert → aggregator`.

---

## Commented-Out Automations (Reference Only)

| ID / Alias | Status | Reason |
|---|---|---|
| `announce_load_shedding_stage` | ✅ MIGRATED 2026-04-29 | → `load_shedding_automations.yaml` |
| `load_shedding_warning_15` | ✅ MIGRATED 2026-04-29 | → `load_shedding_automations.yaml` |
| `load_shedding_warning_2hr` | ✅ MIGRATED 2026-04-29 | → `load_shedding_automations.yaml` |
| `inverter_alert_battery_soc_critical` | ✅ MIGRATED 2026-04-21 | → `power_automations.yaml` (ssa_* → ss_* fix applied) |
| Pool pump ×4 (IDs 1742227789739 etc.) | ✅ MIGRATED 2026-04-22 | → `power_automations.yaml` (pool_pump_solar_control) |
| `grid_status_monitoring` | ✅ SUPERSEDED | By `alert.power_alert` in `alerts_power.yaml` |
| `inverter_pwer_monitoring` | ✅ SUPERSEDED | By `alerts_power.yaml` excess load sensor |
| `notify_prepaid_units_low` | ✅ SUPERSEDED | By `prepaid_core.yaml` (prepaid_auto_reconcile) |
| `1742557570638` (prepaid units low v2) | ✅ SUPERSEDED | By `prepaid_core.yaml` |
| `1742999668407` Pond Pump Unscheduled | ✅ MIGRATED 2026-04-29 | → `packages/alerts/alerts_garden.yaml` (full canonical pipeline) |
| `update_device_uptimes_group` | ✅ DELETED 2026-04-29 | Superseded by `alerts_network.yaml` pipeline |
| `device_restart_info_alert_active` | ✅ DELETED 2026-04-29 | Superseded by `alerts_network.yaml` pipeline |

---

## Migration Priority Queue

### Do Now (non-power, low-risk)
*All immediate items resolved 2026-04-29.*

| Automation | Status |
|---|---|
| `update_device_uptimes_group` | ✅ DELETED — superseded |
| `device_restart_info_alert_active` | ✅ DELETED — superseded |
| `1742999668407` Pond Pump alert | ✅ MIGRATED → `alerts_garden.yaml` |

### Power Session (defer — batch together)
| Automation | Target | Notes |
|---|---|---|
| Load Shedding Inverter Scene Switcher | `load_shedding_automations.yaml` | Fix inverter_power → inverter_load_power, remove whatsapp |
| Solar Forecast 8am-1pm | `power_automations.yaml` | Fix dead entities + direct notify |
| Solar Forecast Appliances 2pm-5pm | `power_automations.yaml` | Fix dead entities + direct notify |
| Solar Forecast Remaining 3-4pm | `power_automations.yaml` | Fix dead entities + disabled actions |
| Solar Forecast Remaining 4-5pm | `power_automations.yaml` | Fix dead entities |
| Solar Forecast Appliances 8am-2pm | `power_automations.yaml` | Fix dead entities + direct notify |
| Grid Usage Warning | `power_automations.yaml` | Fix dead entity + mixed-case notify |

### Geyser Session (never before geyser work starts)
| Automation | Target | Notes |
|---|---|---|
| Geyser Turn On AM/PM | `packages/power/power_geyser.yaml` (create) | Confirm device_id→entity_id |
| Geyser Turn Off PM | `packages/power/power_geyser.yaml` (create) | Confirm device_id→entity_id |

---

## Summary Statistics

| Category | Count |
|---|---|
| Active automations remaining | 9 (was 12; −2 deleted, −1 migrated 2026-04-29) |
| Already migrated to packages | 13 |
| Deferred to power session | 7 |
| Deferred to geyser session | 2 |
| Dead entity references (`sensor.inverter_power`) | 8 automations |
| Direct `notify.STD_*` calls | 8 automations |
| `notify.whatsapp` references | 2 automations (both `enabled: false`) |
