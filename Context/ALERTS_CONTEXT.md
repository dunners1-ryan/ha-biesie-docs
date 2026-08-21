# HABiesie — Alerts & Notifications Domain Context
> **Living document.** Update after every change to alerts or notifications packages.  
> Paste alongside `PROJECT_STATE.md` when working on alerts/notifications.
>
> **⚠️ Superseded by `docs/domains/ALERTS_CONTRACT.md` and
> `docs/domains/NOTIFICATIONS_CONTRACT.md` — use the contracts for real work.** This
> file is a quick-reference summary, dated 2026-04-13, essentially untouched since while
> both domains absorbed months of fixes (BUG-A01 through BUG-A19, BUG-N01 through
> BUG-N18). Corrected 2026-08-21: Package Files listed 2 nonexistent files and every
> notifications filename was wrong; Known Issues #3/#4 (notify bypasses, quiet hours)
> both confirmed fixed. Items #1/#2 and "Next Steps" not individually re-verified this
> pass.

---

## 🎯 Intended Design

Severity-driven alert pipeline with centralised routing, quiet hours, and per-domain context:

```
Domain Condition → Severity Sensor → Alert Entity → Context Sensor → Global Summary → Dashboard + Notify
```

**Core principle:** All logic lives in severity sensors. Notifications are dumb delivery only.

---

## 📁 Package Files

**⚠️ Corrected 2026-08-21 — this list named 2 files that don't exist
(`alerts_core.yaml`, `alerts_device.yaml`, already flagged stale in ALERTS_CONTRACT.md)
and every `packages/notifications/` filename was wrong (wrong naming scheme entirely).
Replaced with the live inventory — 16 files in `alerts/`, 13 in `notifications/`; see
ALERTS_CONTRACT.md Section 2 and NOTIFICATIONS_CONTRACT.md Section "Files Audited" for
full per-file descriptions.**

```
packages/alerts/  (16 files)
  alerts_helper.yaml, alerts_summary.yaml, alerts_doors.yaml, alerts_network.yaml,
  alerts_power.yaml, alerts_temperature.yaml, alerts_device_power.yaml, alerts_media.yaml,
  alerts_system_health.yaml, alerts_presence.yaml, alerts_water.yaml, alerts_security.yaml,
  alerts_garden.yaml, alerts_batteries.yaml, alerts_device_batteries.yaml,
  alerts_camera_health.yaml

packages/notifications/  (13 files)
  notifications_control.yaml, notifications_quiet_hours.yaml, notifications_helpers.yaml,
  notify_power_event.yaml, notify_water_events.yaml, notify_security_events.yaml,
  notify_presence_events.yaml, notify_system_event.yaml, notify_light_events.yaml,
  admin_notifications.yaml, presence_notifications.yaml, water_notifications.yaml,
  power_notifications.yaml (empty stub, unused)
```

---

## 🔴 Critical Entities (DO NOT RENAME)

### Global Alert Pipeline
```
sensor.global_alert_context              # none/info/warning/critical — highest active severity
sensor.alert_device_entities             # SINGLE SOURCE OF TRUTH — attribute: devices list
sensor.active_alert_entities             # comma-separated list of ON alert entity_ids
sensor.total_critical_alert_devices      # count
sensor.total_warning_alert_devices       # count
sensor.total_info_alert_devices          # count
sensor.total_active_alerts               # total ON alerts
sensor.total_acknowledged_alerts         # total idle/off alerts
sensor.total_unacknowledged_alerts       # active and not acknowledged
sensor.total_alert_count                 # all alerts combined
```

### Domain Alert Context Sensors
```
sensor.doors_open_alert_severity         # none/info/warning/critical
sensor.network_alert_context             # none/info/warning/critical  
sensor.power_alert_context               # none/info/warning/critical
sensor.temperature_alert_context         # none/info/warning/critical
sensor.water_alert_context               # none/info/warning/critical
sensor.security_alert_context            # none/info/warning/critical
```

### Notification Scripts (NEVER BYPASS THESE)
```
script.notify_power_event
script.notify_water_event
script.notify_security_event
script.notify_presence_event
script.notify_system_event
script.notify_lighting_event
```

### Notification State
```
input_boolean.notifications_enabled
input_boolean.notifications_quiet_hours
input_datetime.last_power_notification
input_datetime.last_water_notification
input_datetime.last_presence_notification
```

### Control Toggles
```
input_boolean.inverter_grid_offline_notify
input_boolean.inverter_battery_soc_low_notify
input_boolean.power_excess_load_notify
input_boolean.door_alerts_notify
input_boolean.network_device_down_notify
input_boolean.wan_down_notify
input_boolean.device_restart_notify
input_boolean.lan_device_temp_notify
input_boolean.wan_device_temp_notify
input_boolean.device_temp_notify
input_boolean.storage_temp_notify
input_boolean.sensor_health_notify
input_boolean.critical_sensor_health_notify
```

