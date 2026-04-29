##########################################################
# HABiesie — Master System Contract
# Cross-Domain Architecture Audit
# Generated: 2026-04-13 | Updated: 2026-04-16
# Source: All 10 domain contracts + PROJECT_STATE.md
# Contracts: WATER, SECURITY, POWER, PRESENCE, ALERTS, NOTIFICATIONS,
#            LIGHTING, NETWORK, CONTEXT, INFRA
##########################################################

---

## Section 1: System Architecture Overview

HABiesie is a 12-package Home Assistant configuration using a package-based modular
design. Domains are grouped into layers:

```
┌────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE LAYER                         │
│                                                                  │
│  power/         ─── inverters, solar, battery, prepaid         │
│  water/         ─── tank refill lifecycle, safety, audit        │
│  presence/      ─── AP location, occupancy, trust model        │
│  context/       ─── night mode, global state (SHARED STATE)    │
│  core/          ─── HA uptime, CPU, startup guard              │
│  network/       ─── WAN health, UniFi, latency, NOC status     │
│  backup/        ─── GitHub config backup (5AM + manual)        │
│  sensors/       ─── sensor filtering/smoothing utilities       │
└──────────────────────────┬─────────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────────┐
│                    DECISION LAYER                                │
│                                                                  │
│  security/      ─── cameras, event classification, threat      │
│  lighting/      ─── presence-aware, time-based scenes          │
│  alerts/        ─── device/system/domain alert aggregation     │
│  office/        ─── printer monitoring                         │
│  weather/       ─── OWM API health, staleness tracking         │
│  integrations/  ─── Sonoff/EWElink config                      │
└──────────────────────────┬─────────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────────┐
│                    DELIVERY LAYER                                │
│  notifications/ ─── severity routing, quiet hours, Telegram    │
└────────────────────────────────────────────────────────────────┘
```

### Canonical Data Flow (Alerts Pipeline)

```
Domain state change (power/water/presence/security/temperature)
    ↓
binary_sensor.*_alert_active        (logic + anti-flap)
    ↓
sensor.*_alert_context              (severity + devices list attribute)
    ↓
sensor.alert_device_entities        (aggregator — scans all context sensors)
    ↓
sensor.global_alert_context         (critical / warning / info / normal)
    ↓
Dashboard alert widget + script.notify_*_event
```

### Canonical Data Flow (Notification Pipeline)

```
Domain automation detects condition
    ↓
script.notify_*_event(severity, title, message)
    ↓
Quiet hours check  →  route to notify.STD_*
    ↓
Telegram mirror  →  notify.telegram_bot_5527
```

---

## Section 2: Cross-Domain Dependency Matrix

### Published Outputs (Consumed by Other Domains)

