# HABiesie — Alerts & Notifications Domain Context
> **Living document.** Update after every change to alerts or notifications packages.  
> Paste alongside `PROJECT_STATE.md` when working on alerts/notifications.

---

## 🎯 Intended Design

Severity-driven alert pipeline with centralised routing, quiet hours, and per-domain context:

```
Domain Condition → Severity Sensor → Alert Entity → Context Sensor → Global Summary → Dashboard + Notify
```

**Core principle:** All logic lives in severity sensors. Notifications are dumb delivery only.

---

## 📁 Package Files

```
packages/alerts/
  alerts_core.yaml              # alert entity definitions
  alerts_device.yaml            # device-level alert helpers
  alerts_doors.yaml             # door/gate security alerts
  alerts_network.yaml           # WAN/LAN/device alerts
  alerts_power.yaml             # grid/battery/load alerts
  alerts_temperature.yaml       # device temperature alerts
  alerts_water.yaml             # water system alerts
  alerts_security.yaml          # security event alerts

packages/notifications/
  notifications_control.yaml    # master notify script, severity routing
  notifications_quiet_hours.yaml # quiet hours logic
  notifications_presence.yaml   # presence event notifications
  notifications_power.yaml      # power event notifications
  notifications_water.yaml      # water event notifications
  notifications_security.yaml   # security event notifications
  notifications_system.yaml     # HA system notifications
  notifications_lighting.yaml   # lighting event notifications
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

### 3. Not All Domains Unified
- Some older alerts still use direct `notify.*` calls instead of central scripts
- **Fix needed:** Audit all `notify.` calls, migrate to `script.notify_*_event`
- **Status:** In progress

### 4. Quiet Hours Not Consistently Applied
- Applied in central scripts but some legacy automations bypass them
- **Fix needed:** Remove direct notify calls, all through scripts
- **Status:** In progress

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