### Alert Thresholds
```
input_number.inverter_battery_soc_warning_trigger
input_number.grid_off_delay_trigger
input_number.wan_temp_high_trigger
input_number.lan_temp_high_trigger
input_number.device_temp_high_trigger
input_number.storage_temp_high_trigger
input_number.perimeter_open_escalation_minutes
input_number.house_entry_escalation_minutes
input_number.door_warning_escalation_minutes
input_number.alert_repeat_trigger
input_number.network_uptime_trigger
```

---

## 📐 Alert Pipeline Architecture

```
Domain binary/template sensor
        ↓
  alert.<domain>_<device>      ← HA alert entity (manages on/idle/off)
        ↓
  sensor.alert_device_entities  ← aggregates all alerts with metadata
  {
    devices: [
      { device: "Main Gate", domain: "Security", severity: "warning",
        alert_entity: "alert.door_main_gate", duration: 15, value: "Open" },
      ...
    ]
  }
        ↓
  sensor.*_alert_context        ← per-domain highest severity
        ↓
  sensor.global_alert_context   ← overall highest severity
        ↓
  Dashboard + notify script
```

---

## 🚦 Severity Levels

| Level | Display | Delivery | Quiet Hours |
|---|---|---|---|
| `none` | Green / no animation | No notification | N/A |
| `info` | Blue | Mobile only | Suppressed |
| `warning` | Orange | Mobile | Delivered |
| `critical` | Red + pulse animation | Mobile + Telegram | Always delivered |

---

## 🔔 Notification Groups

```
notify.STD_Information   # info level group
notify.STD_Warning       # warning level group
notify.STD_Critical      # critical level group
```

All notifications routed through domain scripts (`script.notify_*_event`).

---

## ✅ What Works Well

- Global alert context sensor
- Severity-driven alert display
- Auto-acknowledge via HA alert integration
- Door alert escalation (open → warning → critical based on time)
- Alert pipeline (alert entities → context sensors → global)
- Quiet hours (info suppressed, warning/critical always through)
- Template hardening (all numeric conversions have defaults)
- Notification formatting (human-readable, not raw dict)
- YAML schema corrected (modern HA syntax throughout)

---

## ⚠️ Known Issues

### 1. Alert Flapping / Auto-Acknowledge Bug
- **Cause:** Rapid state changes cause alert to fire ON then immediately OFF
- **Symptom:** Alert briefly appears then is "already acknowledged"
- **Fix applied:** Delay before alert turns ON to prevent false acknowledgement
- **Status:** Partially fixed — some edge cases remain

### 2. Summary vs Detail Mismatch
- **Cause:** Duplicate logic in summary and detail sensors can produce different counts
- **Fix:** Both now derive from `sensor.alert_device_entities` (same source)
- **Status:** Fixed for most domains

### 3. ✅ DONE — Not All Domains Unified
**Status corrected 2026-08-21:** all direct-notify bypasses this item was tracking
(temperature BUG-A03, device power BUG-A04, presence BUG-N05, plus the whole
`notify.STD_Alerts` group itself, BUG-A10/BUG-N16) are fixed and re-verified live this
session — see ALERTS_CONTRACT.md Section 6 and NOTIFICATIONS_CONTRACT.md Section
"Priority 1". No longer "in progress".
- ~~Some older alerts still use direct `notify.*` calls instead of central scripts~~
- ~~**Fix needed:** Audit all `notify.` calls, migrate to `script.notify_*_event`~~

### 4. ✅ DONE — Quiet Hours Not Consistently Applied
**Status corrected 2026-08-21:** same underlying cause as #3 (direct notify bypasses),
same fix, same confirmation. Every domain now routes through a central `script.notify_*_event`.
- ~~Applied in central scripts but some legacy automations bypass them~~
- ~~**Fix needed:** Remove direct notify calls, all through scripts~~

---

## 🎯 Next Steps (Agreed Priority)

1. **Audit all notify calls** — ensure nothing bypasses central scripts
2. **Subsystem tagging** — add `subsystem` field to all alert metadata
3. **Cooldown handling** — prevent same alert from re-notifying within cooldown window
4. **Confidence-based suppression** — suppress low-confidence alerts
5. **Alert deduplication** — don't create multiple alerts for same underlying cause

---

## 🔗 Dependencies

All domains feed into this system:
- **Security:** `sensor.doors_open_alert_severity`, `input_boolean.security_alert_active`
- **Power:** `sensor.power_alert_context`, `sensor.house_power_health`
- **Water:** `sensor.water_alert_context`, `sensor.water_state`
- **Network:** `sensor.network_alert_context`, `sensor.wan_noc_status`
- **Temperature:** `sensor.temperature_alert_context`, device temp sensors
- **Presence:** `binary_sensor.unknown_unifi_ap_detected`
- **Context:** `binary_sensor.night_confirmed`, `input_boolean.bedtime_mode`

---

*Last updated: <!-- DATE -->*  
*Updated by: <!-- CHANGE SUMMARY -->*
