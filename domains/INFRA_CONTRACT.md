##########################################################
# HABiesie — INFRASTRUCTURE CONTRACT
# Covers: core/, backup/, admin/, office/, weather/,
#         sensors/, integrations/, custom_components/
# Generated: 2026-04-16
##########################################################

This contract covers the utility/support packages that are too small to warrant
their own domain contracts but collectively form the system's observability and
integration plumbing. It also documents all custom integrations (HACS).

---

## Part 1: core/ — HA System Health

### Files
- `packages/core/core_helpers.yaml` — system_startup guard + HA health sensors
- `packages/core/ha_monitoring.yaml` — recorder health, DB growth, EPS (broken), noisy entities
- `packages/core/sensor_smoothing.yaml` — RSSI smoothing for Sonoff devices

### Key Entities

```
input_boolean.system_startup          — ON for 2 min after HA start (startup guard)
sensor.ha_system_health               — healthy / warning / critical (CPU+mem+load+temp)
sensor.ha_uptime_hours                — hours since last boot
sensor.ha_stability_score             — stable / degraded / critical (CPU+load)
sensor.recorder_health                — healthy / warning / critical (DB growth rate)
sensor.db_growth_mb_10min             — MB added to DB in last 10 min
sensor.ha_events_per_second           — ⚠️ BUG-CORE01: misleading (see bugs)
sensor.top_noisy_entities             — attribute list of recently-changed sensors
binary_sensor.device_down_ping_helper — always-off helper to prevent empty groups
```

RSSI Smoothed (sensor_smoothing.yaml — 30s trigger-based):
```
sensor.water_pressure_pump_rssi_smoothed
sensor.borehole_pump_rssi_smoothed
sensor.laundry_switch_rssi_smoothed
sensor.patio_switch_rssi_smoothed
```

### `input_boolean.system_startup` — Critical Usage
Used in `presence_confidence.yaml` as a startup guard. If this is ON, presence
confidence gates are suppressed (prevents spurious arrivals/departures on restart).
Do NOT remove or rename this entity.

### Known Bugs

**~~BUG-CORE01~~ [LOW] ✅ Doc-drift correction 2026-07-10 — `sensor.ha_events_per_second` is not events/second**
The described sensors (`sensor.ha_events_per_second`, `sensor.ha_update_rate` — full-registry
scans on every template render) no longer exist anywhere in the codebase; this entry was
describing dead code that had already been replaced. The current equivalent is
`sensor.ha_event_rate_1m` (`packages/sensors/filter.yaml`), a `statistics` platform sensor
counting `sensor.update_ticker` changes over a rolling 1min window — a properly lightweight
implementation, not a whole-state-machine scan. Per user instruction 2026-07-10, added an
inline comment on that sensor explaining why it must stay this cheap (Raspberry Pi
performance) rather than reintroducing the expensive pattern this bug originally described.

**~~BUG-CORE02~~ [LOW] ✅ FIXED 2026-07-10 — `sensor.ha_stability_score` has hardcoded error count**
Confirmed live and real: `{% set errors = 1 %}  # you can later count log errors` was still
present in `packages/core/core_helpers.yaml`. The `errors` variable was hardcoded to 1 and
never used in any condition — dead code with a misleading comment.
**Fix applied:** removed the dead `errors` variable and comment entirely. Separately, while
in this sensor: `ha_stability_score` never factored in CPU temperature at all (only
`ha_system_health` did), so an overheating Pi with normal CPU/load would still show
"stable" — added `temp > 85` (critical) / `temp > 70` (degraded) thresholds, matching
`ha_system_health`'s existing critical threshold. This sensor drives the color of the "Home
Assistant Health" card on the Home dashboard view, which was also extended (same session) to
display CPU temperature and uptime alongside CPU/load/RAM — see PROJECT_STATE.md session log
2026-07-10.

---

## Part 2: backup/ — GitHub Config Backup

### Files
- `packages/backup/github.yaml` — backup script, status sensor, 5AM automation

### Key Entities

```
input_text.github_backup_reason       — commit message for manual backups
input_boolean.github_backup_running   — ON while backup script runs
input_boolean.github_show_status_badge — dashboard display toggle
sensor.last_github_backup_result      — timestamp of last triggered run
script.run_github_backup              — executes gitupdate.sh with reason param
```

### Backup Automation
- `github_backup_push_5am` — fires at 05:00 daily, calls `script.run_github_backup`
- On success: Telegram message "GitHub Backup Success"
- On failure: Telegram message with stderr

### Known Issues