| Entity | Owned By | Consumed By | Status |
|--------|----------|-------------|--------|
| `binary_sensor.anyone_connected_home` | presence | security, lighting, alerts | ✅ |
| `binary_sensor.anyone_home` | presence (Mobile App) | lighting | ✅ |
| `binary_sensor.low_trust_present` | presence (derived) | security, lighting_departure, alerts_doors | ✅ FIXED (was broken — IV-01/IV-02 resolved) |
| `input_boolean.low_trust_present` | context (legacy manual) | ~~security, lighting~~ — do not use | ❌ NEVER AUTO-SET — replaced by binary_sensor |
| `binary_sensor.staff_on_site` | presence (derived) | context_global, security | ✅ |
| `input_boolean.staff_on_site` | context (legacy manual) | ~~security~~ — do not use | ❌ NEVER AUTO-SET — replaced by binary_sensor |
| `sensor.security_trust_mode` | security (reads binary_sensor now) | alerts_doors, context_global | ✅ FIXED (IV-01 resolved) |
| `binary_sensor.security_low_trust_active` | security (reads binary_sensor now) | lighting_security | ✅ FIXED |
| `binary_sensor.main_gate_sensor` | ZHA hardware | presence, security | ✅ |
| `binary_sensor.night_confirmed` | context (context_night) | security, lighting | ✅ |
| `binary_sensor.security_night_mode` | context (alias of night_confirmed) | security | ✅ |
| `input_boolean.system_startup` | core (core_helpers) | presence (confidence gate) | ✅ |
| `input_boolean.guest_mode` | context | security | ✅ (manual + startup-set) |
| `input_boolean.holiday_mode` | context | security, lighting | ✅ (manual) |
| `input_boolean.entertaining_mode` | context | security | ✅ (manual) |
| `input_boolean.arrival_detected` | presence (helpers) | security | ❌ NEVER SET by resolver |
| `sensor.security_threat_level` | security | security_automations, lighting | ✅ |
| `sensor.security_event_classification` | security | security_event_router | ✅ |
| `sensor.security_movement_path` | security | lighting_security | ✅ |
| `sensor.security_lighting_intent` | security | lighting_security | ✅ |
| `sensor.security_mode` | security (core) | notify_security_events | ✅ |
| `group.inverter_grid` | power | water, alerts_power | ✅ |
| `sensor.inverter_battery_soc` | power | water (via battery SOC check) | ✅ |
| `sensor.inverter_pv_power` | power | alerts_power | ✅ |
| `sensor.inverter_power` | power | alerts_power | ✅ |
| `sensor.inverter_1_battery` | power (slave direct) | alerts_power | ⚠️ bypasses published SOC |
| `sensor.house_energy_resilience_hours` | power | alerts_power (devices list) | ✅ |
| `sensor.house_energy_resilience_status` | power | alerts_power (devices list) | ✅ |
| `sensor.load_shedding_status_card` | power | context_global | ✅ |
| `binary_sensor.load_shedding_active` | power | context_global | ✅ |
| `sensor.power_state` | power | lighting (energy save) | ⚠️ intent exists, wiring unclear |
| `binary_sensor.water_refill_allowed` | water | water_tank_refill_control | ✅ (internal use) |
| `sensor.water_state` | water | water_notifications | ⚠️ dead trigger state "full" |
| `sensor.global_alert_context` | alerts | dashboard | ✅ |
| `sensor.alert_device_entities` | alerts | dashboard | ✅ |
| `input_boolean.notifications_enabled` | notifications (UI ONLY) | water_notifications | ✅ NOW IN YAML (notifications_helpers.yaml — BUG-N03 fixed) |
| `input_boolean.notifications_quiet_hours` | notifications | all scripts | ✅ |
| `sensor.wan_noc_status` | network | dashboard | ✅ |
| `binary_sensor.network_alert_active` | network (alerts_network) | alerts pipeline | ✅ |
| `sensor.ha_system_health` | core | dashboard, alerts_system_health | ✅ |
| `binary_sensor.night_confirmed` | context | security, lighting, alerts_doors | ✅ |
| `binary_sensor.security_nobody_home` | context | lighting_security, dashboard | ✅ |
| `sensor.home_context` | context | dashboard | ✅ (depends on security_mode — layering inversion) |
| `sensor.weather_api_health` | weather | dashboard | ✅ |
| `sensor.printer_cartridge_state` | office | dashboard | ⚠️ binary_sensor.printer_cartridge_low broken (BUG-INF01) |

---

## Section 3: Domain Public Interfaces

What each domain EXPOSES for consumption by other domains.

### Power Domain — Published Interface

```
# Primary sensors (locked names)
sensor.inverter_battery_soc          %   average SOC
sensor.inverter_pv_power             W   PV generation
sensor.inverter_power                W   house load
sensor.inverter_grid_power           W   grid import/export
sensor.inverter_battery_power        W   battery charge/discharge
group.inverter_grid                  on/off  grid online state
group.inverter_battery_state         on/off  battery state

# Strategy outputs
sensor.house_power_health            good/watch/degraded/critical
sensor.power_state                   normal/economy/critical
sensor.power_strategy                descriptive text
sensor.grid_risk_level               low/medium/high/critical

# Load shedding
binary_sensor.load_shedding_active   on/off
sensor.load_shedding_status_card     text

# Resilience
sensor.house_energy_resilience_hours hours remaining
sensor.house_energy_resilience_status normal/warning/critical

# Flexible load groups
group.flexible_power_loads           for excess-load detection
group.critical_power_loads           for load shedding decisions
```

### Water Domain — Published Interface

