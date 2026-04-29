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

**BUG-CORE01 [LOW] — `sensor.ha_events_per_second` is not events/second**
`state: "{{ (states.sensor | selectattr('last_changed','defined') | list | count) }}"`
This counts the total number of sensor entities with a `last_changed` attribute —
effectively the total sensor entity count (~several hundred). It never changes.
It does NOT measure events per second. Rename or fix with a time-delta approach.
Same issue with `sensor.ha_update_rate` which uses `{{ states | list | count }}`.

**BUG-CORE02 [LOW] — `sensor.ha_stability_score` has hardcoded error count**
`{% set errors = 1 %}  # you can later count log errors`
The `errors` variable is hardcoded to 1 and is not used in any condition.
The sensor still works (CPU/load thresholds are correct) but the comment is
misleading and the variable is dead code.

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

**BUG-BKP01 [MEDIUM] — Direct Telegram call bypasses notification pipeline**
`backup/github.yaml` calls `notify.send_message` on `notify.telegram_bot_5527` directly
instead of routing through `script.notify_system_event`. This bypasses quiet hours and
the centralised notification system.
**Fix:** Replace both notify calls with `script.notify_system_event` — use severity `info`
for success and `warning` for failure. Note: backup runs at 05:00 which IS quiet hours,
so the current direct-Telegram approach means backup failures ARE delivered even if
quiet hours are active. Decide if that is intentional before changing.

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

**BUG-INF01 [HIGH] — `binary_sensor.printer_cartridge_low` has dangling `}}`**

```yaml
state: >
  {{ expand('group.printer_cartridges_sensors')
    | map(attribute='state')
    | map('float', 100)
    | select('<', 25)
    | list | count > 0 }}
  }}        # ← extra }} on separate line
```

The YAML folded scalar joins the lines with a space, producing:
`{{ ... | count > 0 }} }}`

In Jinja2, the first `}}` closes the expression; the remaining ` }}` is literal
text appended to the result. The binary sensor state becomes e.g. `True }}` which
HA cannot interpret as on/off → sensor is always unavailable or incorrect.

**Fix:** Remove the trailing `}}` line.

```yaml
state: >
  {{ expand('group.printer_cartridges_sensors')
    | map(attribute='state')
    | map('float', 100)
    | select('<', 25)
    | list | count > 0 }}
```

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

Stale alert automation (`weather_api_stale_alert`):
- Triggers when `weather_last_update_age > 14400s` (4 hours) for 2 minutes
- Sends no notification (no action defined — fires to condition only) ← IMP-WEATHER01
- The automation exists but has no action block — it is effectively a no-op

### Weather Integrations Installed (Multiple — see Part 7)
- `weather.openweathermap` — primary (canonical for weather_core.yaml)
- `weather.metnow_cast` / Met Nowcast — radar-based short-term (1–2h)
- `weather.met_next_6_hours` — Met.no 6-hour forecast
- `openweathermaphistory` — historical weather data (separate sensor domain)

### Known Issues

**BUG-WEA01 [LOW] — `weather_api_stale_alert` automation has no action**
The automation triggers correctly on stale data but has no `actions:` block —
it fires and does nothing. Either add a `script.notify_system_event` call or delete
the automation.

**BUG-WEA02 [LOW] — Inline comment mismatch in threshold**
`above: 14400 # 1hr` — 14400 seconds is 4 hours, not 1 hour. Update comment.

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
| `localtuya` | 5.2.3 | LocalTuya devices (water tank depth) | UI-only (no YAML config) |
| `load_shedding` | 1.5.2 | Eskom load shedding schedule | `packages/power/load_shedding.yaml` |
| `pyscript` | 1.7.0 | Python scripting in HA | `pyscript/sync_power_groups.py` |
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

**`localtuya`** — The water tank depth sensor (`sensor.water_tank_depth_validated`)
is a Tuya device managed by LocalTuya. All device config is UI-only (stored in
`.storage/localtuya`). A false reconnect event can trigger a spurious tank-full
abort — see `input_boolean.water_refill_aborted_due_to_safety` in WATER_CONTRACT.md.

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

*Last updated: 2026-04-16*
*Sources: packages/core/, packages/backup/, packages/office/, packages/weather/,*
*packages/sensors/, packages/integrations/, custom_components/*
