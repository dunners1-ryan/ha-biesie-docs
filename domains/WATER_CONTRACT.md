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
  AND (                                                            ← conserve gate (added 2026-06-21)
    sensor.energy_orchestrator_state != 'conserve'
    OR sensor.water_tank_level <= 20                               ← tank-critical override still applies
  )

Last-sun-slot gate added 2026-05-25: during the last ~2h of peak sun (default 14:00 onwards),
borehole is blocked unless SOC has reached the overnight charge target (80% summer / 90% winter).

Orchestrator gate added 2026-06-14 (Session E1): when the energy orchestrator enters critical,
loadshedding, or loadshedding_critical state, normal refills are blocked. Emergency branches
(Branch 1 safety, Branch 2 critical+grid, Branch 3 critical+limited) in
water_tank_refill_control.yaml bypass this sensor entirely and are unaffected by the
orchestrator gate.

**Conserve gate (added 2026-06-21):** Previously 'conserve' and 'surplus' did NOT block
refill — the existing 50% SOC floor was assumed to already cover conserve days. Incident
2026-06-21: a poor-solar winter day (Solcast forecast_today 6.9 kWh vs ~30-50 kWh normal,
solar_weather_correlation = degraded) with dishwasher/washing machine running most of the day
left SOC at 45% — above the 50% floor in absolute terms wasn't even reached, but the user
flagged that borehole shouldn't run on days like this "unless really necessary." Conserve now
also blocks refill, UNLESS `sensor.water_tank_level <= 20` (tank-critical override — water
supply itself never gets blocked). 'surplus' is unaffected (ample solar = fine to run).
New blocked-reason: `water_refill_blocked_reason` priority 3b — "Conserving battery — poor
solar/high load today, tank not yet critical". New attribute on `water_refill_allowed`:
`conserve_blocked` (bool, diagnostic).
Tank-critical override (≤20%) bypasses both the orchestrator gate and the conserve gate so
essential water supply is never blocked.

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

**Package path:** `packages/water/` — 17 files (18 before `water_policy_helpers.yaml` was
deleted 2026-08-21, Issue 11; WATER_CONTEXT.md listed 8 — outdated)

