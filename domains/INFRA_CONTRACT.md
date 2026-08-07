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
- `packages/core/tuya_health.yaml` — Tuya Cloud state-feedback staleness watchdog +
  auto-reload (added 2026-08-04, BUG-INFRA-TUYA01 — see Part 7 Known Bugs for the full
  incident writeup)

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

Tuya Cloud Health (`tuya_health.yaml` — added 2026-08-04, BUG-INFRA-TUYA01):
```
sensor.tuya_last_activity_age    — seconds since ANY tracked Tuya entity last updated
                                    (geyser/pool/pond switches + water tank depth/liquid
                                    sensors) — no single entity is a safe staleness canary
                                    on its own since an idle switch can legitimately go
                                    hours with no update
sensor.tuya_cloud_health          — healthy (<2h) / delayed (<4h) / stale (>=4h)
script.tuya_reload_and_verify     — reloads the Tuya config entry, waits 45s, re-checks;
                                    shared by the scheduled watchdog and the manual
                                    notification retry button
automation.tuya_cloud_state_feedback_stale        — id: tuya_cloud_stale_alert
automation.tuya_cloud_state_feedback_recovered    — id: tuya_cloud_recovery
automation.tuya_cloud_retry_reload_from_notification — id: tuya_reload_from_notification;
                                    handles the "🔄 Retry Reload" mobile action button
                                    (event TUYA_RETRY_RELOAD)
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

**~~BUG-WEA03~~ [HIGH] ✅ FIXED 2026-08-04 — `sensor.weather_api_health` never actually
equaled `"healthy"` or `"delayed"` in production**
The `state:` template's `healthy`/`delayed` branches had trailing `# < 2 hours` / `# < 4
hours` annotations meant as comments. Because they sat inside a Jinja `>` folded block
scalar being fed to the template engine, YAML never treated them as comments — they were
literal text rendered into the state. The entity's real state was the string
`"healthy        # < 2 hours"` (confirmed live via Supervisor API), never the clean
`"healthy"`. **Correction to BUG-WEA01 above:** its claim that the 2026-07-10 fix (removing
`weather_api_recovery`'s `not_from` guard) fully closed the recovery gap was incomplete —
that automation triggers on `to: "healthy"`, which could never match this dirty string.
The recovery automation has never actually fired in production, independent of the guard
fix. **Fix:** stripped the trailing comment text from both branches. Grepped the rest of
`weather_core.yaml` for the same anti-pattern (`#` text inside a `>`/`|` block scalar feeding
Jinja) — no other live occurrences (the only other match is the fully dead, pre-commented-out
legacy `state:` block, inert). Checked every consumer matching on an exact state:
`security_core.yaml` only checks `== 'stale'` (unaffected). Validated via
`check_config` (valid), reloaded `template` + `automation`, confirmed live state is now
exactly `"healthy"`. `input_boolean.openweathermap_api_limited` was already `off` at fix
time, so the recovery path itself remains functionally unverified live — first real
healthy→limited→healthy cycle will be the actual test. Found as a side-effect of building
`packages/core/tuya_health.yaml`, which had copied this exact pattern (fixed in the copy
before this bug was traced back to the original).

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

