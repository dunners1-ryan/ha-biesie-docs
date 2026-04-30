##########################################################
# NOTIFICATIONS CONTRACT
# HABiesie — Notifications Domain
# Generated: 2026-04-13
# Last updated: 2026-04-22 (BUG-N11 fixed — title undefined guard in notify_power_event.yaml; storage duplicates removed)
# Covers: packages/notifications/ (12 files)
##########################################################

## 1. DOMAIN SCOPE

The notifications domain provides centralized, severity-driven notification routing
with platform-independent script layer. It owns:
- Domain-specific notification scripts (notify_*_event.yaml)
- Severity routing and quiet hours enforcement
- Multi-platform delivery (mobile apps + Telegram)
- Master control toggles

### Files Audited

| File | Role |
|------|------|
| `notifications_control.yaml` | Master notify script, severity routing |
| `notifications_quiet_hours.yaml` | Quiet hours logic & time windows |
| `notifications_helpers.yaml` | Input helpers for control |
| `notify_power_event.yaml` | Solar, battery, grid events |
| `notify_water_events.yaml` | Tank, pump, refill events |
| `notify_security_event.yaml` | Camera, alarm, intrusion events |
| `notify_presence_event.yaml` | Arrival, departure, occupancy events |
| `notify_system_event.yaml` | HA system & core notifications |
| `notify_lighting_event.yaml` | Scene activation, presence-aware lighting |
| `admin_notifications.yaml` | Admin-only notifications |
| `presence_notifications.yaml` | Per-person unknown AP alerts (legacy/supplementary) |
| `water_notifications.yaml` | Water-specific notification automations |

---

## 2. NOTIFICATION ARCHITECTURE

### Canonical Pipeline

```
Domain State Change
  → binary_sensor (alert active)
  → *_alert_context sensor (severity + devices list)
  → alert.* entity (repeat management)
  → alert fires → automation triggers
  → script.notify_*_event (severity routing)
  → Delivery: notify.STD_* + telegram_bot_5527
```

### Severity Levels

| Severity | Quiet Hours | HA Mobile | Telegram |
|----------|-------------|-----------|----------|
| information | suppressed | ✅ | ✅ |
| warning | bypass | ✅ | ✅ |
| critical | bypass | ✅ | ✅ |

---

## 3. NOTIFICATION SCRIPTS — CONFORMANCE TABLE

| Feature | power | water | security | presence | system | lighting |
|---------|-------|-------|----------|----------|--------|----------|
| `continue_on_error` on mobile | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `continue_on_error` on Telegram | ✅ | ✅ *(fixed)* | ✅ | ✅ | ✅ | ✅ |
| Severity normalization | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Quiet hours respected | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Telegram mirroring | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Markdown escaping on Telegram | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* |
| Escaped vars default-guarded | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* | ✅ *(fixed)* |
| Logbook entry | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ |
| Escalation on quiet hours | ❌ | ✅ (unique) | ❌ | ❌ | ❌ | ❌ |

### Notes