```
sensor.water_state                   fault/critical/safety/low/refilling/ok
sensor.water_tank_level_percent      %
sensor.water_tank_depth_validated    m  (filtered)
binary_sensor.water_tank_refilling   on/off
binary_sensor.water_refill_allowed   on/off (permission gate)
input_boolean.water_refill_cycle_active  on/off
input_boolean.water_refill_aborted_due_to_safety  on/off
```

### Presence Domain — Published Interface

```
# Home/away (Layer 2)
binary_sensor.anyone_connected_home  on/off  (AP-based)
binary_sensor.anyone_home            on/off  (Mobile App-based)
sensor.*_ap_location                 room / Away / Disconnected

# Room occupancy (Layer 1)
binary_sensor.office_occupied        on/off
binary_sensor.garage_occupied        on/off
binary_sensor.living_areas_occupied  on/off
binary_sensor.bedrooms_occupied      on/off
binary_sensor.bar_occupied           on/off

# Trust model — CORRECT published output (Layer 3)
binary_sensor.low_trust_present      on/off  (derived, auto-set)
binary_sensor.staff_on_site          on/off  (derived, auto-set)

# Arrival
input_text.last_arrival_person       person name (10s window)
input_datetime.last_arrival_time     persistent
```

### Security Domain — Published Interface

```
sensor.security_threat_level         low/elevated/warning/critical
sensor.security_threat_score         0–100
sensor.security_event_classification critical_intrusion/intruder/service_person/arrival/family_movement/none
sensor.security_correlation          family_arrival/visitor/service_visit/intruder/intruder_high/ignore/none
sensor.security_movement_path        street/driveway/front_door/side_entry/rear_*/none
sensor.security_lighting_intent      ignore/full/area/perimeter
sensor.security_mode                 away/night/home
```

### Context Domain — Published Interface

```
binary_sensor.night_confirmed        on/off
binary_sensor.night_early            on/off
binary_sensor.security_night_mode    on/off  (alias of night_confirmed)
input_boolean.guest_mode             on/off
input_boolean.holiday_mode           on/off
input_boolean.entertaining_mode      on/off
input_boolean.system_startup         on/off (core startup guard)
sensor.global_house_state            attribute-rich state sensor
```

### Alerts Domain — Published Interface

```
sensor.global_alert_context          critical/warning/info/normal
sensor.alert_device_entities         JSON list of all active alert devices
sensor.active_alert_entities         filtered active list
sensor.total_critical_alert_devices  count
sensor.total_warning_alert_devices   count
sensor.total_info_alert_devices      count
sensor.total_active_alerts           count
```

### Notifications Domain — Published Interface

```
script.notify_power_event            severity, title, message
script.notify_water_event            severity, title, message
script.notify_security_event         severity, title, message
script.notify_presence_event         severity, title, message
script.notify_system_event           severity, title, message
script.notify_lighting_event         severity, title, message
```

---

## Section 4: Interface Violations

An interface violation is when Domain A reads an internal implementation detail
of Domain B instead of consuming its published output.

### IV-01 [CRITICAL] Security reads presence internal flags, not published trust outputs

**Domain A:** security (security_core.yaml, security_logic.yaml)  
**Domain B:** presence (via context/context_presence.yaml)

Security reads:
```
input_boolean.low_trust_present    ← INTERNAL manual flag, never auto-set
input_boolean.staff_on_site        ← INTERNAL manual flag, never auto-set
```

Security should read:
```
binary_sensor.low_trust_present    ← PUBLISHED derived output (auto-set by schedules)
binary_sensor.staff_on_site        ← PUBLISHED derived output (auto-set by schedules)
```

**Impact:** Staff presence is never recognized by security. All security events during
maid/gardener visits are classified as `intruder` or `visitor` instead of `service_visit`.
Trust-mode-dependent logic always acts as if no staff is present.

**Files to fix:**
- `packages/security/security_core.yaml` — `sensor.security_trust_mode`, `binary_sensor.security_low_trust_active`
- `packages/security/security_logic.yaml` — `sensor.security_correlation`, `sensor.security_event_classification` (lines ~155, ~158)

---

### IV-02 ✅ FIXED 2026-04-15 — Lighting reads correct trust entity

`lighting_departure.yaml` now reads `binary_sensor.low_trust_present` (published derived output).
Fix confirmed in Group A audit 2026-04-15.

---