**Update 2026-08-04 (BUG-INFRA-TUYA01)** — root cause of the "cloud stall" pinned
down precisely, and the failure now has a proactive watchdog + auto-fix instead of
only being caught after the fact by a geyser command attempt. See Known Bugs below
for the full writeup; short version: the integration's real-time state-feedback
channel (`tuya_sharing`'s background MQTT client) can die from an uncaught
exception on reconnect and never restart itself, freezing every Tuya entity's
state for hours with commands still working fine — and a targeted reload of just
the Tuya config entry (not a full HA restart) reliably fixes it.

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

**BUG-INFRA-TUYA01 [HIGH] ✅ MITIGATED 2026-08-04 — Tuya Cloud MQTT push thread can
die permanently on a cloud-side rate-limit, freezing every Tuya entity's state for
hours with no `unavailable` transition**
**Files:** `packages/core/tuya_health.yaml` (new), `packages/power/geyser_automations.yaml` (E8)
**Reported by:** user, geyser critical "turn-on failed, still off" alerts overnight
2026-08-04 while the Tuya app showed the geyser genuinely on and heating

**What happened.** At 02:25:03 SAST, the Tuya integration's background MQTT client
(`tuya_sharing/mq.py`, the push channel Tuya's cloud uses to tell HA a device changed
state) tried to reconnect and Tuya's cloud API rejected the handshake with
`network error:(-9999999) API_QPS_LIMIT_OR_DEGRADE`. That exception propagated
**uncaught** out of the `paho-mqtt` worker thread, killing it outright — nothing in
`tuya_sharing` or HA's Tuya integration detects a dead thread and restarts it, and
per the existing note above, entities are never marked `unavailable` when this
happens. Every Tuya entity in the house (`switch.geyser_heat_pump_switch`,
`switch.pool_pump_switch`, `switch.pond_filter_pump_switch_1`, the water tank
depth/liquid-level sensors) froze at its last-known value simultaneously — confirmed
via direct recorder DB query (`states` table), all five showing identical
`last_updated` timestamps before and after the outage.

Command delivery (`switch.turn_on`/`off`) is a **separate REST path** and kept
working the whole time — so the geyser's scheduled morning turn-on (04:00 window +
04:15/04:45/05:15/05:45 backstop retries) genuinely reached the device and it was
heating correctly, exactly as the Tuya app showed. But `script.geyser_verified_turn_on`
(BUG-PWR-GEYSER01's fix) only trusts HA's own entity state to confirm success — with
the feedback channel dead, its `wait_for_trigger` on `state: on` could never fire, so
it retried, timed out again, and raised 4 false "🔴 turn-on failed, still off"
critical alerts. This is the mirror image of BUG-PWR-GEYSER01: there the command was
genuinely dropped and the state correctly reflected that; here the command succeeded
and the state feedback is what was broken.

**Confirmed fix, verified from recorder history, not just theory.** A manual reload
of the Tuya config entry at 09:28:16 SAST is visible in the `states` table as all 5
tracked entities going `unavailable` for ~4s then returning with fresh, live-confirmed
values (the `unavailable` blip is the tell for a config-entry reload — the recorder
keeps running throughout and captures it; a full HA restart kills the recorder too,
so that shows as a clean jump with no `unavailable` row — exactly what's seen for the
*separate*, unrelated 14:04 SAST restart that same day, done for an update). Everything
was confirmed healthy from 09:28 SAST onward — the geyser's 11:00 midday turn-on got a
clean confirmed state change with no false alert, unlike the four before it. **A
targeted config-entry reload alone is sufficient; a full HA restart is not required.**

**Fix — `packages/core/tuya_health.yaml` (new package).** `sensor.tuya_last_activity_age`
tracks the MOST RECENT `last_updated` across all 5 Tuya entities (not any single one —
an idle switch can legitimately go hours without an update, e.g. the geyser off
overnight, so no individual entity is a safe staleness canary; the water tank sensors
are the near-continuous signal that keeps this honest). `sensor.tuya_cloud_health`
buckets that age into healthy (<2h) / delayed (<4h) / stale (>=4h), reusing
`weather_api_health`'s thresholds/shape (see BUG-WEA03 above — copying that pattern is
also how BUG-WEA03 itself was found). `automation.tuya_cloud_state_feedback_stale`
fires at 4h+ and calls `script.tuya_reload_and_verify`: reload the Tuya config entry
(resolved dynamically via `config_entry_id('switch.geyser_heat_pump_switch')`, no
hardcoded entry ID) → wait 45s → check if activity age dropped back under 2 min → info
notify on success, or a **warning** notify with a "🔄 Retry Reload" mobile action
button on failure (handled by `automation.tuya_cloud_retry_reload_from_notification`,
event `TUYA_RETRY_RELOAD` — same convention as `CANCEL_GATE_ALERT` in
`alerts_doors.yaml`) so a retry can be triggered from the phone without opening the HA
or Tuya apps. Deliberately does NOT attempt a full HA restart automatically — the
2026-08-04 evidence shows it isn't needed, and a full restart would briefly take down
every other integration and automation in the house for a fix this narrow doesn't
require.