**All scripts — Escape chain reduced to old Markdown only** *(Revised 2026-04-30)*
The Telegram bot integration is configured globally with `parse_mode: markdown` (old Markdown, not MarkdownV2).
Old Markdown only treats `*`, `_`, and `` ` `` as special. The 18-character MarkdownV2 escape chain applied
2026-04-13 caused characters like `\.`, `\(`, `\+` to appear literally in Telegram messages, since old Markdown
does not recognise backslash as an escape for those characters. Reduced to 4-character chain:
`\\`, `\*`, `\_`, `` \` ``. See BUG-N09 (history below).

**All scripts — `escaped_title` / `escaped_message` default guards** *(Fixed 2026-04-13)*
All 6 domain notification scripts (`notify_power_event.yaml`, `notify_water_events.yaml`,
`notify_security_events.yaml`, `notify_presence_events.yaml`, `notify_system_event.yaml`,
`notify_light_events.yaml`) lacked `| default('')` at the start of the escape chain for
`escaped_title` and `escaped_message`. If any caller omitted the `title` or `message` field,
HA threw a TemplateError warning on every render. Fixed by adding `| default('')` immediately
after each variable reference — `{{ title | default('') | replace(...) }}` pattern now
uniform across all scripts. See BUG-N08 (resolved).

**notify_water_events.yaml — Missing `continue_on_error` on Telegram**
The Telegram notify call (approximately line 146) lacks `continue_on_error: true`. All other
scripts have this flag. During HA startup the Telegram service may not be initialized, causing
the water notification script to fail silently or error if called at boot.

**notify_water_events.yaml — Wrong counter entity reference**
The weekly borehole fault summary references `counter.water_borehole_faults_week`.
The correct entity name is `counter.water_borehole_faults_this_week`. This causes the
weekly summary to report an unknown or 0 fault count regardless of actual faults.

**notify_water_events.yaml — Raw sensor reference**
The script references `sensor.water_tank_level_sensor_liquid_level` (raw Tuya sensor,
not validated). The validated/normalized sensor should be used instead — consistent with
patterns in the water package.

**notify_water_events.yaml — Unique escalation pattern**
This script escalates information-severity notifications to warning when quiet hours
are active. This is intentional and documented behavior, but is inconsistent with all
other scripts. If adopted as a pattern, it should be documented in CODING_STANDARDS.md.

---

## 4. GLOBAL CONTROL HELPERS

### Defined in YAML (`notifications_helpers.yaml`)

```
input_boolean.notifications_enabled       # Master on/off toggle
input_boolean.notifications_quiet_hours   # Quiet hours override flag (set by time automations)
input_boolean.telegram_notifications_enabled
input_boolean.mobile_notifications_enabled
```

### `input_boolean.notifications_enabled` — YAML-only (fully resolved 2026-04-21)

`input_boolean.notifications_enabled` is defined in `notifications_helpers.yaml` (added 2026-04-15, BUG-N03).
The legacy UI-created storage entry was removed from `.storage/input_boolean` on 2026-04-21.
Entity is now fully reproducible from YAML alone — no storage dependency.

---

## 5. QUIET HOURS COVERAGE

### Definition

Quiet hours: 22:00–06:00, managed by time-based automations in
`notifications_quiet_hours.yaml` setting `input_boolean.notifications_quiet_hours`.

### Coverage by Script

| Script | Quiet Hours Check | Behaviour |
|--------|------------------|-----------|
| notify_power_event.yaml | ✅ | information suppressed |
| notify_water_events.yaml | ✅ | information escalated to warning |
| notify_security_event.yaml | ✅ | information suppressed |
| notify_presence_event.yaml | ✅ | information suppressed |
| notify_system_event.yaml | ✅ | information suppressed |
| notify_lighting_event.yaml | ✅ | information suppressed |
| presence_notifications.yaml | ✅ routes via script.notify_presence_event | information suppressed (fixed 2026-04-15) |
| water_notifications.yaml | ❌ BYPASS | always sends |
| alerts_temperature.yaml | ✅ routes via script.notify_system_event | quiet hours respected (fixed 2026-04-15) |
| alerts_device_power.yaml | ✅ alert entity only (route automation removed) | no bypass (verified 2026-04-15) |

Warning and Critical notifications bypass quiet hours in all scripts (correct behavior).

---

## 6. DIRECT NOTIFY BYPASS VIOLATIONS

All the following locations bypass the canonical notification pipeline (script → severity routing
→ quiet hours). They call notify platforms directly, skipping quiet hours, Telegram mirroring,
logbook, and escalation rules.

### Priority 1 — High Volume / Always-On Bypasses

| File | Line(s) | Violation | Impact |
|------|---------|-----------|--------|
| `presence_notifications.yaml` | ~19, ~33, ~47, ~61 | 4 automations call `notify.send_message → notify.STD_Information` directly | No quiet hours, no Telegram, no script |
| `alerts/alerts_temperature.yaml` | ~786,797,878,889,970,981,1066,1077 | 4 routing automations (WAN/LAN/Device/Storage) call `notify.STD_*` directly | No quiet hours, no Telegram, no logbook |
| `alerts/alerts_device_power.yaml` | ~216, ~227 | Route automation calls `notify.STD_Critical/Warning` directly AND `alert.device_power_fault` also fires to `STD_Alerts` | Dual delivery on every fault |

### Priority 2 — Functional But Non-Standard

All Priority 2 files now have `continue_on_error: true` on direct Telegram calls (fixed 2026-04-14).
Root cause of `ServiceNotFound: notify.telegram_bot_5527` errors on restart: `solar_forecast.yaml`
fires at `hours: /3` (hitting midnight after restart) before Telegram service is registered.

| File | Line(s) | Violation | `continue_on_error` | Impact |
|------|---------|-----------|---------------------|--------|
| `water/water_reporting.yaml` | ~50 | Weekly summary calls notify directly | ✅ fixed | No quiet hours suppression |
| `water/water_tank_refill_control.yaml` | ~221, ~343 | Emergency pump notifications call notify directly | ✅ fixed | Acceptable for safety-critical path |
| `backup/github.yaml` | ~63, ~71 | Backup status notifications call notify directly | ✅ fixed | Low priority, admin-only |
| `lighting/lighting_bar_presence.yaml` | ~76, ~112 | Bar presence events call notify directly | ✅ fixed | Low priority, presence-adjacent |
| `power/solar_forecast.yaml` | ~175, ~229, ~276 | Solar mode change sends Telegram directly (not via script) | ✅ fixed | Fires at midnight on restart — was primary source of ServiceNotFound errors |

### Migration Priority

1. **`alerts_temperature.yaml`** — 8 direct notify calls, no quiet hours, high frequency alerts
2. **`presence_notifications.yaml`** — 4 per-person automations, should route through `script.notify_presence_event`
3. **`alerts_device_power.yaml`** — resolve dual delivery before migrating to script
4. **`water_reporting.yaml`** — weekly summary, lower urgency
5. Remaining files — low priority

---

## 7. PLATFORM DELIVERY TARGETS

### Defined Notify Platforms

```
notify.STD_Information    # Group: mobile apps (dev + family)
notify.STD_Warning        # Group: mobile apps (dev + family)
notify.STD_Critical       # Group: mobile apps (dev + family)
notify.STD_Alerts         # Group: mobile apps + Telegram
notify.telegram_bot_5527  # Telegram Chat Bot (dedicated service)
notify.mobile_app_*       # Individual mobile devices
```

### Canonical Call Patterns

**HA Notify Groups (scripts):**
```yaml
- action: notify.STD_Critical
  continue_on_error: true
  data:
    title: "..."
    message: "..."
```

**Telegram (scripts):**
```yaml
- action: notify.send_message
  continue_on_error: true
  target:
    entity_id: notify.telegram_bot_5527
  data:
    title: "..."
    message: "..."
```

**DEPRECATED — do not use:**
```yaml
# ❌ Missing target structure
action: notify.telegram_bot_5527

# ❌ Pre-2024 service syntax
service: notify.telegram_bot_5527
```

---

## 8. WATER NOTIFICATIONS AUTOMATION — DEAD TRIGGER

`water_notifications.yaml` contains automation `water_tank_full_notification` which
triggers on `sensor.water_state → "full"`.

Per the Water Domain Contract, `sensor.water_state` never produces the state `"full"`.
The valid terminal state is `"idle"` (after a completed fill). This automation has never
fired and cannot fire without a water state machine fix.

**Related:** `water_refill_never_reached_full` triggers on `binary_sensor.water_tank_refilling`
for 3 hours — this is structurally sound as an escalation monitor.

---

## 9. CROSS-DOMAIN DEPENDENCIES

| Dependency | Consumer | Provider | Status |
|------------|----------|----------|--------|
| `input_boolean.notifications_enabled` | water_notifications.yaml | notifications_helpers.yaml | ✅ YAML-only (storage removed 2026-04-21) |
| `input_boolean.notifications_quiet_hours` | All scripts | notifications_quiet_hours.yaml | ✅ |
| `sensor.water_state` | water_notifications.yaml | water package | ❌ Dead — state "full" never produced |
| `binary_sensor.water_tank_refilling` | water_notifications.yaml | water package | ✅ |
| `binary_sensor.ryan_unknown_ap` etc. | presence_notifications.yaml | presence package | ✅ |
| `alert.power_alert` | alerts_power.yaml | alert entity | ✅ |

---

## 10. BUGS

### BUG-N01 [MEDIUM] ~~Missing `continue_on_error` on Telegram in notify_water_events.yaml~~ ✅ FIXED 2026-04-14

**File:** `packages/notifications/notify_water_events.yaml`  
**Description:** The Telegram notify call lacked `continue_on_error: true`. All other
notification scripts have this flag. During HA boot the Telegram service initializes after
automations, causing error on any water event triggered at startup.  
**Fix applied:** Added `continue_on_error: true` to the `notify.send_message` Telegram call.
Also fixed in same pass: `sev` variable was missing `| default('information')` guard (unlike all other scripts).

---

### BUG-N02 [MEDIUM] Wrong counter entity in notify_water_events.yaml weekly summary

**File:** `packages/notifications/notify_water_events.yaml`  
**Description:** References `counter.water_borehole_faults_week` but correct entity is
`counter.water_borehole_faults_this_week`. Weekly borehole fault count always reports
unknown or zero.  
**Fix:** Replace `counter.water_borehole_faults_week` with `counter.water_borehole_faults_this_week`.

---

### BUG-N03 [HIGH] ~~`input_boolean.notifications_enabled` defined only via UI — not in YAML~~ ✅ FIXED 2026-04-15

**File:** None (UI-created)  
**Description:** `input_boolean.notifications_enabled` is gated in `water_notifications.yaml`
but is not defined in any YAML package. If the HA instance is restored from config backup
without entity registry, this entity is lost.  
**Fix:** Add to `notifications_helpers.yaml`:
```yaml
input_boolean:
  notifications_enabled:
    name: Notifications Enabled
    initial: true
    icon: mdi:bell
```

---

### BUG-N04 [HIGH] ~~Temperature alert routing bypasses notification pipeline entirely~~ ✅ FIXED 2026-04-15

**File:** `packages/alerts/alerts_temperature.yaml`  
**Description:** 4 routing automations (WAN Temp, LAN Temp, Device Temp, Storage Temp)
call `notify.STD_*` directly. No quiet hours, no Telegram delivery, no logbook entry.
High-frequency alert type with no suppression.  
**Fix:** Replace direct notify calls with `script.notify_system_event` or create
`script.notify_temperature_event` following canonical script pattern.

---

### BUG-N05 [MEDIUM] ~~Presence unknown AP alerts bypass notification pipeline~~ ✅ FIXED 2026-04-15

**File:** `packages/notifications/presence_notifications.yaml`  
**Description:** 4 per-person unknown AP alert automations call `notify.send_message →
notify.STD_Information` directly. No quiet hours check. No Telegram. If sent at night
these wake all household members.  
**Fix:** Route through `script.notify_presence_event` with severity `information` to
respect quiet hours.

---

### BUG-N06 [MEDIUM] ~~Device power fault sends dual notification~~ ✅ RESOLVED (verified clean 2026-04-15 — route_device_power_alert already removed in prior session)

**File:** `packages/alerts/alerts_device_power.yaml`  
**Description:** The `route_device_power_alert` automation calls `notify.STD_Critical/Warning`
directly AND `alert.device_power_fault` also fires to `STD_Alerts`. Every device power
fault sends two notifications to mobile apps.  
**Fix:** Remove direct notify calls from `route_device_power_alert` and rely solely on
`alert.device_power_fault` delivery, or remove the alert entity and standardize on the
script path.

---

### BUG-N09 [HIGH] ~~All 6 scripts: incorrect MarkdownV2 escape chain applied to old Markdown bot~~ ✅ REVISED 2026-04-30

**Files:** All 6 `notify_*_event.yaml` scripts  
**History:** 2026-04-13 fix added `.` and `!` to complete an 18-character MarkdownV2 escape chain.
But the Telegram bot config entry (`core.config_entries`) has `options.parse_mode: markdown` (old Markdown),
not `markdownv2`. Old Markdown does not recognise `\.`, `\(`, `\+` etc. as escape sequences —
they rendered literally, producing visible backslashes in all Telegram messages.  
**Fix 2026-04-30:** Reduced all 6 escape chains to 4 characters only: `\\` (backslash), `\*` (bold prevention),
`\_` (italic prevention), `` \` `` (code prevention). These are the only special characters in old Telegram
Markdown. Do NOT expand this chain unless the bot integration is changed to `parse_mode: markdownv2`.

---

### BUG-N08 [LOW] ~~All 6 scripts: missing `| default('')` on escaped Telegram variables~~ ✅ FIXED 2026-04-13

**Files:** All 6 `notify_*_event.yaml` scripts  
**Description:** `escaped_title` and `escaped_message` variable blocks referenced `title` and
`message` directly without a default guard. Any caller that omits the `title` or `message`
argument causes HA to throw `TemplateError: 'title' is undefined` on every render of the
Telegram mirror step (approximately 20–30 occurrences per log cycle).  
**Fix applied:** Added `| default('')` as the first filter in both escape chains in all 6 scripts:
```yaml
escaped_title: >
  {{ title | default('') | replace('\\', '\\\\') | ... }}
escaped_message: >
  {{ message | default('') | replace('\\', '\\\\') | ... }}
```

---

### BUG-N10 [MEDIUM] ~~Missing `from:` guards on state triggers — spurious fires on HA restart~~ ✅ FIXED 2026-04-13

**Files:** `presence_notifications.yaml`, `admin_notifications.yaml`, `water_notifications.yaml`  
**Description:** 6 automations in notification package files had state triggers with only `to:` but
no `from:` guard. On HA restart or template reload, entities transition from `unknown`/`unavailable`
to their real state, which looks like a real state change. All 6 automations would fire on every restart:
- `presence_notifications.yaml`: Ryan/Vicky/Luke/Tayla unknown AP automations (`to: "on"` on binary_sensor)
- `admin_notifications.yaml`: Unknown UniFi AP Detected (`to: "on"` with `for: 2min`)
- `water_notifications.yaml`: `water_tank_full_notification` (`to: "full"`) and `water_refill_never_reached_full` (`to: "on"` with `for: 3h`)

**Fix applied:**
- Binary sensors / input_booleans: added `from: "off"` — prevents trigger from unknown/unavailable→on
- `water_tank_full_notification`: added `not_from: [unknown, unavailable]` (state sensor, not boolean)
- `water_refill_never_reached_full`: added `from: "off"` to binary_sensor trigger

**Also fixed** in same pass (outside notifications package):
- `lighting_bar_presence.yaml`: `bar_occupied` trigger → `from: "off"`
- `lighting_arrival_night.yaml`: `arrival_detected` trigger → `from: "off"`
- `lighting_morning.yaml`: `bedrooms_occupied` trigger → `from: "off"`
- `energy_state.yaml`: `grid_charging_while_solar` trigger → `not_from: [unknown, unavailable]`

---

### BUG-N11 [LOW] ~~`notify_power_event.yaml`: `title` undefined warning on every render~~ ✅ FIXED 2026-04-22

**File:** `packages/notifications/notify_power_event.yaml`  
**Description:** All three severity branches (information / warning / critical) rendered the
title field as `[{{ subsystem | default('system') | upper }}] {{ title }}` with no default
guard on `title`. Any caller that omits the `title` argument (e.g. callers that only pass
`message` and `sev`) caused HA to log `TemplateError: 'title' is undefined` on each render.  
**Note:** `notify_water_events.yaml` already had `{{ title | default('Notification') }}` — this
was inconsistent.  
**Fix applied:** Added `| default('Notification')` to all 3 `{{ title }}` occurrences in
`notify_power_event.yaml` (lines 86, 104, 122). Now consistent with `notify_water_events.yaml`.

---

### BUG-N07 [LOW] Water tank full notification can never fire

**File:** `packages/notifications/water_notifications.yaml`  
**Description:** Automation `water_tank_full_notification` triggers on
`sensor.water_state → "full"`. Per Water Domain Contract, this state is never produced.
Automation is dead code.  
**Fix:** Change trigger state to `"idle"` after verifying with water state machine, or
remove automation and replace with a `done_message` on the water alert entity.

---

## 11. DESIGN RULES (MUST NOT VIOLATE)

1. **All notification calls MUST use `continue_on_error: true`** — startup race condition
   protection. Mandatory on every notify action.

2. **All domain notifications MUST route through the canonical script** — domain automations
   must not call `notify.STD_*` or `notify.send_message` directly. Use `script.notify_*_event`.

3. **Quiet hours suppression MUST be handled by the notification script** — not by the
   caller. Scripts already implement this. Callers that bypass scripts bypass quiet hours.

4. **`input_boolean.notifications_enabled` MUST be defined in YAML** — any helper gated
   by domain automations must exist in a YAML package file, not as a UI-created entity.

5. **Telegram calls MUST use `notify.send_message` with `target: entity_id` structure** —
   not legacy `action: notify.telegram_bot_5527` syntax.

6. **Notification scripts are the boundary for Markdown escaping** — callers pass raw
   strings; scripts escape before Telegram delivery.

7. **All state triggers MUST include `from:` / `not_from:` guards** — `to:` alone fires on
   HA restart when entities restore from `unknown`/`unavailable`. Rule: binary sensors and
   input_booleans use `from: "off"`; state/template sensors use `not_from: [unknown, unavailable]`.

---

## 12. RELATED CONTRACTS

- `ALERTS_CONTRACT.md` — alert pipeline, binary sensor → context sensor → alert entity
- `WATER_CONTRACT.md` — water state machine (governs water_notifications dead trigger)
- `PRESENCE_CONTRACT.md` — presence/trust model (governs presence_notifications routing)
- `POWER_CONTRACT.md` — power alert pipeline feeding notify_power_event
