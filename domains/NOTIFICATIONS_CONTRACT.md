##########################################################
# NOTIFICATIONS CONTRACT
# HABiesie — Notifications Domain
# Generated: 2026-04-13
# Last updated: 2026-07-13 — Vicky onboarded as a 5th per-device notify
# target in notify_security_events.yaml + notify_power_event.yaml (warning/
# critical branches only, info branch excluded). See Section 4A "Per-Person
# Onboarding".
# Last updated: 2026-07-07 — notify_system_event.yaml gained optional
# `actions:` field passthrough (warning/critical only), restoring the garden
# TURN_OFF_POND_PUMP mobile action button (BUG-A12, see ALERTS_CONTRACT.md).
# Last updated: 2026-07-06 — severity/sound classification overhaul: universal
# warning-tier sound (all 6 scripts), power+presence critical silently-broken
# data.data bug fixed, power Telegram inline_keyboard bug fixed, arrivals/
# departures promoted to warning, STD_Alerts pipeline fixed for 13 domains via
# routing automations (see ALERTS_CONTRACT.md). See PROJECT_STATE.md 2026-07-06.
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

| Severity | Quiet Hours | HA Mobile | Telegram | Sound/Channel (2026-07-06) |
|----------|-------------|-----------|----------|------|
| information | suppressed | ✅ | ✅ | default (no distinct sound) |
| warning | bypass | ✅ | ✅ | iOS `interruption-level: time-sensitive`; Android `channel: security_warning` — ONE shared warning sound reused across every domain (power/water/presence/system/lighting/security), no new on-device setup |
| critical | bypass | ✅ | ✅ | iOS `interruption-level: critical`; all `channel: alarm`, `ttl: 0`, `priority: high` |