**Also fixed the same day, same root cause class:** `geyser_verified_turn_on`/`_off`
(E8, `geyser_automations.yaml`) now check `sensor.tuya_cloud_health` before declaring a
genuine command failure. If feedback is already known delayed/stale at the point of
failure, the alert downgrades from critical to warning and says the command likely
succeeded and HA just can't confirm it, instead of accusing a dropped command
(BUG-PWR-GEYSER01's failure mode) — see POWER_CONTRACT.md Issue 26.

**E9 update, next morning (2026-08-05):** E8's stale branches only alerted — a real
occurrence showed the gap: a turn-off correctly detected stale feedback and warned at
08:10, but nothing actually reloaded until the scheduled `tuya_cloud_stale_alert`
watchdog reached its own 4h trigger 36 minutes later at 08:46. The warning couldn't
carry a "🔄 Retry Reload" button either, since it routes through
`script.notify_power_event`, which — unlike `script.notify_system_event` — has no
`actions` field. Rather than add button support to `notify_power_event`, the
stale-feedback branches now call `script.tuya_reload_and_verify` directly the moment
they detect it, instead of just alerting and waiting for the separate watchdog. The
shared script's own notification (info on success, warning + retry button on failure)
is now the single message for the event — the branches no longer send their own
`notify_power_event` in the stale case, avoiding a duplicate/conflicting alert.

**E10 update (2026-08-06) — E9's "stale == probably fine" assumption doesn't
generalize; a genuinely expired Tuya auth token was silently swallowed for a full
morning.** User: "Tuya app credentials expired... did auth them last night... but
nothing turned on this morning." Investigated live via Supervisor API history +
logbook replay. Root cause was NOT the same failure mode as the original
2026-08-04 incident: this time it was a real expired OAuth token, not just a dead
MQTT thread with REST commands still working. `sensor.tuya_cloud_health` went stale
twice overnight (~18:42–22:42 and ~02:57–06:57 SAST) — the second window covered
the *entire* scheduled morning heating sequence (04:00 primary + 04:15/04:45/05:15/
05:45 backstop retries, 5 attempts total). Every attempt's `geyser_verified_turn_on`
detected the stale feedback and, per E9, silently called
`script.tuya_reload_and_verify` instead of alerting — but reloading a config entry
with an expired token can't restore function, so every one of those "✅ reload
succeeded" notifications was a false positive (confirmed via `sensor.geyser_heat_pump_power`
staying flat at 0.0W the entire time — the geyser never actually drew any power).
Net effect: **zero critical alerts reached the user across 5 consecutive real
failures**, because E9's stale-feedback branch always assumed the benign case. The
user only discovered it themselves and fixed it with a manual run + a fresh Tuya
QR-code reauth around 08:07 SAST — the Tuya config entry's `modified_at` timestamp
lines up with that moment exactly, not with the reauth the user believed they'd
done the previous night.

**Fix.** `geyser_verified_turn_on`/`_off` (`geyser_automations.yaml`) no longer
trust stale Tuya feedback alone as evidence the command landed. Both scripts now
baseline `sensor.inverter_load_power` (Solarman-sourced, no Tuya dependency) before
issuing the command; when a retry fails to confirm AND feedback is stale, they
cross-check real evidence — the geyser's own power sensor if it's actually
reporting (>300W confirms on / <200W confirms off), else a >700W jump/drop in
house load — before deciding to stay quiet. Only genuine evidence of the state
change suppresses the alert; otherwise it now sends the critical alert (reworded to
suggest the Tuya app may need re-authenticating) **and** still triggers the reload.
Verified the 700W threshold against three real ramp-up curves (cold morning, midday,
post-outage) — the geyser crosses 300W within ~1 min and 700W within 1.5–4.5 min in
every observed case, comfortably inside the ~10-minute worst-case point (after both
5-minute retry waits) where this check actually runs, so no false-negative risk from
compressor ramp-up lag. Deployed live: `check_config` valid, `script` domain
reloaded via Supervisor API, both scripts confirmed present and idle.

**Same-day follow-up, NOT yet resolved — flagged, not fixed:** later the same day
(2026-08-06), Tuya went stale a **third** time (~12:22–12:24 SAST) and the geyser's
midday solar-gated + forced-minimum turn-on attempts (11:00/11:30/12:00/14:00 SAST,
despite ~6.4kW PV) all failed to confirm "on" — only recovered via the separate
`geyser_period_energy_snapshot` midday safety-net backstop at 15:00. Each of the 4
failed attempts completed in **under 1 second** in the logbook, nowhere near the
5-minute-per-retry timing `geyser_verified_turn_on` should take, and no critical
alert was observed for any of them. This means it's **not yet confirmed that the
E10 fix above actually engages correctly** on this pattern — the fast completion is
unexplained from logbook/history data alone (no script-trace access over the REST
API from this session). **Next session: pull the actual script traces (Settings →
Automations → `geyser_verified_turn_on` → traces) for one of these fast-completing
midday runs and confirm whether the retry/evidence logic is being reached at all,**
before treating E10 as fully proven. Also worth reconsidering: 3 Tuya stale cycles
in under 24h (vs. isolated incidents before) suggests the underlying auth/session
problem may not be fully resolved by the 08:07 reauth either.

