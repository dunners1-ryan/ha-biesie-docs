# HABiesie — Water Domain Contract
> **Living contract.** Produced by deep audit on 2026-04-13.  
> **CRITICAL:** Read `packages/water/a_water_lifecycle_contract.yaml` before modifying ANY water file.  
> This document supersedes `WATER_CONTEXT.md` — keep both in sync.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [File Inventory](#2-file-inventory)
3. [Entity Reference](#3-entity-reference)
4. [Data Flow Maps](#4-data-flow-maps)
5. [Cross-Domain Interface](#5-cross-domain-interface)
6. [Known Issues](#6-known-issues)
7. [Error Signatures (Watchman-Confirmed)](#7-error-signatures-watchman-confirmed)
8. [Optimization Recommendations](#8-optimization-recommendations)
9. [Implementation Checklist](#9-implementation-checklist)
10. [Lifecycle Contract Verification](#10-lifecycle-contract-verification)
11. [Safety System Audit](#11-safety-system-audit)
12. [Sensor Reliability Assessment](#12-sensor-reliability-assessment)

---

## 1. System Overview

### Design Philosophy

```
Truth = physical water level movement, NOT switch state
```

The water system is a **deterministic refill lifecycle engine** that observes physical reality (tank depth sensor) to classify, audit, and control all water entering the tank. The lifecycle contract (`a_water_lifecycle_contract.yaml`) defines hard non-negotiable rules.

### Lifecycle State Machine

```
                  ┌─────────────────────────────────────┐
                  │  water_tank_refill_control.yaml      │
                  │  (decides WHEN to turn pump on)      │
                  └──────────────┬──────────────────────┘
                                 │ turns pump ON
                                 ▼
┌──────────────────────────────────────────────────────────────┐
│                    PUMP STATE: ON                            │
│                                                              │
│  water_capture_refill_start fires → sets:                   │
│    water_refill_cycle_active = ON                           │
│    water_refill_start_depth (RAW)                           │
│    water_refill_start (timestamp)                           │
│    clears: water_refill_aborted_due_to_safety               │
│                                                              │
│  water_refill_visibility_guard fires (2 min delay):         │
│    if depth rose → sets water_refill_manual_run = ON        │
│    (only if no managed cycle active)                        │
└──────────────────────────┬───────────────────────────────────┘
                           │
              ┌────────────┼────────────────────┐
              │            │                    │
              ▼            ▼                    ▼
        NORMAL STOP   SAFETY ABORT        WATER OK STOP
        (pump→off)    (protection fires)  (control sees ok)
              │            │                    │
              └────────────┴────────────────────┘
                           │
                           ▼
            water_capture_refill_end fires → sets:
              water_refill_end (timestamp)
              water_refill_end_depth (VALIDATED)
              water_refill_cycle_active = OFF
```

### Refill Permission Gate

```
binary_sensor.water_refill_allowed = ON when:
  group.inverter_grid = "on"                                       ← grid must be on
  AND sensor.battery_soc >= input_number.water_battery_soc_sufficient
  AND input_boolean.load_control_borehole_enabled = "on"
  AND (
    now_hour < last_sun_slot_start_hour                            ← before last-sun window
    OR battery_soc >= sensor.last_sun_soc_target                   ← OR SOC already at target
    OR sensor.water_tank_level <= 20                               ← OR tank critically low (override)
  )
  AND sensor.energy_orchestrator_state NOT IN                      ← orchestrator gate (added E1 2026-06-14)
      ['critical', 'loadshedding', 'loadshedding_critical']

Last-sun-slot gate added 2026-05-25: during the last ~2h of peak sun (default 14:00 onwards),
borehole is blocked unless SOC has reached the overnight charge target (80% summer / 90% winter).

Orchestrator gate added 2026-06-14 (Session E1): when the energy orchestrator enters critical,
loadshedding, or loadshedding_critical state, normal refills are blocked. The 'conserve' and
'surplus' states do NOT block refill — the existing SOC floor already handles conserve, and
surplus means solar is ample. Emergency branches (Branch 1 safety, Branch 2 critical+grid,
Branch 3 critical+limited) in water_tank_refill_control.yaml bypass this sensor entirely and
are unaffected by the orchestrator gate.
Tank-critical override (≤20%) bypasses this gate so essential water supply is never blocked.

⚠️ IMPORTANT — Branch 2 (critical + grid on) does NOT use this gate (fixed 2026-06-14):
Branch 2 directly checks `group.inverter_grid = on` + `load_control_borehole_enabled = on`.
SOC is irrelevant when water is critical and grid is available — the grid covers the pump load.
The permission gate's 50% SOC floor was previously blocking critical refills during low-SOC
mornings even with grid on, violating the contract table below.

NOTE: This is simpler than WATER_CONTEXT.md specifies — see Issue 7.
```

### Refill Decision Matrix

| Water State | Grid | Battery SOC | Outcome |
|---|---|---|---|
| `low` | on | any | Refill (solar window required) |
| `low` | off | ≥ sufficient | Refill (solar window required) |
| `low` | off | < sufficient | Wait |
| `critical` | on | any | Full refill (unlimited) |
| `critical` | off | ≥ sufficient | Full refill (unlimited) |
| `critical` | off | < hard_stop | 10-min survival refill (emergency) |
| `safety` | any | any | Refill (bypasses permission gate) |
| `ok` | any | any | Stop pump if running |

---

## 2. File Inventory

**Package path:** `packages/water/` — 18 files (WATER_CONTEXT.md listed 8 — outdated)

| File | Role | Layer |
|---|---|---|
| `a_water_lifecycle_contract.yaml` | Specification document — LOCKED hard rules | Contract |
| `water_helpers.yaml` | Lifecycle flags, timestamps, depth thresholds, day demand selectors (7 input_select) | Helpers |
| `water_policy_helpers.yaml` | Policy-driven threshold helpers (full/low/critical/empty) — ORPHANED, unused | Helpers |
| `water_sensor.yaml` | Derivative depth rate sensor (platform: derivative) | Core |
| `water_templates.yaml` | Depth validation, tank level %, refill allowed, solar window, demand planning sensors | Templates |
| `water_state_extensions.yaml` | Derived cycle state mirror binary_sensor | State |
| `water_refill_cycle.yaml` | Cycle state sensor, summary sensor, avg flow rate | State |
| `water_refill_capture.yaml` | Automations that write start/end timestamps and depths | Automations |
| `water_tank_refill_control.yaml` | Main control: demand-based pump on/off; water_stop_at_daily_target; mid-run shutdown | Automations |
| `water_safety.yaml` | Hard stops: max depth (1.95m) + battery SOC floor (water_safety_battery_hard_stop) | Safety |
| `water_protection_automations.yaml` | No-rise protection, spike logging | Safety |
| `water_fault_logging.yaml` | Centralised fault logger (queued) | Logging |
| `water_health.yaml` | Sensor health binary_sensors (healthy/stable) | Health |
| `water_consume_cycle.yaml` | Consumption rate sensor (from depth rate) | Analytics |
| `water_maintenance_automations.yaml` | Counter/flag resets (daily/weekly) | Maintenance |
| `water_scripts.yaml` | Manual pump scripts (5/10/15/30/60 min) + season demand preset scripts | Scripts |
| `water_reporting.yaml` | Weekly summary automation, avg flow sensors | Reporting |
| `water_test_helpers.yaml` | Reset script for test/debug cycle state | Debug |

### File Mapping vs Context Document

The context document (`WATER_CONTEXT.md`) lists 8 files including `water_automations.yaml`, `water_core.yaml`, `water_notifications.yaml`, and `water_debug.yaml`. None of these exist in the actual package. The actual split is more granular — 18 files with purpose-specific separation.

---

## 3. Entity Reference

### Core State

```
sensor.water_state                           fault/critical/safety/low/refilling/ok
sensor.water_tank_depth                      m  raw Tuya pass-through (no filtering)
sensor.water_tank_depth_validated            m  spike-filtered (trigger-based template)
sensor.water_tank_level_percent              %  = (validated_depth / 1.95) * 100
sensor.water_tank_depth_rate                 m/h  derivative of RAW depth (not validated)
sensor.water_tank_consumption_rate           m/h  depth rate when negative (consuming)
```

### Demand Planning Sensors (added 2026-05-25)

```
sensor.water_target_depth_tomorrow           m  — stop depth for tonight's refill based on tomorrow's
                                                  input_select demand type and mapped input_number target
  attributes:
    demand_type: Normal | Wash/Clean | Irrigation | Pool
    tomorrow_day: Mon | Tue | Wed | Thu | Fri | Sat | Sun

sensor.water_demand_today                    string — today's demand type (dashboard display only)

sensor.water_refill_blocked_reason           "none" OR human-readable reason string
                                             Compares against sensor.water_target_depth_tomorrow
                                             (was: water_target_depth_normal — updated 2026-05-25)
```

### Demand Planning Helpers (added 2026-05-25)

```
input_select.water_demand_monday   )
input_select.water_demand_tuesday  )  Options: Normal | Wash/Clean | Irrigation | Pool
input_select.water_demand_wednesday)  Summer defaults: Mon=Wash/Clean Tue=Normal Wed=Irrigation
input_select.water_demand_thursday )    Thu=Wash/Clean Fri=Irrigation Sat=Pool Sun=Irrigation
input_select.water_demand_friday   )  Winter defaults: Mon=Wash/Clean Tue=Normal Wed=Irrigation
input_select.water_demand_saturday )    Thu=Wash/Clean Fri=Normal Sat=Normal Sun=Irrigation
input_select.water_demand_sunday   )

# Season preset scripts — bulk-set all 7 selectors in one tap:
script.water_demand_set_summer_profile
script.water_demand_set_winter_profile

# REMOVED 2026-05-25 (were unused — no logic ever read them):
# input_boolean.irrigation_full_monday … irrigation_full_sunday      (7 removed)
# input_boolean.irrigation_partial_monday … irrigation_partial_sunday (7 removed)
# input_boolean.washing_heavy_monday, washing_heavy_thursday          (2 removed)
# input_boolean.house_cleaning_monday, house_cleaning_thursday        (2 removed)
# input_boolean.washing_partial_friday/saturday/sunday                (3 removed)
```

### Lifecycle Sensors

```
sensor.water_refill_cycle                    idle/refilling/completed
sensor.water_refill_cycle_summary            active/aborted/completed (with attributes)
  attributes:
    start_depth_m, end_depth_m, delta_depth_m
    start_time, end_time, duration_seconds, duration_minutes
    avg_flow_m_per_h, aborted_due_to_safety, manual_run
sensor.water_refill_duration_seconds         s   (end - start timestamp diff)
sensor.water_refill_duration_minutes         min
sensor.water_refill_duration_friendly        HH:MM:SS
sensor.water_refill_duration_display         HH:MM:SS (with full unavailable guard)
sensor.water_refill_last_outcome             Completed Normally / Manual Refill / Aborted (Safety)
sensor.water_refill_cycle_avg_flow_rate      m/h  post-cycle analytics
sensor.water_refill_abort_reason             safety_limit/none
sensor.water_refill_start_depth_display      m  (reads from cycle summary attribute)
sensor.water_refill_end_depth_display        m  (reads from cycle summary attribute)
sensor.water_refill_delta_depth_display      m
sensor.water_refill_outcome_display              Aborted (Safety) / Completed Normally / —
```

### Lifecycle Flags

```
input_boolean.water_refill_cycle_active          ON during managed refill
input_boolean.water_refill_manual_run            ON when manual (unmanaged) pump run detected
input_boolean.water_refill_aborted_due_to_safety ON after any safety abort
input_boolean.water_tank_refill_enabled          Master enable/disable switch
input_boolean.water_emergency_acknowledged       Emergency event acknowledged
input_boolean.water_refill_force_override        Force-start refill bypassing solar window, SOC,
                                                 and last-sun-slot gates. Safety hard-stops
                                                 (battery floor 40%, max depth 1.95m) still apply.
                                                 Turn OFF manually on dashboard after use.
                                                 Added 2026-06-14 (water_helpers.yaml).
```

### Binary Sensors

```
binary_sensor.water_tank_refilling               pump on AND depth < 1.95 (NOT safety-aware)
binary_sensor.water_tank_refilling_derived       mirror of water_refill_cycle_active flag
binary_sensor.water_refill_allowed               grid on + SOC >= minimum (SIMPLIFIED — see Issue 7)
binary_sensor.water_solar_window_active          current time within solar start/stop window
binary_sensor.water_tank_full_depth              depth >= 1.95 (fixed threshold, not policy-driven)
binary_sensor.water_tank_sensor_healthy          no impossible depth jumps (> 0.25m delta)
binary_sensor.water_tank_sensor_stable           alias of sensor_healthy
binary_sensor.water_tank_depth_sensor_stable     |raw - validated| < 0.15
```

### Timestamps and Boundaries

```
input_datetime.water_refill_start           cycle start time
input_datetime.water_refill_end             cycle end time
input_datetime.water_last_emergency_refill  last emergency mode start
input_datetime.water_refill_solar_start     solar window start (time only)
input_datetime.water_refill_solar_stop      solar window stop (time only)
input_number.water_refill_start_depth       depth at start (written by BOTH control + capture)
input_number.water_refill_end_depth         depth at end (written by capture)
input_number.water_refill_start_level       % at start (UI/legacy only — NOT source of truth)
input_number.water_refill_end_level         % at end (UI/legacy only)
```

### Thresholds

```
# From water_helpers.yaml (updated 2026-05-25 — live values set via dashboard):
input_number.water_depth_critical           m  (live: 0.25) — emergency refill trigger
input_number.water_depth_low                m  (live: 0.80) — "low" display state threshold
input_number.water_depth_minimum_safety     m  (live: 0.35) — safety state floor
input_number.water_battery_soc_sufficient   %  (initial: 50)  — min SOC to start refill
input_number.water_battery_soc_hard_stop    %  (initial: 40)  — absolute SOC floor (temp; lower to 20% after new batteries)
input_number.water_refill_max_runtime_minutes min — DEFINED BUT UNUSED (Issue 5)
input_number.water_target_depth_normal      m  (initial: 1.00) — stop depth for Normal days
input_number.water_target_depth_partial     m  (initial: 1.20) — stop depth for Wash/Clean days (entity_id unchanged)
input_number.water_target_depth_irrigation  m  (initial: 1.25) — stop depth for Irrigation days (NEW)
input_number.water_target_depth_full        m  (initial: 1.60) — stop depth for Pool days (entity_id unchanged)

# Day demand selectors (NEW 2026-05-25):
input_select.water_demand_monday … water_demand_sunday
  Options: Normal | Wash/Clean | Irrigation | Pool
  Summer defaults: Mon=Wash/Clean Tue=Normal Wed=Irrigation Thu=Wash/Clean Fri=Irrigation Sat=Pool Sun=Irrigation
  Winter defaults: Mon=Wash/Clean Tue=Normal Wed=Irrigation Thu=Wash/Clean Fri=Normal Sat=Normal Sun=Irrigation
  Preset scripts: script.water_demand_set_summer_profile / water_demand_set_winter_profile

# Derived sensor (NEW 2026-05-25):
sensor.water_target_depth_tomorrow — reads tomorrow's input_select, maps to depth input_number
sensor.water_demand_today          — today's demand type string (dashboard display)

# From water_policy_helpers.yaml:
input_number.water_depth_full_threshold     m  (initial: 1.98) — POLICY driven, NOT used in templates
input_number.water_depth_low_threshold      m  (initial: 1.00) — POLICY driven, NOT used in templates
input_number.water_depth_critical_threshold m  (initial: 0.50) — POLICY driven, NOT used in templates
input_number.water_depth_empty_threshold    m  (initial: 0.10) — POLICY driven, NOT used in templates
```

**CRITICAL NOTE:** `water_policy_helpers.yaml` defines a second set of depth thresholds (`*_threshold`) that are NOT used by any template or automation. The actual templates hardcode values or use `water_helpers.yaml` thresholds. The policy helpers are orphaned.

### Counters

```
counter.water_borehole_faults_today          daily fault counter (reset at 00:00)
counter.water_borehole_faults_this_week      weekly fault counter (reset Monday)
counter.water_refill_aborted_today           daily abort counter (reset daily)
counter.water_refill_aborted_this_week       weekly abort counter (reset Sunday)
input_number.water_emergency_runs_today      emergency run count (reset daily)
input_text.borehole_last_fault               text description of last fault
```

### Pumps and Hardware

```
switch.borehole_pump                 primary refill pump
switch.water_pressure_pump           house pressure pump (separate system, not in water package)
sensor.borehole_pump_power           W — used for dry-run detection (dry-run protection DISABLED)
sensor.water_tank_level_sensor_depth          m  raw Tuya hardware sensor
sensor.water_tank_level_sensor_liquid_level   %  raw Tuya hardware sensor (used in notifications)
```

### Reporting

```
sensor.water_refill_avg_flow_this_week   m/h (reads from MISSING sensor.water_refill_flow_rate)
sensor.water_refill_avg_flow_last_week   m/h (reads from MISSING sensor.water_refill_flow_rate)
```

---

## 4. Data Flow Maps

### Depth Truth Chain

```
Hardware: Tuya depth sensor
    │
    ▼
sensor.water_tank_level_sensor_depth (RAW — never filtered, never smoothed)
    │
    ├──→ sensor.water_tank_depth  (template passthrough, float(-1))
    │        │
    │        ├──→ sensor.water_tank_depth_validated  (spike-filtered, trigger-based)
    │        │        │
    │        │        ├──→ sensor.water_state  (state classification)
    │        │        ├──→ binary_sensor.water_refill_allowed (SOC gate)
    │        │        ├──→ binary_sensor.water_tank_full_depth
    │        │        ├──→ binary_sensor.water_tank_refilling  (uses < 1.95)
    │        │        ├──→ water_safety.yaml  (hard stop at > 1.95)
    │        │        ├──→ water_protection_automations.yaml (no-rise check)
    │        │        └──→ input_number.water_refill_end_depth (written at cycle end)
    │        │
    │        └──→ sensor.water_tank_depth_spike_delta  (debug: cur - validated)
    │
    └──→ sensor.water_tank_depth_rate (DERIVATIVE of RAW — spikes affect rate)
             │
             └──→ sensor.water_tank_consumption_rate (negative rate abs)
                  sensor.water_borehole_no_rise_protection (uses depth_rate)
```

### Cycle Start Sequence (Race Condition)

```
Trigger: water_state = low/critical/safety + conditions met
    │
    ▼ water_tank_refill_control.yaml executes:
    1. Writes input_number.water_refill_start_depth ← VALIDATED depth  ← WRITE 1
    2. Writes input_datetime.water_refill_start ← now()
    3. Turns on switch.borehole_pump
    │
    ▼ Pump transitions off→on (10s stability window):
    4. water_capture_refill_start fires:
       Writes input_number.water_refill_start_depth ← RAW depth  ← WRITE 2 (OVERWRITES)
       Writes input_datetime.water_refill_start ← now() (duplicate write)
       Sets input_boolean.water_refill_cycle_active = ON
       Clears water_refill_aborted_due_to_safety

⚠️ WRITE 2 overwrites WRITE 1 — start_depth ends up as raw sensor value, not validated.
⚠️ Both writes to input_datetime.water_refill_start — minor race, same timestamp.
```

### Cycle End Sequence

```
Pump turns off (any reason) → stays off for 15s:
    │
    ▼ water_capture_refill_end fires (condition: cycle_active = ON):
    1. Writes input_datetime.water_refill_end ← now()
    2. Writes input_number.water_refill_end_depth ← VALIDATED depth
    3. Sets input_boolean.water_refill_cycle_active = OFF
```

### Notification Flow (Water)

```
Any water event → script.notify_water_event
    (defined in packages/notifications/, not in water package)

Critical events also call notify.telegram_bot_5527 directly in water_tank_refill_control.yaml.
The weekly summary in water_reporting.yaml also calls telegram directly.
```

---

## 5. Cross-Domain Interface

### Water Domain ← Inputs Consumed

| External Entity | Source Domain | Used In | Notes |
|---|---|---|---|
| `sensor.battery_soc` | Power (`sensor.power_battery_soc`) | `water_templates.yaml`, `water_tank_refill_control.yaml` | Name-derived alias of "Battery SOC" template |
| `sensor.inverter_battery_soc` | Power | (NOT used directly — goes via `sensor.battery_soc`) | |
| `group.inverter_grid` | Power | `water_templates.yaml` (refill_allowed), control | On = grid available |
| `sensor.energy_orchestrator_state` | Power (`energy_state.yaml`) | `water_templates.yaml` (refill_allowed) | Gates refill when state in [critical, loadshedding, loadshedding_critical] — added E1 2026-06-14 |
| `script.notify_water_event` | Notifications | All water notification calls | Central routing |
| `notify.telegram_bot_5527` | Core | `water_tank_refill_control.yaml`, `water_reporting.yaml` | Direct Telegram (bypasses routing) |
| `input_boolean.notifications_enabled` | Notifications | `water_notifications.yaml` | Quiet hours gate |

### Water Domain → Outputs Consumed by Other Domains

| Entity | Consumer Domain | Notes |
|---|---|---|
| `switch.borehole_pump` | Power (energy tracking) | Switch state tracked in load groups |
| `sensor.water_state` | Alerts | Water alert routing |
| `binary_sensor.water_tank_refilling` | Dashboard | Visual indicator |
| `sensor.water_tank_depth_validated` | Dashboard | UI display |

### Daily Usage Analytics (water_reporting.yaml — added E7 2026-06-14)

```
sensor.water_tank_consumption_integral   m  — cumulative integral of water_tank_consumption_rate
                                              (platform: integration, method: left, unit_time: h)
                                              Resets on HA restart. Source for water_usage_today.

sensor.water_usage_today                 m  — daily-reset accumulation of depth consumed.
                                              utility_meter daily cycle on water_tank_consumption_integral.
                                              Resets to 0 at midnight. ⚠️ Requires HA restart.

sensor.water_daily_usage_mean            m  — 7-day rolling mean of water_usage_today samples.
                                              platform: statistics, state_characteristic: mean, 7d/100 samples.
                                              NOTE: underestimates true daily mean by ~50% (statistics mean
                                              of a 0→peak daily ramp). Multiply by ~2 for calibrated daily
                                              average, or set predictive fill threshold accordingly.
                                              Accurate after ≥7 days of history.
```

### Effective Fill Target Sensor (water_templates.yaml — added E7 2026-06-14)

```
sensor.water_effective_fill_target       m  — dynamic pump stop depth.
  Normal:           returns sensor.water_target_depth_tomorrow (demand-plan depth)
  Predictive fill:  returns input_number.water_target_depth_full (1.6m) when
                    water_predictive_fill_enabled=ON AND orchestrator='conserve'
  attributes:
    source: demand_target | predictive_fill_full
  Read by: water_stop_at_daily_target (above: trigger), Branch 4.7 log message
```

### Borehole Control Status Sensor (water_state_extensions.yaml — added E6 2026-06-14)

`sensor.borehole_control_status` — 9-state priority display mirroring pool/geyser pattern:

| Priority | State | Condition |
|---|---|---|
| 1 | `Disabled` | `load_control_borehole_enabled` off |
| 2 | `Emergency fill — bypassing gates` | pump on AND water_state critical/safety (emergency bypass path — water_refill_allowed not consulted) |
| 3 | `Blocked — <orch> state` | orchestrator in [critical, loadshedding, loadshedding_critical] |
| 4 | `Blocked — grid off, SOC X% < Y%` | grid off AND SOC below water_battery_soc_sufficient |
| 5 | `Filling (X%)` | pump on AND refill cycle active (normal managed fill) |
| 6 | `Tank full (X%)` | water_state ok AND level ≥ 90% |
| 7 | `Waiting — outside solar window` | solar window inactive |
| 8 | `Ready to fill (X%)` | water_refill_allowed on AND water_state not ok |
| 9 | `Monitoring (X%)` | default |

### Predictive Fill Helpers (water_helpers.yaml — added E6 2026-06-14)

```
input_boolean.water_predictive_fill_enabled          ← default: false — enable after ≥7d of water_usage_today history
input_number.water_predictive_fill_threshold_percent ← default: 50% — fill when level drops below this AND conserve mode
input_number.water_max_fill_hours_per_day            ← default: 2h — defined; NOT yet wired into automation (future V2)
```

**Wired in E7 (2026-06-14)** — Branch 4.7 added to `water_tank_refill_control.yaml`. Fires when:
- `water_predictive_fill_enabled = on`
- `energy_orchestrator_state == 'conserve'`
- `water_tank_level < water_predictive_fill_threshold_percent`
- `water_tank_depth_validated < water_effective_fill_target` (avoids immediate stop)
- NOT critical/safety state
- `water_solar_window_active = on` AND `water_refill_allowed = on`

**Stop target**: `sensor.water_effective_fill_target` — returns `water_target_depth_full` (1.6m) in conserve+predictive mode; fills tank to maximum buffer capacity instead of demand-plan depth.

**Threshold calibration**: Default 50% (0.975m) is BELOW all current demand targets (≥1.0m Normal). This makes Branch 4.7 rarely fire independently of Branch 4. For genuine proactive pre-fill when tank is comfortable but solar is poor, **set threshold to 70-80%** via Helpers dashboard. Branch 4 handles fills when tank < demand target; Branch 4.7 handles fills when tank is above demand target but below the 70-80% buffer threshold during conserve mode.

**`water_max_fill_hours_per_day`** (daily pump runtime cap) remains unimplemented — deferred to V2 after data establishes typical fill duration.

### Telegram Direct Bypass

`water_tank_refill_control.yaml` calls `notify.send_message` directly on `notify.telegram_bot_5527` for critical/emergency refills in addition to calling `script.notify_water_event`. This means emergency events send DUPLICATE notifications — once via the central script and once directly. This violates the CODING_STANDARDS requirement to use central scripts only.

---

## 6. Known Issues

Issues ordered by risk/impact. **P1 = breaks functionality. P2 = incorrect data/behaviour. P3 = cleanup.**

---

### Issue 1 — MISSING: `sensor.water_refill_flow_rate` (breaks weekly reporting)
**Priority:** P1 — Weekly reporting sensors always return 0  
**File:** `packages/water/water_reporting.yaml:23,29`  
**Root cause:** `sensor.water_refill_avg_flow_this_week` and `sensor.water_refill_avg_flow_last_week` both read from `state_attr('sensor.water_refill_flow_rate','mean')`. This entity is never defined anywhere — it was intended to be a statistics platform sensor tracking `sensor.water_refill_cycle_avg_flow_rate` but was never implemented.  
**Watchman:** Confirmed missing.  
**Fix:** Add to `water_reporting.yaml`:
```yaml
sensor:
  - platform: statistics
    name: Water Refill Flow Rate
    unique_id: water_refill_flow_rate
    entity_id: sensor.water_refill_cycle_avg_flow_rate
    state_characteristic: mean
    max_age:
      days: 7
    sampling_size: 20
```

---

### Issue 2 — BUG: `water_tank_full_notification` never fires
**Priority:** P1 — Tank full notification silently dead  
**File:** `packages/notifications/water_notifications.yaml:69`  
**Root cause:** Trigger watches `sensor.water_state` → `"full"` but the sensor never produces that state. Actual states are: `fault`, `critical`, `safety`, `low`, `refilling`, `ok`. There is no `"full"` state.  
**Fix:** Change trigger state to watch for `binary_sensor.water_tank_full_depth` transitioning to `"on"`, OR add a `"full"` state branch to `sensor.water_state` at depth ≥ `water_target_depth_full`.

---

### Issue 3 — BUG: Double write to `water_refill_start_depth` — capture overwrites control snapshot
**Priority:** P2 — Start depth recorded as RAW value, not validated  
**Files:** `water_tank_refill_control.yaml` (writes validated) + `water_refill_capture.yaml:62` (overwrites with raw)  
**Root cause:** Control automation writes `water_refill_start_depth` (validated) before starting pump. Then pump on → triggers `water_capture_refill_start` which overwrites the same field with the RAW sensor value (`sensor.water_tank_level_sensor_depth`). Final value is raw, not validated.  
**Contract says:** "Depth values: RAW sensor (Tuya)" — contract v2 line 112 — so this MAY be intentional but the control automation writing validated first suggests the intent has drifted.  
**Fix options:**
1. Remove the start_depth write from `water_tank_refill_control.yaml` — let capture own it exclusively
2. Change `water_refill_capture.yaml:62` to use validated depth: `states('sensor.water_tank_depth_validated')`

---

### Issue 4 — BUG: `water_block_refill_when_not_allowed` has no `from:` constraint
**Priority:** P2 — Fires on Tuya reconnect glitches (unavailable→on) | ✅ FIXED 2026-04-15  
**File:** `packages/water/water_tank_refill_control.yaml`  
**Root cause:** Trigger `to: "on"` with no `from: "off"` caused false safety aborts on Tuya reconnect.  
**Fix:** `from: "off"` already present on this trigger (was pre-existing). Additional trigger integrity fixes applied as part of Group E:
- `water_borehole_no_rise_protection`: added `from: "off"` to pump ON trigger (E1)
- `water_reconcile_cycle_state`: added `from: "on"` to pump OFF trigger (E1)
- `water_refill_visibility_guard`: added `for: "00:00:10"` stability window to pump ON trigger (E2)

---

### Issue 5 — SAFETY GAP: Max runtime cutoff not implemented
**Priority:** P2 — Protection 5 from contract is missing  
**Files:** All water package files — not present in any  
**Root cause:** `input_number.water_refill_max_runtime_minutes` is defined in `water_helpers.yaml` but never referenced in any automation. The safety spec lists "Max runtime cutoff" as Protection 5, but there is no automation that uses this helper.  
**Exception:** The emergency survival mode (case 3) has a hardcoded 10-minute delay. This is NOT the max runtime cutoff — it is a specific emergency duration, not a general safety guard.  
**Fix:** Add to `water_safety.yaml`:
```yaml
- id: water_safety_max_runtime_cutoff
  alias: "Water Safety ▸ Max Runtime Cutoff"
  mode: restart
  trigger:
    - trigger: state
      entity_id: switch.borehole_pump
      from: "off"
      to: "on"
      for:
        minutes: "{{ states('input_number.water_refill_max_runtime_minutes') | int(90) }}"
  condition:
    - condition: state
      entity_id: switch.borehole_pump
      state: "on"
  action:
    - action: switch.turn_off
      target:
        entity_id: switch.borehole_pump
    - action: input_boolean.turn_on
      target:
        entity_id: input_boolean.water_refill_aborted_due_to_safety
    - action: logbook.log
      data:
        name: Water Safety – Max Runtime
        message: >
          Pump stopped: exceeded max runtime of
          {{ states('input_number.water_refill_max_runtime_minutes') }} minutes.
    - action: script.notify_water_event
      data:
        severity: warning
        message: "[Code: water_max_runtime] Pump stopped after max runtime exceeded."
```

---

### Issue 6 — ✅ RESOLVED 2026-05-25: Full mid-run shutdown coverage implemented
**Priority:** P2 — now fully covered  
**Files:** `water_tank_refill_control.yaml`, `water_safety.yaml`

**Fix 1 — `water_borehole_mid_run_shutdown` automation (policy stops):**
- Triggers: `binary_sensor.water_refill_allowed → off` (catches grid offline, SOC drop, last-sun-slot, load control) AND `binary_sensor.water_solar_window_active → off` (end of solar day)
- Stops pump immediately in both cases. Exempt: `water_state = safety` (emergency bypass).
- Does NOT set `water_refill_aborted_due_to_safety` — these are policy stops, not hardware faults.
- Notifies via `script.notify_water_event` with reason (permission_revoked or solar_window_closed).

**Fix 2 — `water_safety_battery_hard_stop` automation (hardware floor, `water_safety.yaml`):**
- Triggers: `sensor.battery_soc` below `input_number.water_battery_soc_hard_stop`
- No SAFETY-state exemption — this IS the absolute floor even for emergency refills
- DOES set `water_refill_aborted_due_to_safety` (fault stop, not policy stop)
- `water_battery_soc_hard_stop` initial = **40%** (temporary; current battery cells unreliable below 40%). Lower to 20% after new battery installation end of week 2026-05-25.

**Coverage summary:**
| Scenario | Covered by |
|---|---|
| Grid goes offline mid-run (normal state) | `water_refill_allowed → off` → mid_run_shutdown |
| SOC drops below 50% mid-run (normal state) | `water_refill_allowed → off` → mid_run_shutdown |
| Last-sun-slot starts at 14:00 mid-run | `water_refill_allowed → off` → mid_run_shutdown |
| Solar window closes end-of-day | `water_solar_window_active → off` → mid_run_shutdown |
| SOC crashes below hard-stop (40%) — any state incl. SAFETY | `water_safety_battery_hard_stop` |

---

### Issue 7 — ✅ FIXED 2026-06-19: `binary_sensor.water_refill_allowed` now includes master switch check
**Priority:** P2 — Missing checks allow refill in conditions the spec prohibits  
**File:** `packages/water/water_templates.yaml`  
**Resolution:** `is_state('input_boolean.water_tank_refill_enabled', 'on')` added as the first
condition in `binary_sensor.water_refill_allowed`. The gate now correctly returns false when the
master switch is off, matching what any external consumer (dashboard, future automations) would expect.
Solar window and safety abort state remain as separate binary_sensors by design.

---

### Issue 8 — BUG: `counter.water_borehole_faults_week` wrong entity name in notifications
**Priority:** P2 — Weekly summary reports 0 faults regardless of actual count  
**File:** `packages/notifications/water_notifications.yaml:154`  
**Root cause:** References `counter.water_borehole_faults_week` but the actual entity is `counter.water_borehole_faults_this_week`.  
**Watchman:** Confirmed missing.  
**Fix:** Change `faults_week: "{{ states('counter.water_borehole_faults_week') }}"` → `"{{ states('counter.water_borehole_faults_this_week') }}"`.

---

### Issue 9 — BUG: `water_safety.yaml` max-depth stop has commented-out debounce
**Priority:** P2 — Tuya spike > 1.95m triggers false safety abort  
**File:** `packages/water/water_safety.yaml:40`  
**Root cause:** `# for: "00:01:00" #REQUIRED DEBOUNCE (ONE LINE)` — the comment says debounce is REQUIRED but it's commented out. Without it, a single erroneous Tuya reading above 1.95m (before the spike filter catches it) immediately stops the pump and marks the cycle as aborted.  
**Note:** The spike filter in `water_tank_depth_validated` should reject jumps > 0.35m. But since the safety trigger watches `sensor.water_tank_depth_validated` (not raw), a legitimate slow rise to 1.96m would correctly trigger. The risk is specifically a spike that passes through the filter (a jump < 0.35m from near-full).  
**Fix:** Uncomment the `for: "00:01:00"` line. One minute debounce is appropriate for a physical safety stop.

---

### Issue 10 — DUAL WRITE: Duplicate notifications on emergency refills
**Priority:** P3 — User receives 2 notifications per emergency refill  
**File:** `packages/water/water_tank_refill_control.yaml:200,221`  
**Root cause:** Both CRITICAL cases (2 and 3) call `script.notify_water_event` AND `notify.send_message` on `notify.telegram_bot_5527` directly. Central script routes to Telegram anyway. Result: 2 Telegram messages per emergency event.  
**Violates:** CODING_STANDARDS notification standard (all calls through central script).  
**Fix:** Remove the direct `notify.send_message` calls. Add `telegram: true` parameter to the central script call, or let the central script handle Telegram routing based on severity.

---

### Issue 11 — ORPHANED: `water_policy_helpers.yaml` threshold entities unused
**Priority:** P3 — Dead code creates confusion about which thresholds are active  
**File:** `packages/water/water_policy_helpers.yaml`  
**Root cause:** Defines `water_depth_full_threshold`, `water_depth_low_threshold`, `water_depth_critical_threshold`, `water_depth_empty_threshold`. None of these are referenced in any template or automation. The active thresholds are in `water_helpers.yaml` (`water_depth_critical`, `water_depth_low`, etc.).  
**Fix:** Either wire the policy helpers into the templates (replacing hardcoded values) — which was the intent — or delete this file entirely.

---

### Issue 12 — DEAD CODE: `water_refill_never_reached_full` effectively unreachable
**Priority:** P3 — The automation can never fire in practice  
**File:** `packages/notifications/water_notifications.yaml:101`  
**Root cause:** Trigger: `binary_sensor.water_tank_refilling` stays on for 3 hours. But `water_borehole_no_rise_protection` fires after 15 minutes of no depth increase and stops the pump. The tank refilling binary_sensor turns off → 3-hour timer resets. No refill that fails to rise will ever stay on for 3 hours. Only a genuine 3-hour non-stop rise (tank filling very slowly) could trigger this — but that would mean normal operation.  
**Fix:** Replace with a trigger-based approach: when cycle starts, check 30–45 minutes later if the depth has risen by a meaningful amount.

---

### Issue 13 — RISK: `water_reconcile_cycle_state` clears active flag on 30s glitch
**Priority:** P3 — Tuya connectivity drop mid-refill resets cycle state  
**File:** `packages/water/water_tank_refill_control.yaml:585`  
**Root cause:** If Tuya connectivity drops while pump is running, switch state becomes `unavailable`. After 30 seconds of `unavailable`... actually, the trigger is `to: "off"` with `for: "00:00:30"` — `unavailable` is not `"off"`, so this may be OK. But if the Tuya device briefly reports `"off"` before recovering, the cycle flag is cleared. When pump is reported back on, capture fires again with a new start timestamp, doubling the apparent cycle.  
**Mitigation:** The `for: "00:00:30"` window provides some protection against very brief glitches.

---

### Issue 14 — ✅ FIXED 2026-06-14: `water_block_refill_when_not_allowed` had no safety-state exemption
**Priority:** P1 — Safety-state refills looped endlessly (start → immediate block → repeat every 5 min)  
**File:** `packages/water/water_tank_refill_control.yaml`  
**Root cause:** When `water_state = safety` (depth 0.25–0.35m), Branch 1 turns the pump on without checking `water_refill_allowed`. But `water_block_refill_when_not_allowed` had no safety-state exemption — it fired on any pump turn-on when `water_refill_allowed = off`, killing the pump immediately. `water_borehole_mid_run_shutdown` already had the safety exemption; the block automation did not. Inconsistency.  
**Symptom:** Pump tried to start multiple times, blocked each time, refill never actually ran.  
**Fix:** Added two NOT-conditions to `water_block_refill_when_not_allowed`:
- NOT (water_state = safety) — matches existing mid_run_shutdown exemption
- NOT (water_state = critical AND group.inverter_grid = on) — see Issue 15

---

### Issue 15 — ✅ FIXED 2026-06-14: Branch 2 (critical+grid) incorrectly required SOC ≥ 50%
**Priority:** P1 — Critical water refills silently failed when battery was 40–50% and grid was on  
**File:** `packages/water/water_tank_refill_control.yaml`  
**Root cause:** Contract table says `critical + grid on + any SOC → full refill`. Branch 2 required `binary_sensor.water_refill_allowed = on`, which has a hard 50% SOC floor (`water_battery_soc_sufficient`). The tank-critical override (≤20% tank level) in `water_refill_allowed` only bypasses the last-sun-slot gate, NOT the base SOC floor. Result: critical water + grid on + SOC at e.g. 45% → `water_refill_allowed = off` → Branch 2 fails → Branch 3 requires grid off → nothing fires.  
**Symptom:** Tank at 2% fill this morning — still not running. User had to manually lower `water_battery_soc_sufficient` to force it.  
**Fix:**
1. Branch 2 now directly checks `group.inverter_grid = on` + `load_control_borehole_enabled = on` instead of `water_refill_allowed`. SOC not required when grid is on for a critical refill.
2. `water_block_refill_when_not_allowed` exempts `critical + grid on` starts (so block doesn't kill the pump Branch 2 just started).

---

## 7. Error Signatures (Watchman-Confirmed)

| Entity | Status | File | Issue |
|---|---|---|---|
| `binary_sensor.borehole_pump_dry_run` | missing | dashboard | Intentionally disabled (commented out) |
| `automation.water_refill_cycle_*` | missing | dashboard | Renamed/restructured — dashboard needs update |
| `automation.water_borehole_dry_*` | missing | dashboard | Intentionally disabled |
| `automation.water_borehole_max_*` | missing | dashboard | Not implemented (Issue 5) |
| `counter.water_borehole_faults_week` | missing | water_notifications.yaml:154 | Wrong name — should be `_this_week` (Issue 8) |
| `sensor.water_refill_flow_rate` | missing | water_reporting.yaml:23,29 | Never defined (Issue 1) |
| `sensor.water_tank_level_percent` | missing | water_reporting.yaml:54 | Entity ID mismatch (name truncation) |

**Note on `sensor.water_tank_level_percent`:** Defined with name "Water Tank Level %" (unique_id: `water_tank_level_percent`). HA generates entity_id from name stripping `%` → may produce `sensor.water_tank_level_` which doesn't match what reporting.yaml calls. Verify actual entity_id in HA UI.

---

## 8. Optimization Recommendations

### Recommendation 1 — Unify start_depth ownership (Issues 3)
Control automation and capture automation both write to `water_refill_start_depth`. Assign ownership to ONE of them. The capture automation should own this (it's the designated data recorder per the contract). Remove the pre-pump write from `water_tank_refill_control.yaml`.

### Recommendation 2 — Wire `water_policy_helpers.yaml` into templates
The policy helpers were designed to replace hardcoded thresholds. Either complete the wiring:
- `water_templates.yaml` `water_state` sensor should use `input_number.water_depth_critical_threshold` instead of `input_number.water_depth_critical`
- OR delete `water_policy_helpers.yaml` to eliminate confusion

There are two competing threshold systems right now. Pick one.

### Recommendation 3 — Add explicit `"full"` state to `sensor.water_state`
Currently `water_state` goes from `refilling` to `ok` as the pump stops. There is no `"full"` state. Adding one would:
- Fix the tank full notification (Issue 2)
- Make dashboards clearer
- Align `sensor.water_state` with the lifecycle spec

```yaml
{% elif d >= states('input_number.water_target_depth_full') | float(1.85) %}
  full
{% elif is_state('switch.borehole_pump','on') %}
  refilling
```

### Recommendation 4 — Formalize state machine as an explicit sensor
The lifecycle contract implies Idle → Running → Completed/Aborted. Currently these states are spread across `sensor.water_refill_cycle` and `input_boolean.water_refill_cycle_active`. A single `sensor.water_lifecycle_state` (idle/running/completed/aborted) would make dashboards and automations more reliable.

### Recommendation 5 — Rate-limit spike notifications
`water_protection_automations.yaml` fires a notification every time a spike > 1.0m is rejected. If the Tuya sensor enters a glitch loop, this produces a notification storm. Add a `rate_limit_minutes: 30` or use a `delay` + `mode: single` to suppress repeat spikes within a window.

---

## 9. Implementation Checklist

### Sprint 1 — Fix Safety Gaps (Issues 5 + 6)

- [ ] Implement max runtime cutoff automation in `water_safety.yaml` using `input_number.water_refill_max_runtime_minutes`
- [ ] Implement battery hard stop reactive automation in `water_safety.yaml` monitoring `sensor.battery_soc`

### Sprint 2 — Fix Trigger Integrity (Issues 4 + 9)

- [✅] Add `from: "off"` to pump ON triggers — `water_borehole_no_rise_protection` fixed 2026-04-15 (E1)
- [✅] Add `from: "on"` to pump OFF triggers — `water_reconcile_cycle_state` fixed 2026-04-15 (E1)
- [✅] Add stability window to `water_refill_visibility_guard` pump ON trigger — `for: "00:00:10"` added 2026-04-15 (E2)
- [✅] Audit all `to: "on"` pump triggers — complete. `water_block_refill_when_not_allowed` already had `from: "off"`. `water_capture_refill_start` already had all constraints.
- [ ] Uncomment `for: "00:01:00"` debounce in `water_stop_refill_at_max_depth` (optional — safety tradeoff)
- [✅] E3 verified: `binary_sensor.water_refill_allowed` checked before pump start on critical+low paths. Safety/limited-critical paths intentionally bypass (emergency scenarios).

### Sprint 3 — Fix Data Integrity (Issues 1 + 2 + 3 + 8)

- [ ] Add `sensor.water_refill_flow_rate` statistics sensor to `water_reporting.yaml`
- [ ] Fix `water_tank_full_notification` — change trigger from `water_state = "full"` to `binary_sensor.water_tank_full_depth = on` OR add `"full"` state to `water_state`
- [ ] Fix double-write to `water_refill_start_depth` — remove write from control automation
- [ ] Fix `counter.water_borehole_faults_week` → `counter.water_borehole_faults_this_week` in `water_notifications.yaml`

### Sprint 4 — Design Clean-up (Issues 7 + 10 + 11)

- [✅] Update `binary_sensor.water_refill_allowed` to include `water_tank_refill_enabled` check — fixed 2026-06-19
- [ ] Remove direct Telegram calls from `water_tank_refill_control.yaml` (use central script only)
- [ ] Decide: wire `water_policy_helpers.yaml` into templates OR delete file

---

## 10. Lifecycle Contract Verification

Each rule from `a_water_lifecycle_contract.yaml` verified against actual code.

### Hard Rules (v1)

| Rule | Statement | Status | Evidence |
|---|---|---|---|
| 1 | Pump ON + depth rising → classify as managed or manual | **PASS** | `water_refill_visibility_guard` tags unmanaged rises as manual |
| 2 | No refill without classification | **PARTIAL** | Managed cycles tagged by capture. Manual guard has 2-min delay — brief runs may be unclassified |
| 3 | Safety stops set `water_refill_aborted_due_to_safety` | **PARTIAL** | Max depth stop ✅, no-rise ✅, block-when-not-allowed ✅. Battery hard stop: **NOT IMPLEMENTED** (Issue 6). Max runtime: **NOT IMPLEMENTED** (Issue 5) |
| 4 | Safety overrides have absolute authority | **PARTIAL** — see Safety Audit | Hard stops work for implemented protections. Max runtime and battery reactive stop are MISSING |
| 5 | Outcomes are exclusive: completed OR aborted | **PASS** | `water_refill_aborted_due_to_safety` is a flag; completion is implied by its absence |
| 6 | Manual refills not treated as faults by default | **PASS** | `water_refill_manual_run` flag set, no fault counter incremented |

### Hard Rules (v2)

| Rule | Statement | Status | Evidence |
|---|---|---|---|
| 1 | Start captured only by sustained depth rise | **FAIL** | Start is triggered by pump state change (off→on 10s), not by depth rise |
| 2 | End captured ONLY if cycle is ACTIVE | **PASS** | `water_capture_refill_end` conditions on `water_refill_cycle_active = on` |
| 3 | END timestamps never precede START | **PASS** | Duration sensor clamps to 0 if end ≤ start |
| 4 | Manual runs explicitly tagged | **PASS** | Visibility guard sets manual_run flag |
| 5 | Safety stops ALWAYS mark aborted_due_to_safety | **PARTIAL** | 3 of 5 protections implemented (Issue 5, 6) |
| 6 | Cleanup clears ACTIVE and MANUAL flags, never safety flags | **PASS** | Capture end clears cycle_active; reconcile clears cycle_active. Safety flag cleared only on new cycle START |

### Contract Drift

The contract states: "Start = physical event — pump ON and depth increasing." The current capture automation (`water_capture_refill_start`) triggers on pump state change (off→on, 10s stability), NOT on depth rise. This means a pump start that does not result in depth increase (dry borehole, closed valve) still creates a "start" record. The depth-rise trigger was commented out at line 26 with comment "removed as have safety for this." This is a deliberate design decision but violates Contract v2 Rule 1.

---

## 11. Safety System Audit

Five protections listed in `WATER_CONTEXT.md`. Each audited independently.

### Protection 1: Max Depth Stop
- **Spec:** Depth ≥ `water_target_depth_full` → Stop pump  
- **Implementation:** `water_safety.yaml` — `water_stop_refill_at_max_depth`
- **Trigger:** `sensor.water_tank_depth_validated` above **1.95** (hardcoded, not policy-driven)
- **Threshold used:** Hardcoded 1.95, NOT `input_number.water_target_depth_full` (initial: 1.85) and NOT `input_number.water_depth_full_threshold` (initial: 1.98)
- **Independence:** ✅ Separate file, separate automation, does not check any refill flags
- **Bypass possible?** Via spike filter — upward spikes > 0.35m are rejected by validated sensor; a genuine depth of 1.96m would trigger correctly
- **Issue:** No debounce (Issue 9) — single spike above 1.95m triggers false abort
- **Verdict:** IMPLEMENTED but with false-abort risk from spike interaction. Threshold mismatch: `water_target_depth_full` = 1.85m but stop fires at 1.95m.
- **2026-05-06:** `water_refill_aborted_due_to_safety` no longer set by this automation. Max depth = successful completion, not a fault. Lifecycle now shows "Completed (Full)" instead of "Aborted (Safety)". Notification changed to "Tank Full" title with clear success message.

### Protection 2: Dry Run Protection
- **Spec:** Pump ON + power below threshold → Stop pump
- **Implementation:** DISABLED — `binary_sensor.borehole_pump_dry_run` and `water_borehole_dry_run_shutdown` are both commented out
- **Reason given:** "turned this off and use rise protection"
- **Fallback:** No-rise protection (Protection 3) covers this after 15 minutes
- **Risk:** 15 minutes of dry-running a borehole pump can cause hardware damage
- **Verdict:** ❌ NOT IMPLEMENTED — replaced by Protection 3. 15-minute window is aggressive for hardware protection. Recommend reinstating with a shorter window (2-3 minutes) alongside no-rise protection.

### Protection 3: No-Rise Protection (Replaces Dry Run)
- **Spec:** Pump running + depth not increasing after timeout → Stop pump
- **Implementation:** `water_protection_automations.yaml` — `water_borehole_no_rise_protection`
- **Trigger:** Pump on for 15 minutes + depth rate < 0.01 m/h + depth < 1.95
- **Independence:** ✅ Separate condition checks; marks safety abort flag
- **Issue:** Uses `sensor.water_tank_depth_rate` which is a derivative of the RAW sensor. If a raw sensor spike occurs within the 15-minute window, the derivative shows a positive rate, resetting the effective timer. A spike could therefore mask a genuine dry-run.
- **Verdict:** IMPLEMENTED. Works correctly for genuine no-rise conditions. Spike sensitivity is a minor risk.

### Protection 4: Battery Hard Stop
- **Spec:** SOC drops below `water_battery_soc_hard_stop` → Stop pump  
- **Implementation:** ✅ `water_safety.yaml` — `water_safety_battery_hard_stop` (added 2026-05-25)
- **Trigger:** `sensor.battery_soc` below `input_number.water_battery_soc_hard_stop`
- **Threshold:** 40% (temporary — current battery cells unreliable below 40%). Lower to 20% after new battery installation end of week 2026-05-25.
- **Coverage:** No SAFETY-state exemption — applies even to emergency refills (unlike `water_borehole_mid_run_shutdown`). Sets `water_refill_aborted_due_to_safety`.
- **Verdict:** ✅ IMPLEMENTED. See Issue 6 for full coverage details.

### Protection 5: Max Runtime Cutoff
- **Spec:** Pump on > `water_refill_max_runtime_minutes` → Stop pump  
- **Implementation:** ❌ NOT IMPLEMENTED  
- **What exists:** `input_number.water_refill_max_runtime_minutes` is defined (unused). Emergency case 3 has a hardcoded 10-minute limit but only for that mode.  
- **Verdict:** ❌ MISSING — see Issue 5. Input helper exists but no automation uses it.

### Safety System Summary

| Protection | Status | Risk Level |
|---|---|---|
| Max depth stop | ✅ Implemented (minor spike risk) | Low |
| Dry run (power-based) | ❌ Disabled | Medium — 15min exposure |
| No-rise protection | ✅ Implemented | Low (minor spike sensitivity) |
| Battery hard stop (reactive) | ✅ Implemented 2026-05-25 — `water_safety_battery_hard_stop` in `water_safety.yaml` | Low — floor set at 40% (temp; lower to 20% after new battery install) |
| Max runtime cutoff | ❌ Missing | Medium — runaway possible |

**One of five protections missing.** Battery hard stop now implemented. Max runtime cutoff (Issue 5) is the remaining gap.

---

## 12. Sensor Reliability Assessment

### Tuya Depth Sensor

| Property | Detail |
|---|---|
| Entity | `sensor.water_tank_level_sensor_depth` (raw), `sensor.water_tank_depth_validated` (filtered) |
| Type | Tuya-based ultrasonic distance sensor |
| Mounting height | ~2.08 m (above tank bottom) |
| Max liquid depth | ~2.05 m (physical) |
| Target full depth | 1.95 m (safety stop threshold) |
| Known behaviours | Occasional large upward spikes (reported > 1.0m delta); connectivity drops causing unavailable transitions |

### Spike Rejection Logic

Implemented in `water_templates.yaml` as a trigger-based template sensor (`sensor.water_tank_depth_validated`):

```
Rules:
1. If current reading < 0: keep previous value
2. If no valid previous value: trust current
3. Always allow large DROPS (cur < prev AND delta > 0.35m): trust current
   Reason: tank can empty quickly; drops are physical
4. Allow large RISES only when pump is running: trust current
   Reason: pump can fill fast enough to cause large genuine rises
5. Reject upward spikes: if (cur - prev) > 0.35m AND pump is off: keep previous
6. All other changes: trust current (delta ≤ 0.35m in either direction)
```

**Spike threshold: 0.35 m**

### Spike Rejection Effectiveness

| Scenario | Handled? | Evidence |
|---|---|---|
| Large upward spike (> 0.35m) without pump | ✅ Rejected | Rule 5 |
| Large upward spike during pump run | ⚠️ NOT rejected — trusts current | Rule 4 |
| Unavailable → reading transition | ✅ Handled (Rule 1 or 2) | |
| Gradual sensor drift | ❌ Not handled | No trend detection |
| Multiple small spikes accumulating | ❌ Not handled | Each passes 0.35m test |
| Large downward spike (sensor detach, dry tank) | ✅ Allowed through | Rule 3 — intentional |

**Key weakness:** During pump operation, ALL readings are trusted regardless of magnitude (Rule 4). A spike of any size is accepted when the pump is running. If the Tuya sensor glitches while the pump is filling, the validated sensor will accept the wrong value. This could cause:
- False early safety stop (if spike > 1.95m while filling)
- Incorrect end depth recorded

### Sensor Trustworthiness Rating

| Dimension | Rating | Notes |
|---|---|---|
| Normal operation (no pump) | ✅ Good | Spike rejection works well |
| During pump run | ⚠️ Moderate | All spikes accepted; glitch risk |
| After connectivity drop | ✅ Good | Falls back to previous value |
| As a control trigger | ⚠️ Moderate | Spike could cause false abort during fill |
| As an audit record | ⚠️ Moderate | Start depth written from RAW (Issue 3) |
| Long-term drift | ❓ Unknown | No drift detection implemented |

### Recommendation

The Tuya sensor is adequate for coarse level monitoring and scheduling decisions. It is NOT adequate as a sole safety arbiter without the debounce on the max-depth stop (Issue 9). The no-rise protection provides a practical second line of defence.

For improved reliability, consider adding a `for: "00:00:30"` delay to the no-rise condition trigger to prevent rapid reconnect/disconnect cycles from producing confusing depth rates.

---

## Locked Design Decisions

| Decision | Rationale |
|---|---|
| Truth = depth movement, NOT switch state | Pump can run without water entering tank; switch state is unreliable signal |
| Contract spec file (`a_water_lifecycle_contract.yaml`) read-only | Must not be modified; changes require full re-audit |
| `water_refill_cycle_active` is the official managed cycle flag | All capture logic keys off this; do not add parallel flags |
| Safety abort flag NOT cleared by cleanup — only by new cycle start | Preserves forensic record of last outcome for dashboards |
| Depth rate uses RAW sensor (not validated) | Spike filter introduces latency; rate sensor needs real-time data |
| No-rise protection replaces dry-run protection | Deliberate trade-off; dry-run binary_sensor commented out intentionally |
| Manual scripts do NOT check `water_refill_allowed` | Manual operator actions override the permission gate by design |
| Emergency Telegram calls are direct (bypassing central script) | Critical emergency notifications need guaranteed delivery; accepted exception |

---

*Last updated: 2026-06-14 (E7)*  
*Updated by: E7 — sensor.water_usage_today (utility_meter), sensor.water_tank_consumption_integral (integration), sensor.water_daily_usage_mean (statistics), sensor.water_effective_fill_target (template). Branch 4.7 predictive fill added. water_stop_at_daily_target updated to read water_effective_fill_target. Predictive fill enabled/wired — see Predictive Fill Helpers section above.*