~~**BUG-BKP01 [MEDIUM] — Direct Telegram call bypasses notification pipeline**~~
**✅ Fixed 2026-06-19.** `backup/github.yaml` now calls `script.notify_system_event` for
both success (severity: info) and failure (severity: warning) branches. The direct
`notify.send_message` calls were replaced. Backup still runs at 05:00 (quiet hours) —
`notify_system_event` will respect quiet hours for the success notification but failure
is severity `warning` which bypasses quiet hours, preserving the original intent.

---

## Part 3: office/ — Printer Monitoring

### Files
- `packages/office/office_helpers.yaml` — printer status, cartridge level sensors

### Key Entities

```
group.printer_cartridges_sensors      — 4 Canon MB2100 cartridge sensors
sensor.printer_status                 — raw state from canon_mb2100_series
sensor.printer_cartridge_state        — min cartridge % with per-colour attributes
binary_sensor.printer_cartridge_low   — ON if any cartridge < 25%   ← BUG-INF01
```

Cartridge attributes on `sensor.printer_cartridge_state`:
- `black`, `cyan`, `magenta`, `yellow` — individual %
- `low_cartridges` — comma-joined list of colour names below 25%

### Known Bugs

~~**BUG-INF01 [HIGH] — `binary_sensor.printer_cartridge_low` has dangling `}}`**~~
**✅ Fixed 2026-06-19.** Trailing `}}` line removed. `binary_sensor.printer_cartridge_low` now
evaluates correctly — state is a clean boolean expression with no appended literal text.

---

## Part 4: weather/ — Weather API Health

### Files
- `packages/weather/weather_core.yaml` — staleness tracking, stale alert
- `packages/weather/weather_helpers.yaml` — OWM API rate limit helpers

### Key Entities

```
sensor.weather_last_update_age        — seconds since weather.openweathermap last changed
sensor.weather_api_health             — healthy / delayed / stale
input_number.openweathermap_api_critical_threshold — UI: API call limit (default 1900/day)
input_boolean.openweathermap_api_limited — flag for API rate limit dashboard display
```

Stale alert automation (`weather_api_stale_alert` / entity id `automation.weather_data_stale_alert`):
- Triggers when `weather_last_update_age > 14400s` (4 hours) for 2 minutes
- Action: `logbook.log` + `script.notify_system_event` (severity warning) — this DOES notify
  (see BUG-WEA01 correction below)

### Weather Integrations Installed (Multiple — see Part 7)
- `weather.openweathermap` — primary (canonical for weather_core.yaml)
- `weather.metnow_cast` / Met Nowcast — radar-based short-term (1–2h)
- `weather.met_next_6_hours` — Met.no 6-hour forecast
- `openweathermaphistory` — historical weather data (separate sensor domain)

### Known Issues