**2026-07-06 — universal warning-tier sound:** previously only `notify_security_events.yaml` had the warning-tier sound (S17c, 2026-07-03). Extended the identical treatment to `notify_power_event.yaml`, `notify_water_events.yaml`, `notify_presence_events.yaml`, `notify_system_event.yaml`, `notify_light_events.yaml` — all 6 scripts now use the legacy per-device `notify.mobile_app_*` pattern for both warning and critical (info stays on plain `notify.send_message`, which can't carry the extra push/channel data). The `security_warning` Android channel name is reused verbatim across all domains by design — one shared warning sound, not per-category sounds, per user's explicit choice.

**2026-07-06 — arrivals/departures promoted to warning:** `security_automations.yaml`'s 4 arrival/departure notification call sites (Arrival Stage 1/2, Departure Stage 1/2) changed from `severity: "information"` to `"warning"` — always audible, no quiet-hours exception. The "Dogs home alone?" departure prompt stays `critical`, unchanged.

**Exception — `dogs_inside_prompt: true` in `notify_security_event`:**
When `dogs_inside_prompt: true` is passed, Telegram is suppressed for ALL severity levels
(the prompt is a phone-action notification — action buttons don't work on Telegram). The
information severity quiet-hours check is also bypassed so the dogs prompt fires even during
sleep hours (departure can happen at any time). Added 2026-06-14.

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

## 3A. PER-PERSON ONBOARDING

**Added 2026-07-13.** This repo has no per-person notification-preference infrastructure
(no `input_boolean.<person>_notify_<domain>` toggles, no dashboard-editable routing) — every
domain script hardcodes a static list of per-device targets directly inside each severity
branch. Onboarding a new household member's device means editing that static list in whichever
domain script(s) they should receive, in whichever severity branch(es) they should receive.
This was a deliberate choice (user picked "hardcode into chosen scripts" over building a
reusable toggle system) — faster, matches the existing convention, smaller blast radius.

**Vicky Dunnington** — onboarded 2026-07-13:
- Targets: `notify.vicky_iphone13_mobile_app` (entity, unused so far — this repo's canonical
  info/warning `notify.send_message` pattern isn't used by either script she's in) /
  `notify.mobile_app_iphone13promax_vicky` (legacy per-device service — confirmed live via
  `GET /api/services`, this is what's actually wired in, matching the pattern both chosen
  scripts already use for warning/critical)
- Domains: **Security** (`notify_security_events.yaml`), **Power** (`notify_power_event.yaml`)
  only. NOT wired into Water, Presence, System, or Lighting.
- Severity: **warning + critical only** — added as a 5th per-device call in both branches,
  immediately after the existing `honorx7_dash` call. Deliberately NOT added to the
  information branch in either script (explicit user request — avoid spamming her with
  low-signal events she doesn't have context for).
- Live-verified: real test warning fired through `script.notify_security_event`; logbook
  confirmed `notify.vicky_iphone13_mobile_app`'s state updated immediately after with no
  errors for that context.

**To onboard another domain for Vicky, or another person entirely:** find the
`notify.mobile_app_<device>` warning/critical `choose:` branches in the target
`notify_<domain>_event.yaml` script(s), and add a new per-device call block immediately after
the last existing one — same shape as the other 4, with `continue_on_error: true`. Skip the
information branch unless explicitly asked to include it. Confirm the legacy service name
exists first via `GET /api/services` (mobile_app device names don't always slugify the way you'd
guess — e.g. `iPhone13promax_Vicky` → `mobile_app_iphone13promax_vicky`).

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
| notify_security_event.yaml | ✅ | information suppressed (exception: dogs_inside_prompt bypasses quiet hours + Telegram) |
| notify_presence_event.yaml | ✅ | information suppressed |
| notify_system_event.yaml | ✅ | information suppressed |
| notify_lighting_event.yaml | ✅ | information suppressed |
| presence_notifications.yaml | ✅ routes via script.notify_presence_event | information suppressed (fixed 2026-04-15) |
| water_notifications.yaml | ❌ BYPASS | always sends |
| alerts_temperature.yaml | ✅ routes via script.notify_system_event | quiet hours respected (fixed 2026-04-15) |
| alerts_device_power.yaml | ✅ alert entity only (route automation removed) | no bypass (verified 2026-04-15) |

Warning and Critical notifications bypass quiet hours in all scripts (correct behavior).

### script.notify_security_event — Extended Fields (2026-06-14)

Standard fields: `severity`, `title`, `message`, `image`, `source`, `gate_control`

**`dogs_inside_prompt` (bool, default false) — added 2026-06-14:**
When true, replaces gate action buttons with `DOGS_INSIDE_ON` / `IGNORE` in all severity levels.
Telegram is suppressed (the mobile action button pattern doesn't work via Telegram).
Information severity bypasses quiet hours (departure can happen at any time).
Used by Stage 1 departure sequence — fires after "🚗 Departure — vehicle leaving" notification
when `dogs_inside = off AND NOT guest_mode AND NOT staff_on_site`.
Handler: `automation.dogs_inside_from_notification` (security_automations.yaml) — turns on
`input_boolean.dogs_inside` when DOGS_INSIDE_ON action is tapped.

### script.notify_system_event — Extended Fields (2026-07-07)

Standard fields: `severity`, `title`, `message`

**`actions` (list, optional) — added 2026-07-07:**
Passthrough list of mobile action buttons, e.g.
`[{"action": "SOME_ACTION", "title": "Button Label"}]`. Applied to the nested
`data:` block of every per-device `notify.mobile_app_*` call on the
**warning and critical branches only**. Not supported on the information
branch — that branch uses `notify.send_message`, which structurally rejects
any extra `data` key (same class of bug as BUG-N13/N14; confirmed live
2026-07-03). When omitted, resolves to `omit` (no `actions` key sent at all)
— fully backward-compatible with every existing caller.

Added to close BUG-A12 (ALERTS_CONTRACT.md) — `alerts_garden.yaml`'s pond-pump
warning now passes `actions: [{action: "TURN_OFF_POND_PUMP", title: "Turn Off Pump"}]`.
Any future caller needing a mobile action button on a system-domain warning/
critical push can reuse this field rather than bypassing the script.

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
notify.STD_Alerts         # Group: KNOWN BROKEN — see bug note below
notify.telegram_bot_5527  # Telegram entity (NOT a bare service — see below)
notify.ryan_iphone16_mobile_app, notify.ap_0223_1001,
notify.honor_10_dash_mobile_app, notify.honor_x7_dash_mobile_app  # Individual mobile devices
```

**BUG FIXED 2026-06-28 (two-pass — first attempt was wrong):** all four `notify.STD_*`
groups (configuration.yaml) originally referenced legacy mobile_app service names
(`mobile_app_iphone16promax_ryan`, `mobile_app_ap_0223_1001`, `mobile_app_honor10_dash`,
`mobile_app_honorx7_dash`). First fix attempt just renamed the group members to the
current entity slugs (e.g. `ryan_iphone16_mobile_app`) — **this did not work** — the
legacy `platform: group` mechanism calls member services by name internally, and `notify.telegram_bot_5527`
as a bare service throws `ServiceNotFound`. The `platform: group` mechanism cannot work
with entity-based targets regardless of what name is used.

**Real fix:** all 19 `notify.STD_Information` / `notify.STD_Warning` / `notify.STD_Critical`
call sites, across `notify_power_event.yaml`, `notify_water_events.yaml`,
`notify_system_event.yaml`, `notify_light_events.yaml`, `notify_presence_events.yaml`,
`notify_security_events.yaml`, were converted to call `notify.send_message` directly with
an explicit `target.entity_id` list of all 4 device entities. The three now-unused
`STD_Information`/`STD_Warning`/`STD_Critical` group definitions were removed from
configuration.yaml. `packages/admin/tablets.yaml` had the same dead-name bug — fixed to
`notify.send_message` + `target.entity_id`.

**CORRECTION 2026-07-01 — nested data: silently dropped ALL notifications + legacy services ARE still valid:**
`notify.send_message` schema (verified via `GET /api/services`) has exactly two fields: `message` and `title` only.
Any extra field in `data:` — including a nested `data: { image, actions, push, channel, ttl }` block — returns
HTTP 400 Bad Request. The 2026-06-28 PART 5 fix correctly replaced STD_* groups but preserved the nested `data:`
block in every critical branch, so ALL notifications continued to silently fail via `continue_on_error: true`.
Confirmed via recorder: 2026-06-30 camera pipeline ran correctly, but zero push notifications reached any phone.
**ALSO CORRECTED:** `notify.mobile_app_iphone16promax_ryan` / `notify.mobile_app_ap_0223_1001` /
`notify.mobile_app_honor10_dash` / `notify.mobile_app_honorx7_dash` ARE still registered as bare services
(confirmed `GET /api/services` 2026-07-01) with a `data:` field in their schema — the PART 5 claim that
"legacy services no longer exist" was wrong. These legacy services are now the ONLY way to pass `data: { push: { interruption-level: critical } }` for iOS DND bypass and Android alarm channel.
**Two-tier fix applied 2026-07-01:** information + warning → `notify.send_message` (schema-valid);
critical → 4 separate `notify.mobile_app_*` calls with `data: { push.interruption-level: critical, channel: alarm, ttl: 0, priority: high }`.

**FIXED 2026-07-06 (option (b) below, chosen and implemented):** `notify.STD_Alerts` itself
is STILL left broken-but-defined in `configuration.yaml` (harmless either way, same as the
temperature domain already did) — but all 13 domains that had ZERO delivery because of it
now have a parallel routing automation added directly in their `packages/alerts/<domain>.yaml`
file: `alerts_network.yaml` (4 subtypes), `alerts_device_power.yaml`, `alerts_media.yaml`,
`alerts_system_health.yaml`, `alerts_water.yaml` (tank-low + both borehole fault tiers),
`alerts_presence.yaml`, `alerts_garden.yaml`, `alerts_batteries.yaml`, `alerts_doors.yaml`
(sustained-open escalation only — the transient gate-open/close pings already worked),
`alerts_power.yaml` (grid-offline/battery-low warning tiers — the SOC-critical case already
had separate coverage via `power_battery_soc_critical_alert`). Each new automation triggers
on the domain's underlying binary_sensor/severity-sensor and calls the working
`script.notify_*_event` directly — same pattern the temperature domain already proved
(`alerts_temperature.yaml`, Route WAN/LAN/Device/Storage Temp Alert). **Every one of these
automations MUST include `from: "off"` (binary_sensor triggers) / `not_from: ["unknown",
"unavailable"]` (severity-sensor triggers)** — omitting this caused a real incident during
the 2026-07-06 rollout: reloading `template:` briefly puts every template entity through
unknown/unavailable while it recomputes, and an unguarded `to: "on"`/`to: "critical"`
trigger fires on that transient, producing false CRITICAL pushes. See PROJECT_STATE.md
2026-07-06 for the live incident and fix; see ALERTS_CONTRACT.md for the full per-domain
bug entries (BUG-A10 through BUG-A13 area).

**Deliberately left unfixed:** `alert.security_alert`'s repeat reminders (5/15/30/60min)
and `alert.camera_health`'s repeat reminders (60/240min) — see ALERTS_CONTRACT.md for why
(duplicate-delivery risk for security; camera_health's `notifiers: [STD_Warning]` was
instead removed outright since that group doesn't exist at all, stopping a hard error,
rather than reintroduced with a notifier).

**Option (a) (HA "Notify Group" UI helper) was considered and NOT used** — the
automation-based fix above is git-trackable, consistent with every other fix already made
in this codebase, and doesn't depend on unverified `alert:`-to-group-entity compatibility.

### Canonical Call Patterns

**Information / Warning — `notify.send_message` with entity target (schema-valid, no data: support):**
```yaml
- action: notify.send_message
  continue_on_error: true
  target:
    entity_id:
      - notify.ryan_iphone16_mobile_app
      - notify.ap_0223_1001
      - notify.honor_10_dash_mobile_app
      - notify.honor_x7_dash_mobile_app
  data:
    title: "..."
    message: "..."
    # ⚠️ NO other fields — schema rejects anything except message and title (HTTP 400)
```

**Critical — legacy per-device services (supports data: for alarm channel + iOS DND bypass):**
```yaml
- action: notify.mobile_app_iphone16promax_ryan    # iOS — use push.interruption-level: critical
  continue_on_error: true
  data:
    title: "..."
    message: "..."
    data:
      push:
        interruption-level: critical
      channel: alarm
      ttl: 0
      priority: high
- action: notify.mobile_app_ap_0223_1001            # Android — no push: needed
  continue_on_error: true
  data:
    title: "..."
    message: "..."
    data:
      channel: alarm
      ttl: 0
      priority: high
- action: notify.mobile_app_honor10_dash
  continue_on_error: true
  data:
    title: "..."
    message: "..."
    data:
      channel: alarm
      ttl: 0
      priority: high
- action: notify.mobile_app_honorx7_dash
  continue_on_error: true
  data:
    title: "..."
    message: "..."
    data:
      channel: alarm
      ttl: 0
      priority: high
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
    inline_keyboard:        # Telegram-specific; accepted by Telegram's send_message schema
      - "Label:/command"
```

**DEPRECATED — do not use:**
```yaml
# ❌ platform: group mechanism calls member names as bare services internally — broken for entity targets
action: notify.STD_Information   # ❌ group removed from configuration.yaml 2026-06-28
action: notify.STD_Warning       # ❌ group removed
action: notify.STD_Critical      # ❌ group removed

# ❌ Telegram as bare service — registers entity-only, no bare notify.telegram_bot_5527 service
action: notify.telegram_bot_5527

# ❌ Pre-2024 service syntax
service: notify.telegram_bot_5527
```

**CONFIRMED REMOVED 2026-06-28 — companion-app commands (e.g. screen brightness) via notify:**
```yaml
# ❌ ALL of these fail with 400 "extra keys not allowed @ data['data']" / "@ data['command']"
action: notify.send_message
target:
  entity_id: notify.honor_10_dash_mobile_app
data:
  message: "command_screen_brightness_level"
  data:
    command: 20          # ❌ nested data — rejected

data:
  message: "command_screen_brightness_level"
  command: 20             # ❌ flat field — rejected
```
Verified directly against the live HA REST API (`GET /api/services` → `notify.send_message`
schema has exactly two fields: `message`, `title` — no `data` field at all) and reproduced
independently via Developer Tools → Actions. No replacement controllable entity exists for
screen brightness — every mobile_app entity on both tablet devices is a read-only
`sensor.`/`binary_sensor.` (checked all 40+ entities per device). This is what disabled
`packages/admin/tablets.yaml`'s 4 brightness automations (`tablets_brightness_night_dim`,
`_morning_restore`, `_away_dim`, `_return_restore`) on 2026-06-28 — not fixable from config
until mobile_app exposes a controllable brightness entity or restores command support.

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

### BUG-N13 [HIGH] ~~notify_power_event.yaml + notify_presence_events.yaml critical branches silently failing~~ ✅ FIXED 2026-07-06

**Files:** `packages/notifications/notify_power_event.yaml`, `packages/notifications/notify_presence_events.yaml`
**Description:** Both scripts' critical branches still called `notify.send_message` with a
nested `data: {push, channel, ttl, priority}` block — the exact "extra keys not allowed
@ data['data']" bug fixed everywhere else on 2026-07-01 (security/water/system/lighting all
moved off this pattern), but power and presence were missed. `continue_on_error: true`
swallowed the HTTP 400 silently. Real-world impact: `power_battery_soc_critical_alert`
(power_automations.yaml) and both prepaid-critical automations (prepaid_core.yaml,
prepaid_strategy.yaml) had been failing to reach any phone for critical severity.
**Fix applied:** Converted both critical branches to the legacy per-device
`notify.mobile_app_*` pattern already proven in `notify_security_events.yaml`.

---

### BUG-N14 [HIGH] ~~notify_power_event.yaml Telegram critical branch — inline_keyboard rejected~~ ✅ FIXED 2026-07-06

**File:** `packages/notifications/notify_power_event.yaml`
**Description:** The Telegram mirror for critical severity called `notify.send_message`
targeting `notify.telegram_bot_5527` with `inline_keyboard` at top level of `data:` — the
same rejection as the security Telegram crash already fixed in S17c
(`extra keys not allowed @ data['inline_keyboard']`), just never applied to power. Live-
reproduced while testing BUG-N13's fix; confirmed firing on real prepaid-critical
automations (`Prepaid Critical Night Protection`, `Prepaid Buy Decision Notification`) via
`ha core logs`. The mobile push itself was unaffected (runs before this step) — only the
Telegram Acknowledge button/message was silently missing.
**Fix applied:** Switched to `telegram_bot.send_message` directly, same as security's S17c fix.

---

### BUG-N01 [MEDIUM] ~~Missing `continue_on_error` on Telegram in notify_water_events.yaml~~ ✅ FIXED 2026-04-14

**File:** `packages/notifications/notify_water_events.yaml`  
**Description:** The Telegram notify call lacked `continue_on_error: true`. All other
notification scripts have this flag. During HA boot the Telegram service initializes after
automations, causing error on any water event triggered at startup.  
**Fix applied:** Added `continue_on_error: true` to the `notify.send_message` Telegram call.
Also fixed in same pass: `sev` variable was missing `| default('information')` guard (unlike all other scripts).

---

### ~~BUG-N02~~ [MEDIUM] Wrong counter entity in weekly water summary
**Status:** ✅ Doc-drift correction 2026-07-08 — already correct in live code, no code change needed; also corrects a wrong file name in this entry.

**File:** `packages/notifications/water_notifications.yaml:154` (this entry previously
named `notify_water_events.yaml`, a different file that has no such reference at all).
**Was claimed:** References `counter.water_borehole_faults_week` but correct entity is
`counter.water_borehole_faults_this_week`. Weekly borehole fault count always reports
unknown or zero.
**Re-verified live 2026-07-08:** `water_notifications.yaml:154` already reads
`states('counter.water_borehole_faults_this_week')` — correct. Grep-confirmed zero
occurrences of the wrong name anywhere in `packages/`. Same claim also lived in
WATER_CONTRACT.md Issue 8 — corrected there too, same session (see PRESENCE_CONTRACT.md
BUG-P17 session for the pattern of drift this was found alongside).

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

### ~~BUG-N07~~ [LOW] ✅ Doc-drift correction 2026-07-10 — Water tank full notification can never fire

**File:** `packages/notifications/water_notifications.yaml`  
**Status:** Already fixed in live code, no code change needed. `water_tank_full_notification`'s
trigger already reads `binary_sensor.water_tank_full_depth` (`from: "off"` → `to: "on"`), not
the dead `sensor.water_state == "full"` path. This entry was stale.  
**Was:** Automation `water_tank_full_notification` triggered on
`sensor.water_state → "full"`. Per Water Domain Contract, this state is never produced.
Automation was dead code.  
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