### IV-03 [HIGH] Alerts reads security sensor that is derived from wrong inputs

**Domain A:** alerts (alerts_doors.yaml:324)  
**Domain B:** security → presence (chain)

Alerts reads `sensor.security_trust_mode`, which itself reads the wrong presence entities
(IV-01). This is a second-order violation — fixing IV-01 automatically fixes this, but
alerts is consuming a security sensor that is known-broken.

---

### IV-04 [MEDIUM] Power alerts reference per-inverter sensor, not published aggregated SOC

**Domain A:** alerts (alerts_power.yaml)  
**Domain B:** power

`binary_sensor.power_battery_low_alert_active` reads:
```
sensor.inverter_1_battery           ← internal per-inverter slave sensor
```

Should read:
```
sensor.inverter_battery_soc         ← PUBLISHED aggregated average
```

**Impact:** Battery low alert reflects only one inverter's SOC. On a Master/Slave setup,
the slave inverter measures battery. If inverter assignment changes this alert breaks.

---

### IV-05 [MEDIUM] `boundary_permissive_window` reads orphaned datetime entities

**Domain A:** security (security_core.yaml)  
**Domain B:** presence/context

`binary_sensor.boundary_permissive_window` reads:
```
input_datetime.low_trust_start     ← ORPHANED — commented out, entity unknown
input_datetime.low_trust_end       ← ORPHANED — commented out, entity unknown
```

These entities were removed from YAML (~2026-03-30) and remain only as orphaned registry
entries returning `unknown`. The sensor always evaluates `false`.

**Impact:** The permissive window during staff visits never opens. `gate_open_too_long`
warning never suppresses during scheduled staff hours.

---

### IV-06 [LOW] Water notifications reference raw Tuya sensor, not validated depth

**Domain A:** notifications (notify_water_events.yaml)  
**Domain B:** water

References `sensor.water_tank_level_sensor_liquid_level` (raw Tuya %) instead of
`sensor.water_tank_level_percent` (computed from validated depth). Raw sensor may
produce spikes.

---

## Section 5: Missing Interfaces

Places where Domain A should be consuming Domain B's output but currently isn't.

### MI-01 [CRITICAL] Water domain has no alert pipeline entry

**Gap:** Water state changes (critical depth, safety state, pump fault) never enter the
alert aggregation pipeline. `alerts_water.yaml` is entirely commented out.

**Consequence:** `sensor.global_alert_context` never reflects water problems.
The dashboard alert widget shows no water alerts. No escalating notifications via alert entity.

**Required interface:** `sensor.water_alert_context` (derived from `sensor.water_state`)
→ `sensor.alert_device_entities` → `sensor.global_alert_context`.

---

### MI-02 [CRITICAL] Security domain has no alert pipeline entry

**Gap:** Security threat events never enter the alert aggregation pipeline.
`alerts_security.yaml` is entirely empty.

**Consequence:** `sensor.global_alert_context` never reflects security threats.
Security is notified directly via `script.notify_security_event` (which works), but
there is no sustained alert entity that escalates and repeats if the threat persists.
Security alerts disappear immediately after the notification fires.

**Required interface:** `sensor.security_alert_context` (derived from
`sensor.security_threat_level`) → `alert.security_alert` → `sensor.alert_device_entities`.

---

### MI-03 [CRITICAL] Security does not consume the correct presence trust output

**Gap:** Security event classification uses trust model data that is always `false`.
The correct published output (`binary_sensor.low_trust_present`) exists but is never
consumed by security.

**Required wire:** Replace all security references from `input_boolean.low_trust_present`
and `input_boolean.staff_on_site` to `binary_sensor.low_trust_present` and
`binary_sensor.staff_on_site`.

---

### MI-04 [HIGH] `input_boolean.arrival_detected` never set by boundary resolver

**Gap:** `input_boolean.arrival_detected` is defined in `presence_helpers.yaml` and
referenced in security context docs, but the boundary resolver automation never sets it.

**Required wire:** After confirmed arrival in `presence_boundary_resolver`, add:
```yaml
- action: input_boolean.turn_on
  target:
    entity_id: input_boolean.arrival_detected
```
with a corresponding reset after cooldown.

---

### MI-05 [HIGH] Presence domain has no alert pipeline entry