**Open question, not resolved this session:** user recalled a prior "6 second
reload" mechanism that they believe was removed around the same time as E9 and
suspect is linked to needing to reauth more often lately. No such interval was
found in `tuya_health.yaml`'s git history (the whole watchdog package is only
2 days old, added 2026-08-04) — if it exists, it's likely in the `custom_components/tuya`
integration itself (e.g. a `scan_interval`), not these packages. Flagged for the
user to point at directly in a future session rather than guessed at here.

**E11 update (2026-08-06/07) — the real cause of the midday silent failures was
an uncaught exception, and the fault is house-wide, not geyser-specific.** Same
day as E10, the geyser's midday turn-on attempts (11:00/11:30/12:00/14:00 SAST,
~6.4kW PV available) all failed with zero alert, each completing in under 1
second — far too fast for E10's retry logic to have run. Pulled actual script
traces to find out why: HA's trace data isn't exposed over the REST API used
throughout this session, so this required a throwaway Python venv +
`websocket-client` against `ws://supervisor/core/websocket` (auth via the same
Supervisor bearer token, then `trace/list` / `trace/get` websocket commands).
The traces showed `switch.turn_on` throwing an **uncaught exception** —
`network error:(-9999999) sign invalid` — on the very first action
(`sequence/1`), aborting the entire script instantly
(`script_execution: "error"`) before it ever reached E10's retry/evidence/alert
logic. Confirmed 4 occurrences this way: 3 on `geyser_verified_turn_on` today,
1 on `geyser_verified_turn_off` the evening before (2026-08-05 21:00) — every
one silent.

**Fix:** added `continue_on_error: true` to all 4 `switch.turn_on`/
`switch.turn_off` calls (initial + retry, both scripts,
`geyser_automations.yaml`) — a hard Tuya API error now flows into the same
wait/retry/evidence/alert path as a silent no-op failure instead of crashing
the script before any notification code runs. Deployed live: `check_config`
valid, `script` domain reloaded, both scripts confirmed idle.

**Scope check via user-supplied HA Logs screenshot — this is bigger than the
geyser.** Filtering HA's own error log for "sign invalid" shows the fault has
been occurring since **2026-08-05 04:00:00, 34+ hours**, not just
2026-08-06 (14 occurrences at the automation level, 11 at the script level for
geyser turn-on alone, plus 2 more on geyser turn-off). **It also hits Pool
Pump** — `automation.pool_pump_solar_control` ("Pool Pump — Solar-Aware Daily
Control," `power_automations.yaml`) shows the identical error signature, 14
occurrences since 12:00:01, on a completely different Tuya device
(`switch.pool_pump_switch`). This confirms the fault is a **Tuya Cloud
command-signing problem at the integration level** — not anything specific to
how the geyser scripts are written. `pool_pump_solar_control` has NOT been
given the `continue_on_error` treatment yet; flagged for a follow-up session
rather than fixed opportunistically without reviewing that automation's own
structure first.

**Positive data point — the fault is intermittent, not a hard outage.** The
evening of 2026-08-06 ran with zero errors: on by 16:18 SAST, at-temperature by
19:06 SAST, through Sports Night, off at 22:00 SAST with a clean confirmed
state change and notification, 4.46 kWh delivered. Consistent with the pattern
of repeated stale/reload cycles rather than a single continuous break.

**Still open, not resolved:** root cause of "sign invalid" itself. Two
untested candidates: (1) host clock/NTP drift against Tuya's servers — Tuya
signs requests with a timestamp and rejects signatures outside its tolerance
window; `timedatectl`/`chronyc` were not reachable from this session's shell
to check. (2) The frequency of `tuya_reload_and_verify`'s own auto-reloads
possibly corrupting signing state — ties to the user's recalled (but
unconfirmed) "6 second reload" mechanism above. `continue_on_error` only
restores *visibility* into these failures; it does not fix why Tuya rejects
the signature in the first place. See POWER_CONTRACT.md Issue 26 (E11).

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