**BUG-WEA01 [CORRECTED 2026-07-06, FIXED] — was "no action", actually a false-positive staleness bug**
This entry previously claimed `weather_api_stale_alert` had no `actions:` block and was a
no-op. That was already stale by the time of this audit — the automation has always called
`script.notify_system_event` (warning severity). The real bug, found 2026-07-06 by live
investigation of recurring "Weather Data Stale" alerts: `sensor.weather_last_update_age`
(weather_core.yaml) computed age from `weather.openweathermap`'s `last_changed`, which only
updates when the entity's **state string** (the condition, e.g. `sunny`) changes — not on
every poll. During long stable-condition stretches (typical Johannesburg winter clear spells),
the condition can hold for 9+ hours even though OWM is actively refreshing temperature/
humidity/pressure attributes every 15-60 min (`last_updated` stays fresh). This made the age
sensor climb past the 4h threshold and fire a false "stale" alert roughly once or twice a day,
confirmed via 3-day history of `sensor.weather_api_health` cycling
healthy → delayed → stale → (reset on next condition change) → repeat.
**Fix applied:** `weather_core.yaml` now reads `entity.last_updated` instead of
`entity.last_changed`. Live-reloaded via `template.reload`; `sensor.weather_api_health`
confirmed back to `healthy` immediately. Also cleared `input_boolean.openweathermap_api_limited`,
which had been stuck `on` since 2026-07-04 as a downstream side effect of the flapping bug (the
matching recovery automation never fired because its `not_from: [unknown, unavailable]` guard
blocked on the template-reload transition — **✅ fixed 2026-07-06 gap closed 2026-07-10**: removed
the `not_from: [unknown, unavailable]` guard from `weather_api_recovery`'s trigger entirely.
Recovery only needs `to: "healthy"` — where it came from doesn't matter, the action is
idempotent, and the automation's own condition already checks
`input_boolean.openweathermap_api_limited == "on"` before doing anything. This was the one
trigger in `weather_core.yaml` where the usual not_from stability-guard pattern was actively
harmful rather than protective.

**~~BUG-WEA02~~ [LOW] ✅ FIXED 2026-07-10 — Inline comment mismatch in threshold**
`above: 14400 # 1hr` — 14400 seconds is 4 hours, not 1 hour. Comment corrected to `# 4hr`.

---

## Part 5: sensors/ — Global Sensor Utilities

### Files
- `packages/sensors/filter.yaml` — sensor smoothing, HA event rate statistics

### Key Entities (active)

```
sensor.house_outdoor_power_clean    — time SMA 30s of house_outdoor_power_throttled
sensor.ha_event_rate_1m             — count of update_ticker changes per minute
```

Most sensor filters in this file are commented out (previously moved to
`core/sensor_smoothing.yaml` as trigger-based templates). The active entries
are `house_outdoor_power_clean` and the `ha_event_rate_1m` statistics sensor.

---

## Part 6: integrations/ — Integration-Specific Config

### Files
- `packages/integrations/sonoff.yaml` — Sonoff/EWElink config + energy tracking

### Sonoff Integration

```
# Config
sonoff:
  sensors: [staMac, bssid, host]
  devices:
    10020e57fc:
      reporting:
        energy: [3600, 10]  # 1-hour update interval, 10-day history
```

Active entity:
```
sensor.pond_fountain_energy    — kWh, integrated from sonoff_pond_fountain_pump_power
```

All other Sonoff power/cost tracking is commented out (placeholder). The Sonoff
integration also manages:
- `switch.water_pressure_pump` (EWElink device)
- `switch.borehole_pump` (EWElink device)
- `switch.stw_2gang_laundry_switch_*`
- `switch.stw_2gang_patio_switch_*`

---

## Part 7: Custom Components (HACS Integrations)

All custom integrations are in `/config/custom_components/`. They are installed
via HACS and must be updated manually or via HACS UI.

### Integration Registry

| Integration | Version | Purpose | Config Location |
|-------------|---------|---------|-----------------|
| `solarman` | 25.08.16 | Dual inverter (Master/Slave) Modbus polling | `packages/power/power_core.yaml` |
| `solcast_solar` | v4.5.2 | Solar PV forecast (Solcast API) | `solcast_solar/` cache dir + UI |
| `hikvision_next` | 1.1.1 | NVR cameras (16ch DS-7116HGHI-F1) | `packages/security/cameras_core.yaml` |
| `sonoff` | 3.11.1 | EWElink smart switches (pumps, lights) | `packages/integrations/sonoff.yaml` |
| `tuya` | (cloud) | Tuya Cloud devices: geyser heat pump switch, pool pump, pond filter pump, water tank depth sensor | UI-only (no YAML config) |
| `localtuya` | 5.2.3 | Installed, unused — zero entities registered as of 2026-07-13 | UI-only (no YAML config) |
| `load_shedding` | 1.5.2 | Eskom load shedding schedule | `packages/power/load_shedding.yaml` |
| `pyscript` | 1.7.0 | Python scripting in HA | `pyscript/sync_power_groups.py` |
| `alarmo` | 1.10.18 | Software alarm panel (DSC/zone interface target) | ⚠️ UI-only — no package YAML; `alarm_control_panel.dsc_partition_1` stub referenced in `security_automations.yaml` |
| `ids_hyyp` | 1.9.0 | IDS alarm system (hawkMod fork) | ⚠️ UI-only — NO package file |
| `hacs` | 2.0.5 | HACS integration manager | UI-only |
| `watchman` | 0.8.4 | Missing entity/service detection | `alerts/alerts_system_health.yaml` |
| `met_next_6_hours_forecast` | v1.1.2 | Met.no 6-hour forecast | UI-only |
| `metnowcast` | v2.3.6 | Met.no nowcast radar | UI-only |
| `openweathermaphistory` | 2026.04.03 | OWM historical weather data | UI-only |

### Integration Notes

**`solarman`** — Critical integration. Dual-inverter setup requires separate
instances: one for Master (grid/BMS), one for Slave (PV/load/battery). If this
integration fails, the entire power domain goes unavailable.

**`hikvision_next`** — Known issue: sets entity IDs with uppercase serial number
(`DS-7116HGHI-F1...`). Deprecation warning in HA — will break in HA 2027.2.
Bug should be filed upstream at maciej-or/hikvision_next.

**`tuya`** — **Corrected 2026-07-13** (was previously misattributed to `localtuya`
throughout this doc and CLAUDE.md — confirmed via `.storage/core.entity_registry`
and `.storage/core.config_entries`, which show a single `tuya` config entry and
zero `localtuya` entities). All Tuya-branded devices in this house run on the
**cloud** Tuya integration: `switch.geyser_heat_pump_switch`, `switch.pool_pump_switch`,
`switch.pond_filter_pump_switch_1`, and the water tank level sensor cluster
(`sensor.water_tank_level_sensor_*`, feeding the template sensor
`sensor.water_tank_depth_validated` — see WATER_CONTRACT.md). All device config is
UI-only (stored in `.storage/core.config_entries` + `.storage/core.entity_registry`,
not `.storage/localtuya`). Being a cloud integration (not local push/poll), a
Tuya API stall can leave HA's cached entity state showing stale/last-known values
— including "on" — for hours with **no `unavailable` transition**, since the
integration doesn't proactively mark entities unavailable on a missed cloud
round-trip. See POWER_CONTRACT.md BUG-PWR-GEYSER01 for a confirmed incident
(2026-07-11) where this caused the geyser to keep running ~2h15m past its
scheduled turn-off. A false reconnect event on the water tank sensor can also
trigger a spurious tank-full abort — see
`input_boolean.water_refill_aborted_due_to_safety` in WATER_CONTRACT.md.

**`localtuya`** — Installed via HACS but has zero entities configured (confirmed
2026-07-13 via entity registry). Not in active use; every Tuya device in this
house is actually on the cloud `tuya` integration above. Candidate for removal
if no future local-control need is identified — local push would avoid the
cloud-stall failure mode noted above, but re-pairing devices under `localtuya`
is manual per-device work, not done as part of this fix.

**`alarmo`** — Software alarm panel (by nielsfaber). Installed and updated to 1.10.18 (2026-05-17). No active automations wired — referenced as comment stubs only in `security_automations.yaml` (`alarm_control_panel.dsc_partition_1` hook target for DSC physical alarm). Config is UI-only. No package YAML exists. Future use: DSC/IDS integration bridge when S6+ alarm wiring is done.

**`ids_hyyp`** — IDS alarm system integration. **No package file exists.**
Alarm states, zones, and arming automations are presumably in the legacy
`automations.yaml` (3836-line UI file). This is the only hardware integration
with no package-based config. See IMP-IDS01.

**`watchman`** — Scans HA config for missing entity/service references.
Feeds `sensor.watchman_missing_entities` which powers `alerts_system_health.yaml`.
This is a useful safety net — keep it installed and do not suppress the alert.

### Known Bugs / Improvements

**IMP-IDS01 [MEDIUM] — IDS Hyyp has no package file**
The IDS alarm system has an integration installed but zero package-based config.
Any alarm automations are buried in `automations.yaml`. Recommend creating
`packages/security/security_alarm.yaml` to document the alarm's entity interface
(armed states, zone sensors, event triggers) even if automations remain in the UI file.

**IMP-IDS02 [LOW] — Multiple weather integrations with unclear canonical source**
Three Met.no variants + OpenWeatherMap + OWM History are installed. Only
`weather.openweathermap` is referenced in package YAML. Document which integration
is canonical for which use case (forecast vs nowcast vs historical).

---

## Part 8: admin/ — Package Directory (Empty)

The `packages/admin/` directory exists but contains no YAML files. It was likely
created for future admin/system automation work. Either populate it or remove it.
The `packages/notifications/admin_notifications.yaml` handles the admin-facing
notifications (unknown AP, etc.) — so that work is done.

---

## Section: Cross-Cutting Issues

### Backup notification routing
`backup/github.yaml` calls Telegram directly. All other notifications go through
`script.notify_system_event`. See BUG-BKP01.

### `input_boolean.system_startup` is a critical cross-domain entity
It gates presence confidence. If the startup automation fails (e.g. HA crashes
during the 2-minute delay), it could remain ON permanently, suppressing presence.
Dashboard exposure recommended for operators.

---

*Last updated: 2026-06-19 — BUG-BKP01 closed (github.yaml uses notify_system_event); BUG-INF01 closed (printer_cartridge_low trailing }} removed); verified in code*
*Last updated: 2026-04-16*
*Sources: packages/core/, packages/backup/, packages/office/, packages/weather/,*
*packages/sensors/, packages/integrations/, custom_components/*