**Gap:** `alerts_presence.yaml` is an empty stub (17 lines, no entities).
Unknown AP connections and occupancy anomalies do not enter the alert pipeline.

**Required interface:** `sensor.presence_alert_context` (derived from
`binary_sensor.unknown_unifi_ap_detected` or anomalous occupancy) → `alert.presence_alert`.

---

### MI-06 [MEDIUM] `binary_sensor.anyone_home` (Mobile App) not reconciled with AP detection

**Gap:** Two parallel home-detection mechanisms exist:
- `binary_sensor.anyone_connected_home` — AP-based (used by security)
- `binary_sensor.anyone_home` — Mobile App person.* (used by some lighting)

When they disagree (phone is home but not on WiFi, or on guest WiFi), security and
lighting make conflicting decisions. No reconciliation sensor exists.

**Required interface:** A unified `binary_sensor.anyone_probably_home` that combines
both signals (AP OR mobile app OR recent gate activity).

---

### MI-07 [MEDIUM] Power strategy not consumed by lighting for energy-save mode

**Gap:** `sensor.power_state` and `sensor.power_strategy` expose power status, but
the lighting domain does not consume these to reduce load during critical power events.

**Required wire:** Lighting automation conditions should check `sensor.power_state ≠ critical`
before activating non-essential scenes. This wiring is mentioned in power contract
architecture diagram but not implemented.

---

### MI-08 [LOW] `input_boolean.notifications_enabled` not in YAML

**Gap:** `water_notifications.yaml` gates on `input_boolean.notifications_enabled` which
was created via the HA UI (not in any YAML package). If the HA instance is restored from
config backup without entity registry, this helper is lost.

**Required wire:** Add to `packages/notifications/notifications_helpers.yaml`.

---

## Section 6: System Risk Assessment

### Single Entity Failure Impact

| Entity | If unavailable | Domains broken | Severity |
|--------|---------------|----------------|----------|
| `group.inverter_grid` | Water refill gate fails (always denied) | water | 🔴 HIGH |
| `sensor.inverter_battery_soc` | Water refill gate fails (SOC check) | water | 🔴 HIGH |
| `binary_sensor.anyone_connected_home` | Security always acts as "nobody home" | security | 🔴 HIGH |
| `sensor.water_state` | Water lifecycle halts — no pump control | water | 🔴 HIGH |
| `sensor.water_tank_depth_validated` | water_state → fault, pump stops | water | 🔴 HIGH |
| `sensor.global_alert_context` | Dashboard alert widget blind | alerts | 🟠 MEDIUM |
| `input_boolean.system_startup` | If stuck ON: all room occupancy suppressed | presence, lighting | 🟠 MEDIUM |
| `sensor.security_threat_level` | Security notifications stop routing | security, alerts | 🟠 MEDIUM |
| `input_boolean.notifications_quiet_hours` | Quiet hours ignored — night-time noise | notifications | 🟡 LOW |
| `sensor.alert_device_entities` | Alert aggregator blind | alerts, dashboard | 🟡 LOW |

### Most Fragile Shared Helpers

| Helper | Risk | Why |
|--------|------|-----|
| `input_boolean.notifications_enabled` | 🔴 HIGH | UI-created only, not in YAML. Lost on config restore |
| `input_boolean.system_startup` | 🟠 MEDIUM | If it gets stuck ON after a crash, all room occupancy is suppressed until next HA restart |
| `input_boolean.low_trust_present` | 🟠 MEDIUM | Never auto-set. Security and lighting behave as if staff is never present |
| `input_datetime.low_trust_start/end` | 🟠 MEDIUM | Orphaned in entity registry. `boundary_permissive_window` always false |
| `input_text.security_event_session` | 🟠 MEDIUM | 255-char limit. Known overflow risk in high-activity periods |

### Integration Failure Cascade Analysis