| File | Role | Layer |
|---|---|---|
| `a_water_lifecycle_contract.yaml` | Specification document — LOCKED hard rules | Contract |
| `water_helpers.yaml` | Lifecycle flags, timestamps, depth thresholds, day demand selectors (7 input_select), degraded-rise-rate thresholds (added 2026-08-21) | Helpers |
| ~~`water_policy_helpers.yaml`~~ | **Deleted 2026-08-21** (Issue 11) — orphaned duplicate threshold set, zero references anywhere, superseded by nothing (the real thresholds already lived in `water_helpers.yaml`) | — |
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
sensor.water_refill_last_outcome             Refilling… (added 2026-08-18, Issue 17) / Completed Normally / Manual Refill / Aborted (Safety)
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
                                                 Midnight auto-clear added 2026-06-29
                                                 (water_refill_force_override_midnight_clear,
                                                 water_tank_refill_control.yaml) — no longer
                                                 relies solely on manual clear.
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
input_number.water_refill_start_depth       depth at start (written by capture only — control's duplicate write removed, Issue 3, 2026-07-08)
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

# Degraded-rise-rate protection (NEW 2026-08-21, WATER_CONTRACT Issue 5 — replaces the
# old unused water_refill_max_runtime_minutes helper, deleted same day):
input_number.water_refill_degraded_rate_threshold  m/h (initial: 0.05) — "healthy" rise floor
input_number.water_refill_degraded_rate_minutes    min (initial: 60)   — sustained duration before stop
```

**Resolved 2026-08-21:** `water_policy_helpers.yaml` (the second, orphaned set of depth
thresholds this note used to warn about) has been deleted — see the File Inventory table
above and Issue 11.

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
sensor.water_refill_avg_flow_this_week   m/h (reads sensor.water_refill_flow_rate, rolling 7-day mean — Issue 1, fixed 2026-07-08)
sensor.water_refill_avg_flow_last_week   m/h (reads input_number.water_refill_avg_flow_last_week, frozen Monday 00:00 — Issue 1, fixed 2026-07-08)
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
| `sensor.water_state` | Dashboard | Drives "% full" status card text/color on Home Overview + `water-control` (recalibrated 2026-07-17 — see note below) |
| `binary_sensor.water_tank_refilling` | Dashboard | Visual indicator |
| `sensor.water_tank_depth_validated` | Dashboard | UI display |

**Dashboard % full card recalibration (2026-07-17)**: The `sensor.water_tank_level` status card on both `.storage/lovelace.dashboard_overview` and `.storage/lovelace.dashboard_operations` (`water-control` page) previously re-derived its own bucket cutoffs (`pct<15` → "VERY LOW WATER", `pct<30` → "Low Water Level", `pct≥60` → green) independently of `sensor.water_state`'s actual classification — a second, drift-prone threshold system duplicating the real one (same class of problem as Recommendation 2 / the orphaned `water_policy_helpers.yaml` thresholds). Converted to % of 1.95m max depth, the real thresholds don't even line up with the card's numbers: `water_depth_critical` (0.25m) = 12.8%, `water_depth_minimum_safety` (0.35m) = 17.9%, `water_depth_low` (0.80m) = 41.0% — none of which are 15/30/60. Fixed by making the card read `sensor.water_state` directly (single source of truth, already driven by the same live input_numbers) instead of re-computing its own buckets — text/color now automatically track whatever those thresholds are tuned to. Also added a `fault`/`unavailable`/`unknown` branch (grey) that didn't exist before, so a Tuya sensor dropout no longer silently renders as green "System Normal". Both dashboard files are `.storage`-only (gitignored, not tracked by `gitupdate.sh`) — per CODING_STANDARDS.md, requires a full HA restart to take effect; do not open the dashboard UI editor before restarting or an autosave from the stale in-memory copy will revert it.

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
input_number.water_predictive_fill_threshold_percent ← YAML initial: 75% (recalibrated 2026-07-17, was 50%)
                                                         ⚠️ LIVE value still 50% as of 2026-07-17 — YAML `initial:`
                                                         only seeds a helper on first creation, it does not retroactively
                                                         update an already-existing input_number. Bump the Helpers
                                                         dashboard slider to 75% to actually apply the recalibration.
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

**UPDATE 2026-07-17**: `water_helpers.yaml` `initial:` recalibrated 50 → 75 per this recommendation. Live entity value unchanged (still 50%) until manually bumped on the dashboard — see caveat above.

**`water_max_fill_hours_per_day`** (daily pump runtime cap) remains unimplemented — deferred to V2 after data establishes typical fill duration.

### Telegram Direct Bypass

`water_tank_refill_control.yaml` calls `notify.send_message` directly on `notify.telegram_bot_5527` for critical/emergency refills in addition to calling `script.notify_water_event`. This means emergency events send DUPLICATE notifications — once via the central script and once directly. This violates the CODING_STANDARDS requirement to use central scripts only.

---

## 6. Known Issues

Issues ordered by risk/impact. **P1 = breaks functionality. P2 = incorrect data/behaviour. P3 = cleanup.**

---

### ~~Issue 1~~ — MISSING: `sensor.water_refill_flow_rate` (breaks weekly reporting)
**Priority:** P1 — Weekly reporting sensors always return 0
**Status:** ✅ FIXED 2026-07-08
**File:** `packages/water/water_reporting.yaml`

**Was:** `sensor.water_refill_avg_flow_this_week` and `sensor.water_refill_avg_flow_last_week` both read from `state_attr('sensor.water_refill_flow_rate','mean')`/`'mean_7d'`. Neither the entity nor a `mean_7d` attribute ever existed — the statistics platform doesn't expose a `mean_7d` characteristic at all, so "last week" was never fixable by just adding the missing sensor alone.

**Fix applied:**
1. Added `sensor.water_refill_flow_rate` (`platform: statistics`, source `sensor.water_refill_cycle_avg_flow_rate`, `state_characteristic: mean`, `max_age: days: 7`) to `water_reporting.yaml` — this is a rolling 7-day window, not a calendar week, and now correctly backs "this week".
2. "Last week" doesn't use a statistics attribute at all — added `input_number.water_refill_avg_flow_last_week` (`water_helpers.yaml`) plus a new automation `water_snapshot_weekly_avg_flow` (`water_maintenance_automations.yaml`, Monday 00:00 — same cadence as the existing `water_reset_weekly_fault_counter`) that freezes the current "this week" value into the input_number before the new week starts accumulating. `sensor.water_refill_avg_flow_last_week` now reads that input_number instead of the nonexistent `mean_7d` attribute.

---

### Issue 2 — ✅ CONFIRMED FIXED (doc-drift correction) 2026-08-21: `water_tank_full_notification` never fires
**Priority:** was P1 — Tank full notification silently dead
**File:** `packages/notifications/water_notifications.yaml`
**Status:** Re-verified live 2026-08-21 — this contract's own description was stale. The
automation already triggers on `binary_sensor.water_tank_full_depth` going `off`→`on` (the
exact fix this issue used to recommend), with an explicit code comment noting
`sensor.water_state` never reaches `"full"`. No code change needed this session; only the
contract text was out of date.

---

### ~~Issue 3~~ — BUG: Double write to `water_refill_start_depth` — capture overwrites control snapshot
**Priority:** P2 — Start depth recorded as RAW value, not validated
**Status:** ✅ FIXED 2026-07-08 (Fix option 1)
**Files:** `water_tank_refill_control.yaml` (was writing validated) + `water_refill_capture.yaml` (owns raw write)

**Was:** Control automation wrote `water_refill_start_depth` (validated) before starting the
pump, in all 6 of its refill branches (Safety/Critical/Emergency/Demand-target/Force-override/
Predictive-fill). Then pump-on → `water_capture_refill_start` fired ~10s later and overwrote
the same field with the RAW sensor value. Final value was raw, but the race meant the
validated write was pure waste and obscured which automation was the real source of truth.

**Resolved per `a_water_lifecycle_contract.yaml`'s own explicit invariant** ("SOURCE OF
TRUTH: Depth values: RAW sensor (Tuya). Validation: analytics-only (never capture).") — this
makes Fix option 1 the contractually-correct one, not just the simpler one. `water_capture_
refill_start` already had a commented-out validated-depth alternative next to its raw write,
confirming raw was already the deliberate choice there.

**Fix applied:** Removed the `input_number.set_value` write to `water_refill_start_depth` from
all 6 branches in `water_tank_refill_control.yaml`, each replaced with a one-line comment
pointing at `water_capture_refill_start` as sole owner — mirrors the same file's existing
"Mark cycle active ----> let capture control this" pattern already used for `water_refill_
cycle_active`. `water_refill_capture.yaml` untouched (was already correct per the contract).

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

### Issue 5 — ✅ FIXED 2026-08-21 (redesigned, not the originally-drafted fix): degraded-rise-rate protection
**Priority:** was P2, "Max runtime cutoff not implemented"
**Files:** `packages/water/water_protection_automations.yaml`, `packages/water/water_helpers.yaml`
**Original root cause:** `input_number.water_refill_max_runtime_minutes` was defined in `water_helpers.yaml` but never referenced in any automation. The safety spec listed "Max runtime cutoff" as Protection 5, but there was no automation using this helper.
**2026-08-21 re-assessment (user question: is this arbitrary given the existing depth
hard-stop + no-rise protection?):** Not fully redundant — real gap confirmed. The 1.95m
depth hard-stop (`water_stop_refill_at_max_depth`, Issue 9 above) catches overfill; the
15-minute `water_borehole_no_rise_protection` catches a pump running but not moving the
level at all. Neither catches a **slow-but-genuinely-rising** fill — a partially blocked
line or a degraded borehole yield — that never trips "no rise" and never reaches 1.95m,
but runs for hours longer than normal.
**User correctly rejected the original fix shape**: a flat wall-clock "max runtime
minutes" is arbitrary — a fill from empty legitimately takes far longer than a top-up, so
one fixed number either false-trips long normal fills or is set loose enough to miss real
degradation. **Redesigned as rate-based instead**: reuses `sensor.water_tank_depth_rate`
(m/h, the same sensor `water_borehole_no_rise_protection` already trusts) — new
`water_borehole_degraded_rise_rate_protection` automation stops the pump if the rate stays
continuously between the no-rise floor (0.01 m/h) and a tunable "healthy" floor
(`input_number.water_refill_degraded_rate_threshold`, default 0.05 m/h) for a tunable
sustained duration (`input_number.water_refill_degraded_rate_minutes`, default 60 min) —
self-scales to whatever the fill actually needs instead of a fixed clock. The old
`input_number.water_refill_max_runtime_minutes` helper (also unused/orphaned, same class
of issue as Issue 11) was replaced by the two new rate-based helpers rather than kept
alongside them.
**Caveat carried forward, not fully mitigated:** `sensor.water_tank_depth_rate` is a raw
`derivative` sensor with no smoothing (`time_window` not set) — a brief noise spike could
in principle reset the sustained-`for` timer before it completes, same risk the existing
no-rise protection already lives with. Not addressed here since the established pattern in
this codebase already trusts this sensor as-is; revisit both automations together if noise
turns out to be a practical problem.
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
- Triggers: `binary_sensor.water_refill_allowed → off` (catches grid offline, SOC drop, last-sun-slot, load control) — **debounced `for: "00:01:00"` since 2026-09-02** (same pattern as Issue 9's max-depth stop; recorder-confirmed the gate occasionally blips off for a few seconds on Tuya/recorder restart glitches, not real revocations — a real revocation stays off well past a minute) — AND `binary_sensor.water_solar_window_active → off` (end of solar day, not debounced — this is a scheduled input_datetime cutoff, not a chattering sensor).
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

### ~~Issue 8~~ — BUG: `counter.water_borehole_faults_week` wrong entity name in notifications
**Priority:** P2 — Weekly summary reports 0 faults regardless of actual count
**Status:** ✅ Doc-drift correction 2026-07-08 — already correct in live code, no code change needed
**File:** `packages/notifications/water_notifications.yaml:154`

**Was claimed:** References `counter.water_borehole_faults_week` but the actual entity is
`counter.water_borehole_faults_this_week`. Watchman "confirmed missing."

**Re-verified live 2026-07-08:** `water_notifications.yaml:154` already reads
`states('counter.water_borehole_faults_this_week')` — the correct entity, grep-confirmed
(`grep -rn "faults_week" packages/` returns only this one correct reference). Fixed at some
point without the doc being updated (same drift pattern as SECURITY ISSUE 3 and PRESENCE
BUG-P07, closed the same day). Same underlying claim also appears in NOTIFICATIONS_CONTRACT.md
BUG-N02 — corrected there too.

---

### Issue 9 — ✅ FIXED 2026-08-21: `water_safety.yaml` max-depth stop had commented-out debounce
**Priority:** was P2 — Tuya spike > 1.95m triggers false safety abort
**File:** `packages/water/water_safety.yaml`
**Root cause:** `# for: "00:01:00" #REQUIRED DEBOUNCE (ONE LINE)` — the comment said debounce was REQUIRED but it was commented out. Without it, a single erroneous Tuya reading above 1.95m (before the spike filter catches it) immediately stopped the pump and marked the cycle as aborted.
**Verified not covered elsewhere:** re-checked whether the separate stop-at-target-depth
automation (`water_tank_refill_control.yaml`, stops at `water_effective_fill_target` ≈
1.6m) makes this moot — it doesn't. That's a different automation, and this hard stop is
explicitly designed as an *independent* backstop for when that logic fails to fire (see
Locked Design Decisions: "Independent of refill logic"), so it can't assume the
target-reached path already protects it.
**Fix applied:** Uncommented the `for: "00:01:00"` line. A genuine rise to 1.95m+ sustains
well past 1 minute, so the debounce doesn't weaken the backstop — it only rejects a single
spurious reading.

---

### Issue 10 — ✅ FIXED 2026-08-21: DUAL WRITE — Duplicate notifications on emergency refills
**Priority:** was P3 — User receives 2 notifications per emergency refill
**File:** `packages/water/water_tank_refill_control.yaml`
**Root cause:** Both CRITICAL cases (2 and 3) called `script.notify_water_event` AND `notify.send_message` on `notify.telegram_bot_5527` directly. `script.notify_water_event` already has its own unconditional Telegram mirror (`telegram_bot.send_message`, fires for every severity including warning) — confirmed by reading `notify_water_events.yaml` directly. Result: 2 Telegram messages per emergency event.
**Note — supersedes a stale "Locked Design Decision":** this file's Locked Design Decisions
table carried a row calling the direct bypass an "accepted exception... critical emergency
notifications need guaranteed delivery." That exception predates `notify_water_event`
gaining its own reliable Telegram mirror (added afterward) — it's now obsolete. Row removed
below.
**Fix applied:** Removed both direct `notify.send_message` blocks. The central script's
mirror now delivers the sole Telegram message for both CRITICAL branches.

---

### Issue 11 — ✅ FIXED 2026-08-21: ORPHANED — `water_policy_helpers.yaml` threshold entities unused
**Priority:** was P3 — Dead code creates confusion about which thresholds are active
**File:** `packages/water/water_policy_helpers.yaml` (deleted)
**Root cause:** Defined `water_depth_full_threshold`, `water_depth_low_threshold`, `water_depth_critical_threshold`, `water_depth_empty_threshold`. None of these were referenced in any template, automation, or dashboard (re-confirmed via repo-wide grep). The active thresholds are in `water_helpers.yaml` (`water_depth_critical`, `water_depth_low`, etc.).
**Fix applied:** Deleted the file outright rather than wiring it in — the active thresholds
in `water_helpers.yaml` already serve this role and there was no in-progress consumer to
preserve.

---

### Issue 12 — ✅ PARTIALLY FIXED 2026-08-21: DEAD CODE — `water_refill_never_reached_full` had a tautological guard condition
**Priority:** was P3 — The automation can never fire in practice
**File:** `packages/notifications/water_notifications.yaml`
**Root cause (original, still applies):** Trigger: `binary_sensor.water_tank_refilling` stays on for 3 hours. But `water_borehole_no_rise_protection` fires after 15 minutes of no depth increase and stops the pump. The tank refilling binary_sensor turns off → 3-hour timer resets. No refill that fails to rise will ever stay on for 3 hours — this half of the reachability problem is unresolved, see "Fix needed" below.
**Second, independent bug found and fixed 2026-08-21:** the automation's `condition` checked
`sensor.water_state != 'full'` — a state that sensor can never produce (see Issue 2) — so the
condition was tautologically always `True`. Even in the rare case the trigger did fire, the
guard was providing zero actual filtering. Fixed to check
`binary_sensor.water_tank_full_depth == 'off'`, the real signal.
**Fix still needed (reachability, unchanged):** Replace the 3-hour stays-on trigger with a
trigger-based approach: when cycle starts, check 30–45 minutes later if the depth has risen
by a meaningful amount.

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

### Issue 16 — ✅ FIXED 2026-08-18: `input_text.borehole_last_fault` permanently stale — write path only existed in a disabled automation
**Priority:** P2 — every fault notification quoted the wrong fault  
**File:** `packages/water/water_protection_automations.yaml`  
**Root cause:** `input_text.borehole_last_fault` was only ever written by `water_borehole_dry_run_shutdown` — which is commented out (Issue in "Dry-Run Shutdown" section, intentionally disabled in favour of no-rise protection). The automation that actually fires, `water_borehole_no_rise_protection`, never wrote it. Six notification templates read it (`alerts_water.yaml`, `water_borehole_first_fault_notification`, tier 2/3 routing) expecting live fault detail — all of them were quoting a fault from **2026-02-18** regardless of when the real fault happened.  
**Found:** investigating the 2026-08-18 09:11 no-rise fault — user asked why alert messaging "seemed all over the place"; `input_text.borehole_last_fault` still showed the February value at the time of the live fault.  
**Fix:** Added the same `input_text.set_value` action (worded for no-rise, not dry-run) to `water_borehole_no_rise_protection`'s action sequence, alongside the existing counter increment.

---

### Issue 17 — ✅ FIXED 2026-08-18: `sensor.water_refill_last_outcome` showed "Completed Normally" mid-refill
**Priority:** P2 — dashboard "Refill Status"-adjacent sensor actively misleading during a live refill  
**File:** `packages/water/water_helpers.yaml:64`  
**Root cause:** The template's `else` branch defaulted to `"Completed Normally"` any time neither `water_refill_aborted_due_to_safety` nor `water_refill_manual_run` was `on` — a fallback, not an actual completion check. Since the safety-abort flag clears the instant a new cycle starts, the sensor read "Completed Normally" from the first second of the *next* cycle, including while the pump was still actively running.  
**Found:** same 2026-08-18 investigation — user reported the tank "is busy refilling manually" while status sensors looked wrong.  
**Fix:** Added an `input_boolean.water_refill_cycle_active` check ahead of the existing branches, reporting `"Refilling…"` while a cycle is in progress.

---

### Issue 18 — ✅ FIXED 2026-08-18: Dashboard "Refill Status" card always blank — pointed at a nonexistent attribute
**Priority:** P2 — user-facing, permanently blank field  
**File:** `.storage/lovelace.dashboard_operations` (gitignored — not covered by `gitupdate.sh`)  
**Root cause:** The "Refill Status" entities-card row read `attribute: reason` off `binary_sensor.water_refill_allowed`. That entity's template (`water_templates.yaml:302`) defines `grid`, `battery_soc`, `required_soc`, `borehole_enabled`, `orchestrator_state`, `conserve_blocked`, `last_sun_blocked` — never `reason`. No git history exists for this file (`.storage`, gitignored) so it's unclear whether `reason` was ever implemented; likely a card built ahead of the entity that never got wired up.  
**Fix:** Card now reads `sensor.water_refill_blocked_reason` directly (already existed, priority-ordered block reason string, "none" when refill isn't blocked) instead of the fake attribute. **⚠️ Requires full HA restart** (`.storage/lovelace` change — CODING_STANDARDS.md).

---

### Issue 19 — ✅ FIXED 2026-08-18: Water alert entities double-notified once `notify.STD_Alerts` was fixed
**Priority:** P2 — duplicate pushes per safety event  
**File:** `packages/alerts/alerts_water.yaml`  
**Root cause:** `alert.water_alert`, `alert.water_borehole_fault`, and `alert.water_borehole_critical_fault` all notified via `notifiers: STD_Alerts`. That group was dead (see `NOTIFICATIONS_CONTRACT.md` BUG-N16) when the 2026-07-06 `route_water_tank_alert` / `route_water_borehole_fault_alert_tier_2_3` / `route_water_borehole_critical_fault_alert_tier_3_5` automations were built as the working replacement, calling `script.notify_water_event` directly. `STD_Alerts` mobile delivery was fixed 2026-08-09 (`configuration.yaml:78`, BUG-N16) but the alert entities' own `notifiers:` were never removed — both paths then fired for the same event. Confirmed on the 2026-08-18 09:11 fault: `sensor.water_alert_context` hit `critical` at 09:11:01 (firing the route automation) while `alert.water_alert` turned `on` at 09:11:31 (its own `STD_Alerts` notifier, now live) — two pushes ~30s apart for one event, on top of a third, separately-triggered push from `water_borehole_first_fault_notification` (fault-count reaching 1, also at 09:11:01) — three pushes for one 70-second event. Full cross-domain writeup: `NOTIFICATIONS_CONTRACT.md` BUG-N18.  
**Fix:** Removed `notifiers: STD_Alerts` from all three `alert:` entities (matches the `alerts_security.yaml` `security_alert` BUG-A10 precedent — the alert entity stays for dashboard/ack visibility and its `repeat:` schedule, but the route automation is the sole delivery path). **⚠️ Requires full HA restart** (`alert:` entity change — CODING_STANDARDS.md).  
**Not fixed — flagged for a future session:** this exact pattern (`alert:` with live `notifiers: STD_Alerts` *and* a parallel `route_*` automation calling `script.notify_*_event`) also exists in `alerts_temperature.yaml`, `alerts_doors.yaml`, `alerts_presence.yaml`, `alerts_device_power.yaml`, `alerts_power.yaml`, `alerts_media.yaml`, `alerts_batteries.yaml`, `alerts_garden.yaml`, and `alerts_network.yaml` — all built during the same 2026-07-06 STD_Alerts-was-dead window. Since the 2026-08-09 fix, every one of those domains is likely double-notifying the same way water was. Out of scope for this session (water-only ask); worth a dedicated cross-domain sweep. See `NOTIFICATIONS_CONTRACT.md` §7.

---

### Issue 20 — ✅ ENHANCEMENT 2026-08-18: No-rise stop now auto-retries instead of always requiring a manual restart
**Priority:** P3 — reduces manual intervention, bounded added dry-run risk (user-accepted tradeoff)  
**File:** `packages/water/water_protection_automations.yaml` — `water_borehole_no_rise_protection`  
**Motivation:** Investigating the 2026-08-18 09:11 fault (see Issue 16-19 above) showed a no-rise trip isn't proof of a dry borehole — it's a NET signal, blind to the difference between "producing nothing" and "producing slowly, outpaced by concurrent house consumption." That fault: net depth -0.04m over the 15min run, but `sensor.water_tank_consumption_rate` read 0.24-0.48 m/h the *entire* window (never zero); raw vs validated depth matched exactly and `binary_sensor.water_tank_depth_sensor_stable`/`sensor.tuya_cloud_health` stayed on/healthy throughout (no sensor glitch); a manual restart refilled normally within seconds. User asked for an automatic bounded retry instead of always waiting for a human.  
**Design (user-selected from 2 options each):**
- **Delay:** urgency-scaled — 2 min if `sensor.water_state == 'safety'` (most urgent, depth 0.25-0.35m), 5 min otherwise (more patient — lets a slow borehole/air-lock settle or concurrent usage stop).
- **Cap:** auto-retry only while `counter.water_borehole_faults_today <= 2` (i.e. retries after fault #1 and #2, not #3+). Deliberately lines up with the existing tier-2 "repeated fault" alert at `>=3 faults` (`binary_sensor.water_borehole_fault_alert_active`) — by the time auto-retry stops, the user is already being told via that pipeline.
- **Re-check before retrying:** pump still off, `water_tank_refill_enabled` + `load_control_borehole_enabled` still on, depth still `< 1.95`. Any failing just ends the sequence quietly — no retry, no error.
- **Scope:** added to the no-rise automation's own action sequence, not a separate automation keyed off the shared `water_refill_aborted_due_to_safety` flag — battery-hard-stop (Protection 4) and max-depth-stop are different failure modes that should NOT auto-retry the same way.
- **Notification:** fires an `information`-severity `[Code: borehole_auto_retry]` push before retrying (in addition to the existing warning-severity no-rise notification).
- **Accepted tradeoff:** worst-case dry-run exposure rises from 15min (1 attempt) to 45min (3 attempts x 15min) if the borehole genuinely is dry. User confirmed this is acceptable given the alert-threshold cap.
**Reload:** Automations reload only — no restart required (unlike Issues 18/19 above).

---

## 7. Error Signatures (Watchman-Confirmed)

| Entity | Status | File | Issue |
|---|---|---|---|
| `binary_sensor.borehole_pump_dry_run` | missing | dashboard | Intentionally disabled (commented out) |
| `automation.water_refill_cycle_*` | missing | dashboard | Renamed/restructured — dashboard needs update |
| `automation.water_borehole_dry_*` | missing | dashboard | Intentionally disabled |
| `automation.water_borehole_max_*` | missing | dashboard | Not implemented (Issue 5) |
| ~~`counter.water_borehole_faults_week`~~ | N/A | water_notifications.yaml:154 | ✅ Doc-drift, closed 2026-07-08 — live code already correct (Issue 8) |
| ~~`sensor.water_refill_flow_rate`~~ | defined | water_reporting.yaml | ✅ Fixed 2026-07-08 (Issue 1) |
| `sensor.water_tank_level_percent` | missing | water_reporting.yaml:54 | Entity ID mismatch (name truncation) |

**Note on `sensor.water_tank_level_percent`:** Defined with name "Water Tank Level %" (unique_id: `water_tank_level_percent`). HA generates entity_id from name stripping `%` → may produce `sensor.water_tank_level_` which doesn't match what reporting.yaml calls. Verify actual entity_id in HA UI.

---

## 8. Optimization Recommendations

### Recommendation 1 — ✅ DONE 2026-07-08 (Issue 3): Unify start_depth ownership
Control automation and capture automation both wrote to `water_refill_start_depth`. Resolved per the contract's own "RAW sensor, validation analytics-only" invariant — capture automation is sole owner, the pre-pump write removed from `water_tank_refill_control.yaml`.

### Recommendation 2 — ✅ DONE 2026-08-21 (Issue 11): `water_policy_helpers.yaml` deleted
The policy helpers were designed to replace hardcoded thresholds but never got wired in. Resolved by deleting the file (option 2) rather than wiring it — the real thresholds already lived in `water_helpers.yaml`, so there was no in-progress consumer to preserve.

There are two competing threshold systems right now. Pick one.

### Recommendation 3 — ❌ EVALUATED, NOT IMPLEMENTED 2026-08-21: Add explicit `"full"` state to `sensor.water_state`
**Decision: do not implement.** The practical need this recommendation targeted is
already covered by `binary_sensor.water_tank_full_depth` (fires at depth ≥ 1.95m,
drives `water_tank_full_notification` — see Issue 2's 2026-08-21 resolution, which
found the notification already working off that binary sensor, not off
`sensor.water_state`) and by `sensor.borehole_control_status`'s "Tank full" status
line (`water_state == 'ok' and level >= 90`, `water_state_extensions.yaml`).

Adding `"full"` as a new `sensor.water_state` value is not a safe drop-in — it would
be a **breaking change** to existing consumers that treat `'ok'` as the sole "nothing
wrong" state:
- `alerts_water.yaml`'s `binary_sensor.water_alert_active` and
  `sensor.water_alert_context` both gate on
  `states('sensor.water_state') not in ['ok', 'unknown', 'unavailable']` to decide
  whether the tank is in a bad state. A tank reaching `'full'` would newly satisfy
  that condition and could false-fire a `warning`/`critical` water alert (e.g.
  combined with a stuck `water_refill_aborted_due_to_safety` flag) for the one
  state that is least worth alerting on.
- `sensor.borehole_control_status`'s own "Tank full" branch already tests
  `water_st == 'ok'` — it would silently stop matching and fall through to a
  different status line.

Making this change safely would mean auditing and updating every `== 'ok'` /
`not in ['ok', ...]` check across `alerts_water.yaml`, `water_state_extensions.yaml`,
and any dashboard cards — for a feature whose only stated benefits (tank-full
notification, dashboard clarity) are already delivered by the binary sensor and the
control-status sensor. Not a clear win; closing as evaluated-and-declined rather than
leaving it open indefinitely.

### Recommendation 4 — ❌ EVALUATED, NOT IMPLEMENTED 2026-08-21: Formalize state machine as an explicit sensor
**Decision: do not implement as specified.** `sensor.water_refill_cycle_summary`
(`water_refill_cycle.yaml`) already substantially satisfies this recommendation's
intent — its own header comment calls it "Single source of truth for completed
refill cycles. Dashboards, reports, and audits read ONLY from here." It already
merges `input_boolean.water_refill_cycle_active` +
`input_boolean.water_refill_aborted_due_to_safety` into one state
(`active`/`aborted`/`completed`) with full attributes (start/end depth, timing,
avg flow, `aborted_due_to_safety`, `manual_run`), and is already consumed
elsewhere (`water_templates.yaml:192`, referenced in `water_safety.yaml`).
`sensor.water_refill_cycle` separately gives `idle`/`refilling`/`completed`.

Adding a third sensor (`sensor.water_lifecycle_state`, idle/running/completed/aborted)
would not consolidate anything — it would be a **third** representation of the same
underlying flags, the exact kind of drift-prone duplication this contract's
Recommendation 2 (deleted `water_policy_helpers.yaml`) and the 2026-08-21 dashboard
recalibration note (§7) both already flagged as a pattern to avoid.

It would also brush up against `a_water_lifecycle_contract.yaml`'s (locked, read-only)
"REQUIRED FLAGS" section, which names
`water_refill_cycle_active`/`water_refill_manual_run`/`water_refill_aborted_due_to_safety`
as the source of truth — a new sensor claiming to be *the* lifecycle state risks being
read as a second, competing source of truth rather than a pure derived view.

**Minor gap noted, not actioned:** `sensor.water_refill_cycle_summary` has no true
"idle" state — before any cycle has ever run, or between cycles, it falls through to
`completed` (since it only distinguishes `active` vs `aborted`, defaulting to
`completed` otherwise). This is cosmetic (matters only fresh off a first-ever install)
and not worth a new sensor to fix; flagging here in case a future session wants to add
an explicit idle branch to the existing sensor instead of creating a new one.

### Recommendation 5 — ✅ DONE 2026-08-21: Rate-limit spike notifications
Implemented as a cooldown gate, not the `rate_limit_minutes: 30` shorthand originally
suggested — that field doesn't exist on `script.notify_water_event`
(`packages/notifications/notify_water_events.yaml` has no `rate_limit_minutes` field
and no logic for it; the two `rate_limit_minutes: 60` call-sites already in
`water_tank_refill_control.yaml`'s emergency-refill branches are themselves silently
inert extra data, unrelated to this fix). Instead used this repo's existing
`input_datetime.<x>_last_alert` cooldown-gate idiom (same pattern as
`unknown_draw_warning` in `power_automations.yaml`):
- New `input_datetime.water_spike_notify_last_sent` +
  `input_number.water_spike_notify_cooldown_minutes` (default 30) in
  `water_helpers.yaml`.
- `water_depth_spike_rejected` (`water_protection_automations.yaml`) now wraps the
  `script.notify_water_event` call in an `if:` gated on elapsed time since
  `water_spike_notify_last_sent` ≥ the cooldown, stamping the timestamp *before*
  notifying (blocks re-entrant fires during the notify call itself). `logbook.log`
  stays unconditional — every rejection is still in the audit trail, only the
  push/Telegram notify is throttled.
- Validated: local YAML parse + `ha core check` both pass.

---

## 9. Implementation Checklist

### Sprint 1 — Fix Safety Gaps (Issues 5 + 6)

- [✅] Redesigned, not a flat cutoff — `water_borehole_degraded_rise_rate_protection`
      added to `water_protection_automations.yaml` 2026-08-21 (Issue 5). Rate-based
      (`sensor.water_tank_depth_rate` vs `input_number.water_refill_degraded_rate_threshold`,
      sustained for `input_number.water_refill_degraded_rate_minutes`), not the flat-minutes
      `water_refill_max_runtime_minutes` this line originally specified — that helper was
      deleted, judged arbitrary compared to a rate that self-scales with fill duration.
- [✅] Implement battery hard stop reactive automation in `water_safety.yaml` monitoring `sensor.battery_soc` — `water_safety_battery_hard_stop` (already implemented, see Issue 6)

### Sprint 2 — Fix Trigger Integrity (Issues 4 + 9)

- [✅] Add `from: "off"` to pump ON triggers — `water_borehole_no_rise_protection` fixed 2026-04-15 (E1)
- [✅] Add `from: "on"` to pump OFF triggers — `water_reconcile_cycle_state` fixed 2026-04-15 (E1)
- [✅] Add stability window to `water_refill_visibility_guard` pump ON trigger — `for: "00:00:10"` added 2026-04-15 (E2)
- [✅] Audit all `to: "on"` pump triggers — complete. `water_block_refill_when_not_allowed` already had `from: "off"`. `water_capture_refill_start` already had all constraints.
- [✅] Uncomment `for: "00:01:00"` debounce in `water_stop_refill_at_max_depth` — done 2026-08-21 (Issue 9)
- [✅] E3 verified: `binary_sensor.water_refill_allowed` checked before pump start on critical+low paths. Safety/limited-critical paths intentionally bypass (emergency scenarios).

### Sprint 3 — Fix Data Integrity (Issues 1 + 2 + 3 + 8)

- [✅] Add `sensor.water_refill_flow_rate` statistics sensor to `water_reporting.yaml` — DONE 2026-07-08 (Issue 1), plus `input_number.water_refill_avg_flow_last_week` + weekly snapshot automation for "last week"
- [✅] Fix `water_tank_full_notification` — already triggers on `binary_sensor.water_tank_full_depth = on`, confirmed live 2026-08-21 (Issue 2, was doc-drift not a real bug)
- [✅] Fix double-write to `water_refill_start_depth` — remove write from control automation — DONE 2026-07-08 (Issue 3), all 6 branches in `water_tank_refill_control.yaml`
- [✅] Fix `counter.water_borehole_faults_week` → `counter.water_borehole_faults_this_week` in `water_notifications.yaml` — was already correct live, doc-drift closed 2026-07-08 (Issue 8)

### Sprint 4 — Design Clean-up (Issues 7 + 10 + 11)

- [✅] Update `binary_sensor.water_refill_allowed` to include `water_tank_refill_enabled` check — fixed 2026-06-19
- [✅] Remove direct Telegram calls from `water_tank_refill_control.yaml` (use central script only) — done 2026-08-21 (Issue 10)
- [✅] Decide: wire `water_policy_helpers.yaml` into templates OR delete file — deleted 2026-08-21 (Issue 11)

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
- **Fixed 2026-08-21 (Issue 9):** 1-minute debounce re-enabled — no longer a single-spike false-abort risk.
- **Verdict:** ✅ IMPLEMENTED, debounced. Threshold mismatch: `water_target_depth_full` = 1.85m but stop fires at 1.95m.
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
- **Auto-retry (added 2026-08-18, Issue 20):** a no-rise trip is a NET signal — it can't distinguish "borehole producing nothing" from "borehole producing slowly, outpaced by concurrent house consumption." Confirmed real on 2026-08-18: net depth -0.04m over the 15min run, but `sensor.water_tank_consumption_rate` read 0.24-0.48 m/h the entire window (never zero), sensors were healthy/stable throughout, and a manual restart refilled normally within seconds. The automation now auto-retries after a cooldown instead of requiring a manual restart every time — see Issue 20.
- **Verdict:** IMPLEMENTED. Works correctly for genuine no-rise conditions. Spike sensitivity is a minor risk.

### Protection 4: Battery Hard Stop
- **Spec:** SOC drops below `water_battery_soc_hard_stop` → Stop pump  
- **Implementation:** ✅ `water_safety.yaml` — `water_safety_battery_hard_stop` (added 2026-05-25)
- **Trigger:** `sensor.battery_soc` below `input_number.water_battery_soc_hard_stop`
- **Threshold:** 40% (temporary — current battery cells unreliable below 40%). Lower to 20% after new battery installation end of week 2026-05-25.
- **Coverage:** No SAFETY-state exemption — applies even to emergency refills (unlike `water_borehole_mid_run_shutdown`). Sets `water_refill_aborted_due_to_safety`.
- **Verdict:** ✅ IMPLEMENTED. See Issue 6 for full coverage details.

### Protection 5: Degraded Rise Rate Protection (redesigned 2026-08-21, was "Max Runtime Cutoff")
- **Original spec:** Pump on > `water_refill_max_runtime_minutes` → Stop pump
- **Redesigned, not implemented as originally specified:** a flat wall-clock cutoff was
  judged arbitrary (a fill from empty legitimately takes longer than a top-up) — see Issue
  5. Replaced with a rate-based guard instead.
- **Implementation:** ✅ `water_protection_automations.yaml` — `water_borehole_degraded_rise_rate_protection`
- **Trigger:** Pump on, depth rate stays between the no-rise floor (0.01 m/h) and
  `input_number.water_refill_degraded_rate_threshold` (default 0.05 m/h) continuously for
  `input_number.water_refill_degraded_rate_minutes` (default 60 min)
- **Coverage:** Catches a pump that's genuinely rising but too slowly — the gap between
  Protection 3 (needs near-zero rate) and Protection 1 (needs overfill). The old
  `input_number.water_refill_max_runtime_minutes` helper was deleted, superseded by the two
  new rate-based helpers. Emergency case 3 still has its own hardcoded 10-minute limit,
  unaffected by this change.
- **Verdict:** ✅ IMPLEMENTED. Same noisy-sensor caveat as Protection 3 (untested at
  scale — flag if it proves too trigger-happy or too lax in practice).

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
| As an audit record | ⚠️ Moderate | Start depth written from RAW, single writer only since Issue 3 fix (2026-07-08) |
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
| ~~Emergency Telegram calls are direct (bypassing central script)~~ **SUPERSEDED 2026-08-21** | Row removed — `script.notify_water_event` gained its own unconditional Telegram mirror after this exception was written, making the direct bypass a duplicate-message bug (WATER_CONTRACT Issue 10, fixed) rather than a needed guarantee. Direct calls removed from `water_tank_refill_control.yaml`'s CRITICAL branches. |

---

### Investigated 2026-09-02: "watering full M/W/F/S, smaller afternoon, half other days" — not a schedule, and not predictive fill

User reported a perceived weekly watering rhythm and asked whether predictive fill needed tuning. Checked live entities + recorder DB directly rather than relying on docs:

- **Predictive fill (Branch 4.7) was a red herring.** `water_predictive_fill_enabled=on`, threshold correctly live at 75% (the 2026-07-17 recalibration caveat — "live value still 50%" — no longer applies, it's live now). But Branch 4.7 also gates on `sensor.energy_orchestrator_state == 'conserve'`, and over the prior 10 days the orchestrator was **never once** in `conserve` (only `normal`/`surplus`, plus a few instant restart-glitch `critical` blips). Branch 4.7 hasn't fired at all — correctly idle given healthy solar/SOC, not broken.
- **The demand-day → depth-target system is the real driver, and it's working as designed.** `sensor.water_effective_fill_target` reads `sensor.water_target_depth_tomorrow` (line 443 of `water_templates.yaml`), i.e. the tank fills toward **tomorrow's** demand target, not today's — intentional look-ahead pre-positioning, confirmed against the original E5 design note. This makes Friday's fill look "biggest" (it's pre-filling for Saturday's Pool/1.6m day), which can read as a M/W/F/S pattern depending on how the demand selectors land that week.
- **Multiple pump on/off cycles per day are NOT `water_borehole_mid_run_shutdown` chatter.** Initial hypothesis (built from a raw `COUNT(state='off')` per day — 21–56/day) was wrong: those rows are mostly duplicate re-published states (attribute-only updates), not real transitions. Deduplicating to actual state changes, `binary_sensor.water_refill_allowed` is stable all day; the only transitions are the evening dusk-cutoff/22:00 resume and a handful of few-second Tuya/recorder blips (mostly after solar hours). Real daytime blip count in 10 days: **one** (Aug 31, 12:00–12:03). The 2-4 separate pump sessions/day seen in the recorder are almost certainly the pump legitimately reaching `water_effective_fill_target`, stopping, then house consumption draining the tank back below target later the same day, and Branch 4 restarting it — correct behavior, not a bug.
- **Fix applied anyway:** debounced the `permission_revoked` trigger in `water_borehole_mid_run_shutdown` with `for: "00:01:00"` (mirrors Issue 9's max-depth debounce) as cheap insurance against that one blip class. Confirmed low-risk — it's a `condition:` re-check at trigger time elsewhere (Branch 4, block automation), not a subscribed watcher, so nothing else depends on instant reaction to this sensor.
- **No code bug found or fixed in the demand-target/predictive-fill logic itself** — both are working as designed. Nothing else changed.

---

*Last updated: 2026-08-21 — Issues 2 (doc-drift correction, already fixed), 5 (downgraded
P2→P3, not implemented, awaiting user decision), 9, 10, 11, 12 (closed/partially closed) —
see PROJECT_STATE.md 2026-08-21 session entry for full detail. Locked Design Decisions:
"Emergency Telegram calls are direct" row superseded (Issue 10 fix).*
*Last updated: 2026-06-14 (E7)*  
*Updated by: E7 — sensor.water_usage_today (utility_meter), sensor.water_tank_consumption_integral (integration), sensor.water_daily_usage_mean (statistics), sensor.water_effective_fill_target (template). Branch 4.7 predictive fill added. water_stop_at_daily_target updated to read water_effective_fill_target. Predictive fill enabled/wired — see Predictive Fill Helpers section above.*
