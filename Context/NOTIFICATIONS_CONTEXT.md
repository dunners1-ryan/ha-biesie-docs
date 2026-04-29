# HABiesie — Notifications Domain Context
> **Living document.** Update after every change to notification patterns or scripts.  
> Paste alongside `PROJECT_STATE.md` when working on notifications.

---

## 🎯 Notification Architecture

Centralized, severity-driven notification routing with platform-independent script layer:

```
Domain Automation → Notification Script → Severity Routing → Platform Delivery
```

**Core principle:** All notification logic lives in domain-specific scripts. Delivery is dumb routing only.

---

## 📁 Package Files

```
packages/notifications/
  notify_system_event.yaml         # HA system & core notifications
  notify_power_event.yaml          # Solar, battery, grid events
  notify_water_event.yaml          # Tank, pump, refill events
  notify_security_event.yaml       # Camera, alarm, intrusion events
  notify_presence_event.yaml       # Arrival, departure, occupancy events
  notify_lighting_event.yaml       # Scene activation, presence-aware lighting
  notifications_control.yaml       # Master notify script, severity routing
  notifications_quiet_hours.yaml   # Quiet hours logic & time windows
  admin_notifications.yaml         # Admin-only notifications
  presence_notifications.yaml      # Deprecated or supplementary presence notifs
  notifications_helpers.yaml       # Input helpers for control
```

---

## 🔴 Critical Notification Entities

### Notification Scripts (NEVER BYPASS THESE)
```
script.notify_system_event       # HA uptime, system errors, config reload
script.notify_power_event        # Solar, battery, grid, load shedding
script.notify_water_event        # Tank levels, pump status, refills
script.notify_security_event     # Cameras, door access, alarms
script.notify_presence_event     # Arrival, departure, occupancy changes
script.notify_lighting_event     # Lighting scene activations
```

### Notification State & Control
```
input_boolean.notifications_enabled
input_boolean.notifications_quiet_hours
input_datetime.last_power_notification
input_datetime.last_water_notification
input_datetime.last_presence_notification
```

### Notification Platforms (in configuration.yaml)
```
notify.STD_Information           # Group: mobile apps (dev + family)
notify.STD_Warning               # Group: mobile apps (dev + family)
notify.STD_Critical              # Group: mobile apps (dev + family)
notify.STD_Alerts                # Group: mobile apps + Telegram
notify.telegram_bot_5527         # Telegram Chat Bot (dedicated service)
notify.mobile_app_*              # Individual mobile devices
```

---

## 📋 Implementation Patterns (HA 2026.4.2+)

### Correct Notify Action Syntax

**CRITICAL:** All notification calls must use `notify.send_message` with `target: entity_id` structure. The legacy `service:` syntax is deprecated.

**Pattern for direct service calls:**
```yaml
- action: notify.send_message
  continue_on_error: true          # Required: catches startup race conditions
  target:
    entity_id: notify.telegram_bot_5527  # or notify.STD_Alerts, etc.
  data:
    title: "Title text"
    message: "Body text"
```

**Pattern for notify action groups:**
```yaml
- action: notify.STD_Critical
  continue_on_error: true
  data:
    title: "Alert Title"
    message: "Alert body"
```

### Why `continue_on_error: true` is Mandatory

1. **HA startup sequence:** Automations run BEFORE notification services fully initialize
2. **Race condition:** "ServiceNotFound: notify.telegram_bot_5527" errors appear during boot
3. **Graceful handling:** Flag prevents script failures; notifications still queue and send
4. **Universal requirement:** Apply to ALL notification calls (Telegram, mobile, groups)

### Startup vs. Runtime Behavior

| Phase | Service State | Behavior |
|-------|---------------|----------|
| **Startup** | Initializing | Brief "ServiceNotFound" errors logged, `continue_on_error` catches |
| **After ~30s** | Ready | All notifications send successfully |
| **Runtime** | Active | No errors, instant delivery |

**This is normal and not a configuration problem.** Notifications arrive reliably once services initialize.

---

## 📊 Notification Routing

### Severity-Based Routing

Each domain's `notify_*_event.yaml` script follows this pattern:

```yaml
severity:                          # information | warning | critical
  ├─ information
  │   ├─ Quiet hours: ❌ suppressed
  │   └─ Delivery: HA ❌ / Telegram ✅
  ├─ warning
  │   ├─ Quiet hours: ✅ ignore quiet hours
  │   └─ Delivery: HA ✅ / Telegram ✅
  └─ critical
      ├─ Quiet hours: ✅ ignore quiet hours
      └─ Delivery: HA ✅ / Telegram ✅
```

### Example: notify_power_event.yaml