| Integration | Goes offline | Cascade |
|-------------|-------------|---------|
| **Solarman** | Inverter sensors → unavailable | Power sensors unknown → water refill gate fails (SOC unknown → denied) → refill stops even if tank is critical |
| **Tuya (water sensor)** | `water_tank_depth` → unavailable → `water_state` → fault → pump disabled | Water system fully stops |
| **UniFi Network** | All `device_tracker.*_iphone*` → disconnected | `anyone_connected_home` → off → security mode → away → security heightened + lighting departure triggers |
| **Hikvision NVR / go2rtc** | Camera motion sensors → unknown/off | Security no events — silent. No false alarms but no real detection |
| **Load Shedding integration** | `binary_sensor.load_shedding_active` unknown | Grid risk calculation wrong. No pre-outage notification |
| **OpenWeatherMap** | `weather.home` unknown (already Watchman-flagged) | Security visibility condition always evaluates unknown. No cascading but degraded classification |
| **Telegram Bot** | `notify.telegram_bot_5527` → ServiceNotFound | Startup: `continue_on_error` catches. Runtime: all alert notifications lose Telegram channel |

### Highest-Risk Failure Scenario

**Solarman + Tuya simultaneous failure** — both integrations share LocalTuya dependency.
If the Tuya cloud or local bridge fails:
- Water depth sensor → unavailable → water_state = fault → pump disabled
- Inverter Solarman may be unaffected (uses Modbus, not Tuya)

But if Solarman fails too:
- SOC unknown → water refill gate denies
- Power strategy unknown → no buy score decisions

The house can run out of water and battery with no alerts reaching the pipeline because:
- alerts_water.yaml is empty (MI-01)
- The notification path for water works, but there's no alert entity for sustained escalation

### Most Impactful Single Fix

Fixing **IV-01** (security reads wrong presence entities) is the single highest-leverage
fix in the system. It repairs:
- `sensor.security_trust_mode` — now correctly reflects staff presence
- `binary_sensor.security_low_trust_active` — now gates correctly
- `sensor.security_correlation` — service visits now classified correctly
- `sensor.security_event_classification` — service_person state now reachable
- `alerts_doors.yaml` trust mode check — fixed by cascade (IV-03)
- `sensor.security_lighting_intent` — lighting response to trust now correct
- `binary_sensor.boundary_permissive_window` — see IV-05 (separate fix, but related)

---

## Section 7: Recommended Execution Order for Fixes

Fixes are grouped by dependency chain. Each group can be done independently.
Within a group, do them in order.

### Group A — Presence Trust Model (Unblocks: security, lighting, alerts, boundary)

These fixes form a chain. Each one enables the next.

```
A1. Add input_datetime.low_trust_start/end back to context_presence.yaml
    (restores orphaned entities, unblocks A2)
    
A2. Fix boundary_permissive_window template to use restored datetimes
    (unblocks gate_open_too_long suppression during staff hours)
    
A3. Replace input_boolean.low_trust_present and input_boolean.staff_on_site
    in security_core.yaml with binary_sensor versions
    (IV-01 fix — single most impactful fix in the system)
    
A4. Replace input_boolean.low_trust_present in lighting_departure.yaml
    with binary_sensor.low_trust_present
    (IV-02 fix — same token change, different file)
```

**Why this order:** A1 must precede A2 (entity must exist first). A3 and A4 can be done
in parallel after A1/A2. A3 and A4 do not depend on A1/A2 — they can be done first
and will immediately improve trust classification even without the time-window fix.

---

### Group B — Alert Pipeline Gaps (Independent — can be done any order)

```
B1. Implement alerts_water.yaml
    - binary_sensor: water_state in [critical, safety, fault]
    - context sensor: severity + devices list (depth, pump state, duration)
    - alert entity: alert.water_alert
    - Wire into aggregator trigger list
    
B2. Implement alerts_security.yaml
    - binary_sensor: security_threat_level in [warning, critical]
    - context sensor: severity + devices list (threat score, classification, camera)
    - alert entity: alert.security_alert
    - Wire into aggregator trigger list
    
B3. Add alert.network_alert and alert.media_alert to aggregator trigger list
    (1-line fix each, eliminates 60s lag for these domains)
```

B1 and B2 are independent. B3 is a trivial 2-line fix with no dependencies.

---

### Group C — Notification Pipeline Cleanup (Independent — safe to batch)

```
C1. Add continue_on_error: true to Telegram call in notify_water_events.yaml
    (trivial 1-line fix)
    
C2. Fix counter reference in notify_water_events.yaml weekly summary
    counter.water_borehole_faults_week → counter.water_borehole_faults_this_week
    (trivial 1-word fix)
    
C3. Add input_boolean.notifications_enabled to notifications_helpers.yaml
    (prevents loss on config restore)
    
C4. Migrate temperature routing automations to script.notify_system_event
    (alerts_temperature.yaml — replace 8 direct notify calls)
    
C5. Migrate presence unknown AP automations to script.notify_presence_event
    (presence_notifications.yaml — replace 4 direct notify calls)
    
C6. Resolve device_power dual notification
    (choose: remove route automation direct calls OR remove alert entity)
```

C1, C2, C3 are trivial and safe to do in one pass. C4, C5, C6 are larger refactors.
C4 and C5 can be done in any order. C6 should be done after deciding which path to keep.

---

### Group D — Arrival Detection (Depends on: nothing, but low risk)

```
D1. Wire input_boolean.arrival_detected in presence_boundary_resolver
    (currently defined but never set)
    
D2. Remove presence_test_arrival automation from production
    (test artifact — sets last_arrival_time on every gate open, no conditions)
```

D1 and D2 are independent. D2 is a deletion — verify nothing else depends on the alias.

---

### Group E — Power Interface Cleanup (Independent)

```
E1. Replace sensor.inverter_1_battery in alerts_power.yaml battery low check
    with sensor.inverter_battery_soc
    (IV-04 fix — 1 entity name change)
```

---

### Group F — Presence / Home Detection Reconciliation (Requires design decision)

```
F1. Decide canonical home-detection entity (AP vs Mobile App vs combined)
F2. Implement binary_sensor.anyone_probably_home if combined approach chosen
F3. Audit all consumers and migrate to single entity
```

This requires a design decision first. Do not implement until F1 is resolved.

---

### Summary: Safe to Do Immediately (No Dependencies, No Design Decisions)

| Fix | File | Type | Effort |
|-----|------|------|--------|
| A3 | security_core.yaml, security_logic.yaml | entity rename (×4) | 10 min |
| A4 | lighting_departure.yaml | entity rename (×1) | 5 min |
| B3 | alerts_summary.yaml | add 2 trigger lines | 5 min |
| C1 | notify_water_events.yaml | add 1 line | 2 min |
| C2 | notify_water_events.yaml | rename 1 entity | 2 min |
| C3 | notifications_helpers.yaml | add 4 lines | 5 min |
| D2 | presence_boundary.yaml | delete automation | 5 min |
| E1 | alerts_power.yaml | rename 1 entity | 2 min |

These 8 fixes address 5 bugs, close 2 interface violations, and can all be validated
in a single HA config reload. Total estimated effort: ~36 minutes.

---

## Section 8: Architecture Violations (Non-Bug)

These are structural issues that don't break functionality today but violate the
intended architecture.

| Violation | Description | Priority |
|-----------|-------------|----------|
| Trust model in context/ | `context_presence.yaml` owns trust booleans and schedules — should be in `presence/` | Low |
| `power_helpers.yaml` layering | Contains both group: (helpers) and template: sensor: (templates) — should be split | Low |
| `automations.yaml` has 3,836 lines | No security automations in it (good), but legacy UI automations not yet migrated | Ongoing |
| Motion sensors are stubs | 7 motion binary_sensors hardcoded `state: "off"` — confidence layer is AP-only until real sensors added | Planned |
| PRESENCE_CONTEXT.md outdated | Documents presence_templates.yaml, presence_state.yaml, presence_automations.yaml — none exist | Docs |
| ALERTS_CONTEXT.md outdated | Documents alerts_core.yaml, alerts_device.yaml — neither exists | Docs |
| Lighting bugs (9 open) | BUG-L01–L09 — see LIGHTING_CONTRACT.md Section 7 for full list. HIGH: L01 (scene_night_away missing entrance lights), L02 (missing main_entrance off), L03 (wrong presence entity in arrival). | See Group L in PROJECT_STATE.md |
| WATER_CONTEXT.md outdated | Lists 8 files vs actual 18. water_automations.yaml, water_notifications.yaml don't exist | Docs |

---

## Appendix: Bug Cross-Reference

All bugs from domain contracts, consolidated with system context added.