**HA mobile apps:**
```yaml
- choose:
    - conditions: "{{ sev == 'critical' }}"
      sequence:
        - action: notify.STD_Critical
          data: ...
    - conditions: "{{ sev == 'warning' }}"
      sequence:
        - action: notify.STD_Warning
          data: ...
    - conditions: "{{ sev == 'information' }}"
      sequence:
        - action: notify.STD_Information
          data: ...
```

**Telegram (always separate, respects quiet hours):**
```yaml
- action: notify.send_message
  continue_on_error: true
  target:
    entity_id: notify.telegram_bot_5527
  data:
    title: "{{ icon }} [POWER] {{ title }}"
    message: "{{ message }}"
```

---

## 🚫 Deprecated Patterns (DO NOT USE)

```yaml
# ❌ Pre-2024 service syntax
service: notify.telegram_bot_5527
data:
  message: "..."

# ❌ Missing target structure
action: notify.telegram_bot_5527
data:
  message: "..."

# ❌ notify.send_message to specific service (wrong)
- action: notify.send_message
  data:
    target: "notify.telegram_bot_5527"
    message: "..."
```

---

## 🎯 Integration with Domain Automations

**Pattern:** Domain automations do NOT call scripts directly. They publish events/states that other automations observe:

```
[Automation] → [Severity Sensor] → Observable State Change → [Notification Trigger] → [Script Call]
```

**Bad (direct call from domain automation):**
```yaml
- action: script.notify_power_event
  data:
    severity: warning
    title: "Power Alert"
    message: "..."
```

**Good (state-driven notification):**
```yaml
# In power automation:
- action: homeassistant.update_entity
  target:
    entity_id: sensor.power_alert_severity
  data:
    value: "warning"

# In notification trigger (separate automation):
- trigger: state
  entity_id: sensor.power_alert_severity
  to: "warning"
- action: script.notify_power_event
  data:
    severity: warning
    title: "{{ state_attr('sensor.power_alert_severity', 'title') }}"
    message: "{{ state_attr('sensor.power_alert_severity', 'message') }}"
```

---

## 🔧 Control Toggles

### Global Controls
```
input_boolean.notifications_enabled           # Master on/off
input_boolean.notifications_quiet_hours       # Respect quiet hour windows
```

### Quiet Hours Definition
```
input_datetime.quiet_hours_start          # e.g., 22:00
input_datetime.quiet_hours_end            # e.g., 07:00
input_boolean.quiet_hours_weekends_only   # Apply only on weekends
```

### Platform-Specific Toggles
```
input_boolean.telegram_notifications_enabled
input_boolean.mobile_notifications_enabled
```

---

## 📱 Platform Configuration

### Telegram Bot Setup

**Service Name:** `telegram_bot_5527` (derived from config entry ID)

**Configuration in configuration.yaml:**
```yaml
telegram_bot:
  api_key: !secret telegram_bot_api_key
  parse_mode: markdown
  chat_ids:
    -1002439895527:  # Group chat ID
```

**Notification routing:**
```yaml
notify:
  - platform: group
    name: STD_Alerts
    services:
      - service: mobile_app_iphone16promax_ryan
      - service: mobile_app_ap_0223_1001
      - service: telegram_bot_5527  # ⚠️ Legacy syntax in groups (acceptable here)
```

### Mobile App Targets

Configured in `configuration.yaml` notify section. Each device appears as:
```
notify.mobile_app_<device_name>
```

---

## 🧪 Testing Notifications

### Via Developer Tools (HA UI)

1. **Developer Tools → Actions**
2. Call test action:
```yaml
action: script.notify_system_event
data:
  severity: warning
  title: "Test Notification"
  message: "If you received this, the notification system is working!"
```

3. **Check results:**
   - ✅ Mobile apps received notification
   - ✅ Telegram received message
   - ❌ Check logs for any "ServiceNotFound" errors (expected during startup only)

### Via Service Call (Terminal)

```bash
cd /root/config
cat > /tmp/test_notify.yaml << 'EOF'
service: script.notify_system_event
data:
  severity: warning
  title: "CLI Test"
  message: "Testing from shell"
EOF

# Then call via HA API if accessible
```

### Verification Checklist

- [ ] Notifications arrive on mobile apps
- [ ] Telegram receives messages
- [ ] Quiet hours suppress information-level notifications
- [ ] Critical notifications bypass quiet hours
- [ ] `continue_on_error: true` prevents startup failures
- [ ] No persistent "ServiceNotFound" errors after 30s startup

---

## 🔗 Related Documents

- **Alert System:** See `docs/domains/ALERTS_CONTEXT.md` for alert definitions
- **General API:** See `CLAUDE.md` section "Notification Architecture" for overview
- **Configuration:** See `configuration.yaml` notify platform definitions
- **Quiet Hours:** See `packages/notifications/notifications_quiet_hours.yaml`