| ID | Domain | Severity | One-Line Summary | Blocked By | Unblocks |
|----|--------|----------|-----------------|------------|----------|
| BUG-P01 (Presence) | Presence | Critical | `input_boolean.low_trust_present` never auto-set | Nothing | IV-01, IV-02 |
| BUG-P02 (Presence) | Presence | High | `input_boolean.staff_on_site` never auto-set | Nothing | IV-01 |
| BUG-P03 (Presence) | Presence | High | `boundary_permissive_window` always false (orphaned datetimes) | A1 | Permissive gate |
| BUG-P04 (Presence) | Presence | High | Intruder automation conditions commented out | Nothing | Security alerts |
| BUG-P05 (Presence) | Presence | High | `holiday_mode` produces `intruder_high` state, no automation handles it | Nothing | Security |
| BUG-P06 (Presence) | Presence | Medium | `presence_test_arrival` production test artifact | Nothing | Clean start |
| BUG-P07 (Presence) | Presence | Medium | `input_boolean.arrival_detected` never set | Nothing | Security wiring |
| BUG-P08 (Presence) | Presence | Medium | device_tracker entity name inconsistency (tracker vs no-tracker suffix) | Nothing | AP detection |
| BUG-A01 (Alerts) | Alerts | Critical | `alerts_water.yaml` empty stub | Nothing | Water in global context |
| BUG-A02 (Alerts) | Alerts | High | `alerts_security.yaml` empty stub | Nothing | Security in global context |
| BUG-A03 (Alerts) | Alerts | High | Temperature bypass of notification pipeline | Nothing | Quiet hours for temp |
| BUG-A04 (Alerts) | Alerts | High | Device power dual notification | Nothing | Clean delivery |
| BUG-A05 (Alerts) | Alerts | Medium | network_alert / media_alert missing from aggregator trigger | Nothing | Real-time network alerts |
| BUG-A06 (Alerts) | Alerts | Medium | Battery low / prepaid drift not in power context devices list | Nothing | Richer alert UI |
| BUG-N01 (Notif) | Notifications | Medium | Missing continue_on_error on Telegram in notify_water_events | Nothing | Startup reliability |
| BUG-N02 (Notif) | Notifications | Medium | Wrong counter entity in water weekly summary | Nothing | Correct fault count |
| BUG-N03 (Notif) | Notifications | High | `notifications_enabled` not in YAML | Nothing | Config restore safety |
| BUG-N04 (Notif) | Notifications | High | Temperature bypass of notification pipeline (duplicate of BUG-A03) | Nothing | Quiet hours for temp |
| BUG-N05 (Notif) | Notifications | Medium | Presence unknown AP bypass of notification pipeline | Nothing | Quiet hours |
| BUG-N06 (Notif) | Notifications | Medium | Device power dual notification (duplicate of BUG-A04) | Nothing | Clean delivery |
| BUG-N07 (Notif) | Notifications | Low | Water tank "full" state trigger never fires | Nothing | Dead code removal |
| BUG-W01 (Water) | Water | High | `water_policy_helpers.yaml` thresholds orphaned (unused) | Nothing | Clean threshold model |
| BUG-W02 (Water) | Water | Medium | Refill permission gate simpler than WATER_CONTEXT specifies | Nothing | Docs alignment |
| BUG-W03 (Water) | Water | Medium | Depth rate derived from raw sensor (not validated) | Nothing | Rate accuracy |
| BUG-Pow01 (Power) | Power | Medium | Duplicate unique_id: solar_surplus_available (sensor + binary_sensor) | Nothing | HA startup warning |
| SEC-BUG-01 (Security) | Security | High | `sensor.security_correlation` has no unique_id | Nothing | Entity registry stability |
| SEC-BUG-02 (Security) | Security | High | `input_text` session storage 255-char overflow risk | Nothing | Event logging stability |
| SEC-BUG-03 (Security) | Security | Medium | cam03_rear_perimeter does not exist | Nothing | Phantom zone |
| SEC-BUG-04 (Security) | Security | Medium | Two overlapping snapshot automations | Nothing | Duplicate snapshots |

---

*Generated from: SECURITY_CONTRACT.md, POWER_CONTRACT.md, WATER_CONTRACT.md,*  
*PRESENCE_CONTRACT.md, ALERTS_CONTRACT.md, NOTIFICATIONS_CONTRACT.md*  
*Last updated: 2026-04-13*
