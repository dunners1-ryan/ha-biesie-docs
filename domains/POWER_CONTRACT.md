# HABiesie — Power & Energy Domain Contract
> **Living contract.** Produced by deep audit on 2026-04-13.  
> Update after every meaningful change to the power package.  
> This document supersedes `POWER_CONTEXT.md` — keep both in sync.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [File Inventory](#2-file-inventory)
3. [Inverter Register Map](#3-inverter-register-map)
4. [Measurement Model](#4-measurement-model)
5. [Entity Reference — Real-Time Sensors](#5-entity-reference--real-time-sensors)
6. [Entity Reference — Energy & Prepaid](#6-entity-reference--energy--prepaid)
7. [Entity Reference — Strategy & Decision Layer](#7-entity-reference--strategy--decision-layer)
8. [Entity Reference — Input Helpers](#8-entity-reference--input-helpers)
9. [Data Flow Maps](#9-data-flow-maps)
10. [Cross-Domain Interface](#10-cross-domain-interface)
11. [Known Issues](#11-known-issues)
12. [Error Signatures (Watchman-Confirmed)](#12-error-signatures-watchman-confirmed)
13. [Optimization Recommendations](#13-optimization-recommendations)
14. [Implementation Checklist](#14-implementation-checklist)
15. [Package Architecture & Dependency Analysis](#15-package-architecture--dependency-analysis)

---

## 1. System Overview

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEASUREMENT LAYER                            │
│                                                                 │
│  Inverter 1 (Master)          Inverter 2 (Slave)               │
│  ├─ grid_power                ├─ pv_power                      │
│  ├─ grid_import/export        ├─ load_power                    │
│  ├─ losses                    ├─ battery_power                 │
│  └─ BMS / battery SOC         └─ PV string 1 & 2              │
│                                                                 │
│  Prepaid Meter (Eskom)        Solcast (Forecast)               │
│  └─ lifetime_import (manual)  └─ forecast_today / next_hour   │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                  AGGREGATION LAYER (power_core.yaml)            │
│                                                                 │
│  house_solar_power      = inv1_pv + inv2_pv                    │
│  house_load_power       = inv1_load + inv2_load                │
│  house_grid_power       = inv1_grid + inv2_grid                │
│  house_battery_power    = inv1_batt + inv2_batt                │
│  inverter_battery_soc   = (inv1_soc + inv2_soc) / 2           │
│  inverter_pv_power      = alias of house_solar_power           │
│  inverter_load_power    = alias of house_load_power            │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              DERIVED / FINANCIAL LAYER                          │
│                                                                 │
│  energy_core.yaml    ─── energy flow split, percentages        │
│  energy_helpers.yaml ─── utility meters, daily/monthly totals  │
│  prepaid_core.yaml   ─── authoritative balance, drift model    │
│  prepaid_strategy.yaml── buy score, depletion date, decision   │
│  solar_core.yaml     ─── export potential, surplus             │
│  solar_clipping.yaml ─── unused power, opportunity level       │
│  solar_state.yaml    ─── ramp rate, confidence, stability      │
│  battery_runtime.yaml─── program-aware runtime estimation      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│               STRATEGY / DECISION LAYER                         │
│                                                                 │
│  power_state.yaml     ─── house_power_health, power_state      │
│  power_strategy.yaml  ─── power_strategy, severity             │
│  energy_state.yaml    ─── orchestrator, resilience hours       │
│  grid_risk.yaml       ─── grid_risk_level, grid_risk_severity  │
│  battery_state.yaml   ─── battery_state_health                 │
│  solar_forecast.yaml  ─── inverter mode automation             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              CONSUMER LAYER (cross-domain)                      │
│                                                                 │
│  Water: binary_sensor.water_refill_allowed                     │
│  Alerts: sensor.house_power_health → alert routing             │
│  Lighting: sensor.power_state → energy save mode               │
│  Automations: energy_orchestrator_state → load scheduling      │
└─────────────────────────────────────────────────────────────────┘
```

### Hardware

| Component | Details |
|---|---|
| Inverter 1 (Master) | Sunsynk — measures grid, losses, BMS |
| Inverter 2 (Slave) | Sunsynk — measures PV, load, battery |
| PV strings | 4× MPPT (2 per inverter), max ~10.8 kW |
| Battery bank | ~31,776 Wh total (31.8 kWh) |
| Grid | Eskom prepaid meter — **Empire CIU EV-KP**, Johannesburg City Power Area 3-11 Weltevredenpark |
| Solar forecast | Solcast integration |
| Load shedding | SePush/load_shedding integration |

---

## 2. File Inventory

**Package path:** `packages/power/` — 25 files

| File | Role | Layer |
|---|---|---|
| `power_core.yaml` | Dual-inverter aggregation — all unified sensors | Aggregation |
| `power_helpers.yaml` | Load groups (groups) + area/load power templates | Mixed (violates layering) |
| `power_templates.yaml` | Per-inverter PV, battery, grid sub-sensors | Derived |
| `power_state.yaml` | house_power_health, power_state, inverter health, freq/temp | State |
| `power_statistics.yaml` | Rolling average sensors + weather correlation (added 2026-04-22) | Derived |
| `power_strategy.yaml` | power_strategy, power_strategy_status, severity | Decision |
| `power_automations.yaml` | Power domain automations — migrated from automations.yaml | Automation |
| `power_contract.yaml` | Documentation notes only — no executable YAML | Docs |
| `battery_runtime.yaml` | Program-aware battery runtime, severity, confidence | Derived |
| `battery_state.yaml` | battery_state_health (strong/healthy/low/critical) | State |
| `energy_core.yaml` | Energy flow split sensors, percentages, integration sensors | Derived |
| `energy_helpers.yaml` | Area utility meters, energy series template sensor | Helpers |
| `energy_state.yaml` | Orchestrator state, economy score, loss analysis, statistics sensors | State |
| `grid_risk.yaml` | grid_risk_level, grid_risk_status, grid_dependency_eta | State |
| `grid_state.yaml` | grid_state_health (stable/risk/unstable/offline) | State |
| `load_control.yaml` | Load enable/disable helpers + turn-off automations (geyser/pool/borehole) | Helpers + Automation |
| `load_shedding.yaml` | **EMPTY** — content migrated to `packages/load_shedding/` (2026-04-21) | — |
| `prepaid_core.yaml` | Prepaid balance, drift model, utility meters, automations | Core + Derived |
| `prepaid_helpers.yaml` | All prepaid input_number helpers | Helpers |
| `prepaid_strategy.yaml` | Buy score, buy decision, depletion date, net position | Decision |
| `solar_clipping.yaml` | Unused power, opportunity level, efficiency loss, clipping | Derived |
| `solar_core.yaml` | Solar export potential, surplus sensor + binary_sensor | Core |
| `solar_forecast.yaml` | Forecast sensors, inverter mode automation (3h) | Derived + Automation |
| `solar_helpers.yaml` | solar_surplus_threshold, solar_export_tariff | Helpers |
| `solar_state.yaml` | Solar status, ramp rate, confidence, stability, window | State |

**Load shedding package path:** `packages/load_shedding/` — 1 file (new 2026-04-21, requires restart)

| File | Role |
|---|---|
| `load_shedding_templates.yaml` | Status card, active binary_sensor, severity, minutes countdown |

### Layering Violation

`power_helpers.yaml` contains both `group:` definitions (helpers layer) and `template: sensor:` blocks (templates layer). This violates the `*_helpers.yaml` convention but is functional. Flag for eventual split into `power_groups.yaml` + `power_templates.yaml` addition, but do not split unless refactoring.

---

## 3. Inverter Register Map

### Master/Slave Assignment

```
Inverter 1 = MASTER
  Solarman device: inverter_1_*
  Measures:
    sensor.inverter_1_grid_power         ← grid import/export (W)
    sensor.inverter_1_today_energy_import← today grid imported (kWh)
    sensor.inverter_1_total_energy_import← lifetime grid imported (kWh)
    sensor.inverter_1_pv_power           ← string 1+2 PV (W)
    sensor.inverter_1_pv1_power          ← string 1 (W)
    sensor.inverter_1_pv2_power          ← string 2 (W)
    sensor.inverter_1_pv1_voltage        ← string 1 voltage (V)
    sensor.inverter_1_pv2_voltage        ← string 2 voltage (V)
    sensor.inverter_1_battery_state      ← charging/discharging/idle
    sensor.inverter_1_battery_temperature
    sensor.inverter_ac_temperature       ← AC temperature (max of both)
    sensor.inverter_dc_temperature       ← DC temperature (max of both)
    sensor.inverter_1_battery_soc        ← contributes to averaged SOC
    sensor.inverter_1_load_power         ← contributes to combined load

Inverter 2 = SLAVE
  Solarman device: inverter_2_*
  Measures:
    sensor.inverter_2_pv_power           ← string 3+4 PV (W)
    sensor.inverter_2_pv1_power          ← string 3 (W)
    sensor.inverter_2_pv2_power          ← string 4 (W)
    sensor.inverter_2_pv1_voltage        ← string 3 voltage (V)
    sensor.inverter_2_pv2_voltage        ← string 4 voltage (V)
    sensor.inverter_2_battery_power      ← battery charge/discharge (W)
    sensor.inverter_2_load_power         ← contributes to combined load
    sensor.inverter_2_battery_soc        ← contributes to averaged SOC
```

### Combined Sensor Aggregation Rules

| Combined Sensor | Rule | Source |
|---|---|---|
| `sensor.house_solar_power` | inv1_pv + inv2_pv | power_core.yaml |
| `sensor.house_load_power` | inv1_load + inv2_load | power_core.yaml |
| `sensor.house_grid_power` | inv1_grid + inv2_grid | power_core.yaml |
| `sensor.house_battery_power` | inv1_batt + inv2_batt | power_core.yaml |
| `sensor.inverter_battery_soc` | (inv1_soc + inv2_soc) / 2 | power_core.yaml |
| `sensor.inverter_ac_temperature` | max(inv1, inv2) | power_state.yaml |
| `sensor.inverter_dc_temperature` | max(inv1, inv2) | power_state.yaml |
| `sensor.inverter_grid_frequency` | (inv1_freq + inv2_freq) / 2 | power_state.yaml |

### Known Aggregation Workaround

`sensor.inverter_today_energy_import` uses `inverter_1_today_energy_import * 2` when inverter_2 reads zero. This is a Solarman integration quirk where Slave doesn't reliably report grid import. Acknowledged workaround; pending direct grid meter integration to replace.

### PV String Layout

```
Inverter 1 (Master):
  PV1 → string 1 (roof section A)
  PV2 → string 2 (roof section B)

Inverter 2 (Slave):
  PV1 → string 3 (roof section C)
  PV2 → string 4 (roof section D)

sensor.inverter_1_pv_voltage = max(pv1_voltage, pv2_voltage)  ← BUG: elif branch uses wrong entity
```

---

## 4. Measurement Model

### Physical Meter — Empire CIU EV-KP

The prepaid meter is **physically separate from the inverter system** — it has no network connection and is NOT integrated into HA. All readings are entered manually.

| Meter Code | Reading | Notes |
|---|---|---|
| `1.8.0` | Lifetime total import (kWh) | → `input_number.prepaid_meter_lifetime_import` |
| `1.9.0` | Monthly import (kWh) | Reference/cross-check only |
| `98.2.1` | Yesterday import (kWh) | Reference/cross-check only |
| `C.51.0` | Current remaining balance (kWh) | → `input_number.prepaid_units` |

### Source of Truth Hierarchy

```
Priority 1 (AUTHORITATIVE — manual entry):
  input_number.prepaid_meter_lifetime_import
  └── Manually entered from physical meter display (code 1.8.0)
  └── ONLY updated when user reads physical meter

Priority 2 (OPERATIONAL — high-frequency):
  sensor.grid_energy_import_total
  └── Inverter 1 cumulative grid import
  └── Resets on power cycle — drift accumulates over time

Priority 3 (RECONCILIATION — manual correction):
  input_number.prepaid_alignment_offset
  └── Corrects accumulated drift between priorities 1 and 2
  └── Applied as: authoritative_usage = inverter_total + offset - baseline
```

### Authoritative Balance Formula

```
sensor.prepaid_units_used_authoritative =
    sensor.grid_energy_import_total
  + input_number.prepaid_alignment_offset
  - input_number.initial_prepaid_grid_import    (baseline at last meter reading)

sensor.prepaid_units_left_authoritative =
    input_number.prepaid_meter_lifetime_import
  - sensor.prepaid_units_used_authoritative
  (capped at prepaid_units input if positive)

sensor.prepaid_units_left_safe =
    IF drift_pct > 1%:
      input_number.prepaid_units    (manual override, from physical meter display)
    ELSE:
      sensor.prepaid_units_left_authoritative
```

### Drift Model

```
sensor.prepaid_grid_meter_drift =
    sensor.grid_energy_import_total - sensor.prepaid_meter_lifetime_import

sensor.prepaid_drift_percentage =
    (drift / prepaid_meter_lifetime_import) * 100

When drift_pct > 1%:
  → prepaid_units_left_safe switches to manual input
  → reconciliation suggestion and auto-realign offered

Auto-realign script (script.prepaid_realign_offset):
  → sets alignment_offset to match current actual readings
```

### Known Drift Pattern

Daily reconciliation is required because:
- Physical meter updates infrequently (daily or on-demand)
- Inverter records continuously but is subject to resets, reboots, and startup lag
- **Measurement mismatch**: Inverter measures AC-side; meter measures after wiring losses — this creates structural ±1-3% monthly variance even without resets
- Expected variance: **±1-3% monthly** (grid losses + meter type differences)

**Monitoring**: Check `sensor.prepaid_grid_meter_drift` and `sensor.prepaid_drift_percentage` regularly. If drift > 5%, use the "Realign Prepaid Offset" script button to correct.

**Auto-reconciliation (added 2026-04-22):** `prepaid_auto_reconcile` automation triggers on `input_number.prepaid_meter_lifetime_import` change. If drift % exceeds `input_number.prepaid_drift_threshold` (default 2%), calls `script.prepaid_realign_offset` automatically and notifies via `script.notify_power_event`.

**Balance confidence (added 2026-04-22):** `sensor.prepaid_balance_confidence` (0–100%) — graduated score: 100% at drift <0.5%, decays linearly to 50% at 2%, to 0% at 5%+. Additional -2%/day penalty for each day over 30 since last meter verification (`input_datetime.prepaid_last_meter_verification`), max penalty 30%.

**Drift history tracking (added 2026-04-23):** `script.prepaid_realign_offset` enhanced — captures drift kWh immediately before each realign into `input_number.prepaid_drift_at_last_realign`, stamps `input_datetime.prepaid_last_realign_time`, and calculates `sensor.prepaid_drift_rate_per_day` (kWh/day since last realign). Notification includes WORSENING/IMPROVING trend flag. `weekly_prepaid_meter_verification_reminder` bug fixed — no longer stamps `prepaid_last_meter_verification` on reminder fire (only realign click counts as verification).

**Prepaid top-up workflow (correct order):**
1. Enter `prepaid_meter_lifetime_import` (1.8.0) + `prepaid_units` (C.51.0) → click **Realign** (establishes ground truth, records drift)
2. Enter purchase details → click **Update Prepaid Units** (snapshots grid baseline on clean post-realign state)

**Prepaid diagnostic (added 2026-04-23):** `pyscript.prepaid_diagnostic` (mirrors `power_snapshot.py` pattern) — creates persistent_notification with full reconciliation state, drift trend, balance cross-check, and realign preview. Callable via `script.prepaid_diagnostic` dashboard button or Developer Tools → Services.

**`sensor.prepaid_net_position_this_month`**: Always available (no guard, all inputs use `| float(0)`). Returns 0 at month start before utility meters accumulate. Formula: `solar_savings_this_month - prepaid_spend_this_month`.

### Reliability Classification

| Tier | Sensors | Trust | Notes |
|---|---|---|---|
| AUTHORITATIVE | `prepaid_meter_lifetime_import`, `prepaid_units_left_safe` (when drift low) | High | Manual entry is ground truth |
| OPERATIONAL | `grid_energy_import_total`, `inverter_battery_soc`, `house_solar_power` | High | High frequency, reset risk |
| DERIVED/RELIABLE | `prepaid_units_used_authoritative`, `prepaid_drift_percentage`, `energy_independence` | High | Clean formulas, verified |
| ESTIMATED | `prepaid_estimated_days_remaining`, `prepaid_adaptive_burn_rate`, `battery_runtime` | Medium | Depends on rolling averages + thresholds |
| BROKEN | `battery_night_survival` | None | References non-existent entities |
| FIXED 2026-04-21 | `grid_charging_while_solar`, `grid_to_house_power`, `grid_used_by_house_today` | High | grid_to_house now subtracts battery charge; daily today sensors use inverter-native totals |

---

## 5. Entity Reference — Real-Time Sensors

### Aggregated Power (power_core.yaml)

```
sensor.house_grid_power          W   grid import (positive) / export (negative)
sensor.house_solar_power         W   total PV generation
sensor.house_load_power          W   total house consumption
sensor.house_battery_power       W   battery charge (negative) / discharge (positive)
sensor.inverter_battery_power    W   battery power (same as house_battery_power)
sensor.inverter_battery_soc      %   average of inv1 + inv2 SOC
sensor.inverter_pv_power         W   alias of house_solar_power
sensor.inverter_load_power       W   alias of house_load_power
sensor.inverter_grid_power       W   alias of house_grid_power
```

### UI-Created Template Numbers (core.config_entries — NOT in YAML)

These are template number entities created via Settings → Devices & Services → Template. They live in
`.storage/core.config_entries` and cannot be edited in YAML. If HA storage is restored from backup,
these may revert to their pre-fix state and need to be re-applied.

```
number.inverter_battery_capacity  Ah  average of inverter_1 and inverter_2 battery capacity
                                       Both inverters share the same battery bank (cluster config)
                                       so the average equals either individual reading.
                                       Template: {{ (states('number.inverter_1_battery_capacity')|int(0)
                                                   + states('number.inverter_2_battery_capacity')|int(0)) / 2 }}
                                       ⚠️ Must use |int(0) — sources go unavailable at startup/poll gaps.
                                       Entry ID: 01JKJES2XR88FXQ8TZHST4YA46
```

### Per-Inverter Sub-sensors (power_templates.yaml)

```
sensor.inverter_battery_voltage  V   (inv1_voltage via primary; inv2_voltage fallback)
sensor.inverter_battery_current  A
sensor.inverter_battery_capacity Wh  (31776 * SOC/100, rough estimate)
sensor.inverter_1_pv_voltage     V   max of pv1/pv2 strings (BUG: elif uses wrong entity)
sensor.inverter_2_pv_voltage     V   max of pv1/pv2 strings
sensor.inverter_grid_l1_current  A   grid line current
sensor.inverter_grid_l1_voltage  V   grid line voltage
sensor.inverter_load_l1_current  A
sensor.inverter_load_l1_voltage  V
sensor.inverter_1_pv1_power      W   string 1 power
sensor.inverter_1_pv2_power      W   string 2 power
sensor.inverter_2_pv1_power      W   string 3 power
sensor.inverter_2_pv2_power      W   string 4 power
```

### Temperature / Frequency (power_state.yaml)

```
sensor.inverter_ac_temperature   °C  max of both inverters
sensor.inverter_dc_temperature   °C  max of both inverters
sensor.inverter_grid_frequency   Hz  average of both inverters
sensor.inverter_load_frequency   Hz  average of both inverters
```

### Inverter Health (power_state.yaml)

```
sensor.inverter_health                     operational/degraded/offline
sensor.inverter_1_device_since_last_update seconds since last Solarman poll (MISSING — watchman)
sensor.inverter_2_device_since_last_update seconds since last Solarman poll (MISSING — watchman)
sensor.update_ticker                       1-min time trigger (state = formatted time)
```

### Solar Real-Time (solar_clipping.yaml, solar_state.yaml, solar_core.yaml)

```
sensor.solar_unused_power         W   solar produced but not used/charged/exported
sensor.solar_unused_power_smoothed W  30s debounced version
sensor.solar_opportunity_level        high/medium/low/none (based on unused+export)
sensor.solar_efficiency_loss_percent %  forecast vs actual (curtailment indicator)
sensor.solar_export_potential     W   export headroom (if export were enabled)
sensor.solar_system_status            active/low_output/not_generating
sensor.solar_ramp_rate            kW/min  1-min trigger rate-of-change
sensor.solar_utilisation_percent  %   actual/forecast ratio
sensor.solar_generation_confidence    high/medium/low
sensor.solar_stability                rapidly_improving/improving/stable/declining/collapsing
sensor.solar_window_remaining_hours h  hours of useful solar remaining today
sensor.solar_surplus_runtime_hours h  hours flex loads can run on surplus
binary_sensor.solar_surplus_available    on when export_potential > threshold
```

**Note:** `solar_core.yaml` defines BOTH `sensor.solar_surplus_available` and `binary_sensor.solar_surplus_available` with the same `unique_id: solar_surplus_available` — this is a duplicate unique_id conflict that will cause HA to ignore one of them.

### Load Shedding (load_shedding.yaml)

```
sensor.load_shedding_status_card      human-readable countdown text
binary_sensor.load_shedding_active    on when area sensor state = 'on'
sensor.load_shedding_severity         error/ok (for alert routing)
sensor.load_shedding_minutes_remaining min  numeric countdown (0 = inactive/unknown)
```

**Source integration entities:**
```
sensor.load_shedding_stage_eskom
sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark
```

### Load Power (power_helpers.yaml)

```
sensor.house_kitchen_power         W  (group.house_kitchen_power_sensors — unknown state)
sensor.house_living_areas_power    W  (group.house_living_areas_power_sensors — unknown)
sensor.house_laundry_power         W  (group.house_laundry_power_sensors — unknown)
sensor.house_entertainment_power   W  (group.house_entertainment_power_sensors — unknown)
sensor.house_outdoor_power         W  (group.house_outdoor_power_sensors — unknown)
sensor.house_security_power        W  (group.security_power_sensors — MISSING)
sensor.house_known_power           W  sum of group.known_power_loads
sensor.house_flexible_power        W  sum of group.flexible_power_loads
sensor.house_critical_power        W  sum of group.critical_power_loads
sensor.house_unknown_power         W  house_load - known_power (unmonitored load)
sensor.load_visibility_score       %  known_power / house_load_power
sensor.flexible_load_percent       %  flexible_power / house_load_power
sensor.house_outdoor_power_throttled W  10s debounce version of outdoor
```

---

## 6. Entity Reference — Energy & Prepaid

### Daily Energy Totals (power_core.yaml)

```
sensor.inverter_today_production          kWh  PV generated today
sensor.inverter_today_load_consumption    kWh  load consumed today
sensor.inverter_today_battery_charge      kWh  battery charged today
sensor.inverter_today_battery_discharge   kWh  battery discharged today
sensor.grid_energy_import_today           kWh  grid imported today
sensor.inverter_today_energy_export       kWh  exported today
sensor.inverter_today_energy_import       kWh  grid import today (inv1 * 2 WORKAROUND)
```

### Lifetime Energy Totals (power_core.yaml)

```
sensor.grid_energy_import_total           kWh  cumulative grid import (operational truth)
sensor.inverter_total_energy_export       kWh
sensor.inverter_total_pv_generation       kWh
sensor.inverter_total_load_consumption    kWh
```

### Energy Flow Breakdown (energy_core.yaml)

```
sensor.solar_to_house_power      W   solar power going directly to load
sensor.solar_to_battery_power    W   solar power going to battery
sensor.solar_to_grid_power       W   solar power exported (grid offset)
sensor.battery_to_house_power    W   battery discharge to load
sensor.grid_to_house_power       W   grid power to load
sensor.grid_to_battery_power     W   grid power to battery (charging)
sensor.house_power_losses        W   inverter/wiring losses

sensor.solar_used_by_house_today     kWh  (integration)
sensor.grid_used_by_house_today      kWh  (integration)
sensor.battery_used_by_house_today   kWh  (integration)
sensor.house_power_losses_today      kWh  (integration)

sensor.solar_self_consumption    %   solar used locally / solar generated
sensor.energy_independence       %   non-grid energy / total consumed
sensor.grid_dependency           %   grid power / total load
sensor.solar_coverage            %   solar / (load + losses)
sensor.battery_contribution      %   battery_to_house / total
```

### Utility Meters (energy_helpers.yaml + prepaid_core.yaml)

```
sensor.house_energy_daily            kWh  total house energy daily reset
sensor.house_energy_monthly          kWh  total house energy monthly reset
sensor.house_grid_energy_daily       kWh  grid contribution daily
sensor.house_grid_energy_monthly     kWh  grid contribution monthly
sensor.house_solar_energy_daily      kWh  solar contribution daily
sensor.house_solar_energy_monthly    kWh  solar contribution monthly
sensor.house_load_energy_daily       kWh  total load daily
sensor.house_load_energy_monthly     kWh  total load monthly

sensor.prepaid_import_daily          kWh  grid import daily (prepaid tracking)
sensor.prepaid_import_monthly        kWh  grid import monthly
sensor.prepaid_import_yearly         kWh  grid import yearly
sensor.solar_production_monthly      kWh  solar monthly (utility_meter on inverter_today_production)
sensor.solar_clipping_today          kWh  unused/clipped solar today (integration)
sensor.solar_clipping_month          kWh  monthly (utility_meter on solar_clipping_today)
sensor.solar_export_potential_today  kWh  (integration, if export were enabled)
sensor.solar_export_potential_month  kWh  (utility_meter)
```

**Note:** `energy_helpers.yaml` references `sensor.house_grid_energy`, `sensor.house_solar_energy`, `sensor.house_load_energy` as utility meter sources — confirm these are provided by `power_core.yaml` aggregation sensors or energy_core.yaml.

### Prepaid Financial Model (prepaid_core.yaml, prepaid_strategy.yaml)

```
sensor.prepaid_units_left_authoritative  kWh  calculated balance
sensor.prepaid_units_left_safe           kWh  authoritative OR manual (if drift > 1%)
sensor.prepaid_units_used_authoritative  kWh  consumption since last meter reading
sensor.prepaid_units_used_since_last_update kWh  incremental since last update
sensor.prepaid_grid_meter_drift          kWh  absolute drift (inverter vs meter)
sensor.prepaid_drift_percentage          %    drift as % of lifetime import
sensor.prepaid_last_update_date              when prepaid_units was last entered
sensor.prepaid_estimated_days_remaining  days  at adaptive burn rate
sensor.prepaid_rolling_30day_usage       kWh  rolling 30-day window (entity_id: prepaid_monthly_usage_true — NOT calendar month reset, use prepaid_import_monthly for that)

sensor.prepaid_energy_cost_per_kwh       R/kWh  last purchase cost
sensor.prepaid_true_cost_per_kwh         R/kWh  lifetime blended cost
sensor.prepaid_true_cost_per_kwh_monthly R/kWh  monthly blended cost
sensor.prepaid_spend_this_month          R      running spend this month

sensor.prepaid_reconciliation_status         aligned/minor_drift/significant_drift/critical_drift
sensor.prepaid_reconciliation_suggestion     human text
sensor.prepaid_suggested_offset              kWh  suggested new alignment_offset
sensor.prepaid_balance_confidence            %    0-100% drift-decay + stale-penalty confidence score
sensor.prepaid_drift_rate_per_day            kWh/day  drift accumulation rate since last realign (+ = meter ahead)

# Drift history (updated by script.prepaid_realign_offset)
input_number.prepaid_drift_at_last_realign   kWh  drift captured immediately before last realign click
input_datetime.prepaid_last_realign_time          timestamp of last realign click

# Diagnostic tool
script.prepaid_diagnostic                    dashboard button → pyscript.prepaid_diagnostic → persistent_notification

# Strategy sensors (prepaid_strategy.yaml)
sensor.prepaid_solar_savings_today       R    solar value vs grid cost
sensor.prepaid_solar_savings_monthly     R    solar value monthly
sensor.prepaid_fixed_charge_impact       %    fixed charges as % of total spend
sensor.prepaid_topup_strategy                critical_buy_now/buy_today/delay_topup/prepare/optimal
sensor.prepaid_depletion_date                ISO date string (BUG: references missing entity)
sensor.prepaid_buy_score                 0-100  composite buy trigger score
sensor.prepaid_buy_decision                    buy_now/buy_soon/delay/hold (BUG: automation checks 'hold' but produces 'delay')
sensor.prepaid_optimal_topup_size            R value (hardcoded brackets: 200/300/400/500/600)
sensor.prepaid_net_position_this_month   R    month position vs budget
sensor.prepaid_adaptive_burn_rate        kWh/day  rolling average
sensor.prepaid_rolling_burn_rate         kWh/day  shorter window
sensor.prepaid_daily_burn_avg_3d         kWh/day  statistics sensor (3-day mean)
```

### Battery (battery_runtime.yaml, battery_state.yaml)

```
sensor.ss_battery_capacity              Wh   program-aware target SOC × 31776
sensor.ss_soc_battery_time_left         s    seconds of runtime at current load
sensor.ss_soc_battery_time_left_friendly     human readable (Xh Ym)
sensor.battery_minutes_remaining_safe   min  conservative estimate
sensor.battery_runtime_status_card           text card for dashboard
sensor.battery_runtime_severity              critical/warning/information/normal
sensor.battery_runtime_confidence            high/medium/low/unavailable
binary_sensor.battery_runtime_unreliable     on when confidence = low/unavailable
sensor.battery_state_health                  strong/healthy/low/critical (% thresholds: 70/40/25)
```

---

## 7. Entity Reference — Strategy & Decision Layer

```
# Power health (power_state.yaml)
sensor.house_power_health     grid_stable/solar_powered/battery_resilient/battery_risk/critical
sensor.power_state            grid/solar/battery_safe/battery_risk/battery_critical
sensor.house_power_source     grid/solar/battery  (dominant source)
sensor.power_flow_state       solar_only/solar_and_battery/grid_only/battery_only/offgrid_critical

# Power strategy (power_strategy.yaml)
sensor.power_strategy         solar_surplus/solar_ok/battery_guard/battery_critical/grid_only/normal
sensor.power_strategy_status  human readable strategy text
sensor.power_strategy_severity critical/warning/opportunity/normal

# Energy orchestrator (energy_state.yaml)
sensor.energy_orchestrator_state        run_heavy_loads/run_medium_loads/reduce_load/conserve_energy/normal
sensor.energy_economy_score             0-100
sensor.house_energy_resilience_hours    h  how long house can run on current energy
sensor.house_energy_resilience_status       text description
sensor.energy_loss_percent_today        %
sensor.energy_loss_root_cause               text diagnosis
sensor.power_loss_root_cause_live           live version
sensor.energy_loss_state                    ok/minor_loss/significant_loss/major_loss
sensor.battery_night_survival               BROKEN (references missing entities)
sensor.solar_sufficiency_tomorrow           based on solcast forecast
binary_sensor.grid_charging_while_solar     BROKEN (references missing entities)
sensor.grid_charging_while_solar            BROKEN (references missing entities)
sensor.energy_model_integrity               ok/warning/error (consistency check)

# Grid risk (grid_risk.yaml)
sensor.grid_risk_level        unknown/critical/warning/moderate/safe
sensor.grid_risk_status       emoji + text status
sensor.grid_dependency_eta    h  estimated time until critical grid dependency
sensor.grid_risk_severity     normal/critical/warning/information

# Grid state (grid_state.yaml)
sensor.grid_state_health      stable/risk/unstable/offline

# Battery state (battery_state.yaml)
sensor.battery_state_health   strong/healthy/low/critical

# Rolling averages & weather correlation (power_statistics.yaml — added 2026-04-22)
sensor.inverter_production_7d_mean    kWh  7-day rolling mean of daily PV production
sensor.inverter_production_30d_mean   kWh  30-day rolling mean of daily PV production
sensor.house_load_24h_mean            W    24h rolling mean load power
sensor.solar_vs_forecast_ratio_7d     %    7d mean actual vs today's Solcast forecast
sensor.solar_weather_correlation           normal/degraded — degraded when ratio < 70%
# Source entities: inverter_today_production, inverter_load_power, solcast_pv_forecast_forecast_today
# All five sensors are excluded from the recorder.

# Solar decision (solar_forecast.yaml)
sensor.solar_forecast_available_conservative  kWh  conservative day forecast
sensor.solar_max_power_safe                   W    (hardcoded: 8640 W)
sensor.solar_remaining_energy_safe_wh         Wh
sensor.solar_power_available_safe             W
sensor.solar_confidence                        high/medium/low
sensor.solar_usable_power_max                 W
sensor.solar_forecast_next_hour               W    this_hour + remaining/6
sensor.solar_power_available_conservative     W    next_hour * 0.7
sensor.solar_window_remaining_hours           h
```

---

## 8. Entity Reference — Input Helpers

### Prepaid Controls (prepaid_helpers.yaml)

```
input_number.prepaid_units                  current prepaid balance (kWh, from meter display)
input_number.prepaid_units_left             LEGACY — appears missing, referenced in strategy (BUG)
input_number.initial_prepaid_grid_import    grid import at time of last meter reading (baseline)
input_number.prepaid_meter_lifetime_import  physical meter reading (kWh, authoritative)
input_number.prepaid_daily_burn_average     expected daily usage (kWh)
input_number.prepaid_minimum_buffer_kwh     minimum balance buffer
input_number.prepaid_critical_days_remaining  alert threshold (days)
input_number.prepaid_warning_days_remaining   warning threshold (days)
input_number.prepaid_alignment_offset       manual reconciliation offset (kWh)
input_number.prepaid_last_topup_units       units in last purchase
input_number.prepaid_last_topup_cost        cost of last purchase (R)
input_number.prepaid_last_topup_fixed_deducted  fixed charge deducted from last topup
input_number.prepaid_last_topup_discount    discount received
input_number.prepaid_total_spent            running total spend this month (R)
input_number.prepaid_total_units            running total units purchased
input_number.prepaid_total_fixed_cost_paid  total fixed charges paid
input_number.prepaid_total_discount         total discounts received
input_number.grid_tariff_nominal            Eskom tariff (R/kWh) for comparison
input_datetime.prepaid_notify_start         quiet hours start for prepaid alerts
input_datetime.prepaid_notify_end           quiet hours end for prepaid alerts
```

### Grid Thresholds (power_helpers.yaml via power package)

```
input_number.grid_import_usage_low_warning_trigger
input_number.grid_import_usage_medium_warning_trigger
input_number.grid_import_usage_high_warning_trigger
input_number.inverter_battery_soc_warning_trigger
```

### Solar Controls (solar_helpers.yaml)

```
input_number.solar_surplus_threshold    W  threshold for surplus opportunity (default: 800)
input_number.solar_export_tariff        R/kWh  notional export value (default: 1.30)
```

### Solar Available Surplus (power_state.yaml — added 2026-04-22)

Single shared metric for all optional-load decisions (pool pump, geyser, borehole, etc.).

```
sensor.solar_available_surplus    W   pv_power - load_power
```

**Why one sensor:** When any optional load is running, its draw is already inside `inverter_load_power`,
so surplus automatically decreases — no double-counting. Each appliance automation checks this sensor
against its own threshold:
- Pool pump: turn on >1200W, turn off <800W (hysteresis; ~700W pump + 500W buffer)
- Geyser: ~3000W threshold (pending)
- Borehole: threshold TBD (pending)

Does NOT subtract battery charging. In self-consumption mode battery charging is an outcome of surplus,
not a cost that reduces it. `solar_export_potential` (which does subtract battery charging) is NOT the
right metric — it reads near-zero while the battery fills all day.

### Pool Pump Control (power_helpers.yaml + power_state.yaml — added 2026-04-22)

The pool pump is solar-managed by `pool_pump_solar_control` in `power_automations.yaml`.
The enable gate (`input_boolean.load_control_pool_enabled`) is defined in `load_control.yaml`.
Switch entity: `switch.pool_pump_switch` (renamed from `switch.pool_pump_switch_1` 2026-04-22).

**Helpers:**
```
input_number.pool_minimum_run_minutes       min  min run time before turn-off allowed (default: 45)
input_number.pool_pre_shed_battery_threshold %   SOC threshold for pre-shed gate (default: 80)
input_number.pool_target_hours_summer       h    daily run target in summer (default: 4)
input_number.pool_target_hours_winter       h    daily run target in spring/autumn/winter (default: 1.5)
input_datetime.pool_pump_last_on                 time-only — set by automation on pump start
```

**Runtime sensors:**
```
sensor.pool_pump_run_hours_today          h (float)  UI Helper — History Stats (NOT in YAML)
                                                       history_stats fails in packages when source
                                                       entity is newly registered at load time.
                                                       Create via: Settings → Helpers → History Stats
                                                       Entity: switch.pool_pump_switch, State: on,
                                                       Type: Time, Start: midnight today, End: now
sensor.pool_pump_continuous_run_minutes   min         elapsed time since last pump start
sensor.pool_target_run_hours_today        h           season-aware target (summer vs other)
sensor.pool_pump_control_status           string      human-readable reason why pump is/isn't running
```

**Control logic summary:**
- Active window: 08:00–16:00. Gate: `input_boolean.load_control_pool_enabled = on`.
- **Turn off (16:00 hard stop — Branch 0)**: unconditional shutdown at end of solar window — no minimum run time guard. Fires at exactly 16:00:00.
- **Turn on** when: `solar_available_surplus > 1200W`, battery not critical, stage < 2, daily target not met, no upcoming shed with low SOC. Gated `before: "16:00:00"` on Branch 5.
- **Turn off (load shedding)**: stage >= 2, or upcoming shed within 3h AND SOC < threshold — enforces minimum run time.
- **Turn off (solar dropped)**: `solar_available_surplus < 800W` (hysteresis lower band) or battery low/critical — enforces minimum run time.
- **Turn off (target met)**: daily run hours >= season target — enforces minimum run time.
- Minimum run time guards prevent short cycles (pump must run `pool_minimum_run_minutes` before any turn-off). Does NOT apply to the 16:00 hard stop.
- Season logic: `sensor.season == 'summer'` → summer target; all other seasons → winter target.
- Load shedding inputs: `state_attr('sensor.load_shedding_stage_eskom', 'stage')` (int 0–8) and `starts_in` attribute on load shedding area sensor.
- All notifications via `script.notify_power_event` (severity: information, subsystem: energy) — routes to HA app + Telegram mirror.
- Hourly default branch fires when pv > 500W and pump is off — Telegram trace of why pump didn't run.
- `mode: queued` (changed from `single` 2026-04-28) — ensures Branch 1 (records `pool_pump_last_on`) is not dropped when Branch 5 turns on the pump.

**Replaced automations (removed 2026-04-22):**
```
1742227789739 — Pool Pump Turn Off Daily PM
1742294033785 — Pool Pump Turn on Early - Daily AM 9-1pm
1742294477609 — Pool Pump Turn on - Daily AM 11-1pm
1742295582190 — Pool Pump Turn on - Daily PM 1-4pm
```

**Missing solar helpers** (referenced in solar_forecast.yaml but not in solar_helpers.yaml):
```
input_number.low_solar_forecast_trigger     kWh threshold for Low solar scene
input_number.high_solar_forecast_trigger    kWh threshold for High solar scene
input_select.inverter_solar_mode_helper     Low/Medium/High solar mode selector
```

### Inverter Program Awareness (battery_runtime.yaml reads)

```
sensor.inverter_1_program_1_charge_power_percent   %  (or similar Solarman register)
sensor.inverter_1_program_2_charge_power_percent   %
... (program-specific target SOC — read by battery_runtime.yaml)
```

---

## 9. Data Flow Maps

### Prepaid Balance Update Flow

```
User reads physical meter → enters value into:
  input_number.prepaid_meter_lifetime_import
  input_number.prepaid_units (display balance)
  input_number.initial_prepaid_grid_import (grid import at reading time)
                    │
                    ▼
  sensor.prepaid_units_used_authoritative =
    grid_energy_import_total + alignment_offset - initial_prepaid_grid_import
                    │
                    ▼
  sensor.prepaid_units_left_authoritative =
    prepaid_meter_lifetime_import - prepaid_units_used_authoritative
                    │
                    ▼
  sensor.prepaid_drift_percentage =
    (grid_import_total - prepaid_meter_lifetime_import) / prepaid_meter_lifetime_import × 100
                    │
                  drift > 1%?
                  /         \
           YES               NO
            │                 │
            ▼                 ▼
  prepaid_units_left_safe  prepaid_units_left_safe
    = prepaid_units           = authoritative
    (manual fallback)
```

### Buy Score Calculation Flow

```
Inputs:
  prepaid_estimated_days_remaining
  prepaid_units_left_safe
  solar_forecast_available_conservative
  current time, time of month

  → prepaid_topup_strategy  (critical/buy_today/delay/prepare/optimal)
  → prepaid_buy_score       (0-100 composite)
  → prepaid_buy_decision    (buy_now/buy_soon/delay/hold)

When buy_score crosses threshold →
  prepaid_buy_decision_notify automation → script.notify_power_event
  BUG: condition checks 'hold' but sensor produces 'delay'
```

### Solar Mode Automation Flow (solar_forecast.yaml)

```
Every 3 hours (time_pattern trigger) →
  Read sensor.solar_forecast_available_conservative
  Compare vs input_number.low_solar_forecast_trigger / high_solar_forecast_trigger
    < low_trigger  → activate scene.inverter_solar_mode_low
    < high_trigger → activate scene.inverter_solar_mode_medium
    ≥ high_trigger → activate scene.inverter_solar_mode_high
  Sets input_select.inverter_solar_mode_helper

NOTE: low/high_solar_forecast_trigger helpers NOT defined in solar_helpers.yaml (they must
exist elsewhere or are missing — watchman would flag them)
```

### Energy Accounting

```
House Load = Solar to House + Battery to House + Grid to House
Solar Total = Solar to House + Solar to Battery + Solar to Grid + Unused Solar
Battery Charge = Solar to Battery + Grid to Battery
battery_charge_split_error = battery_charge_energy_today - (solar_to_batt + grid_to_batt)
```

---

## 10. Cross-Domain Interface

### Power Domain → Outputs Consumed by Other Domains

| Entity | Consumer Domain | Consuming Entity/File |
|---|---|---|
| `sensor.inverter_battery_soc` | Water | `binary_sensor.water_refill_allowed` |
| `sensor.house_power_health` | Alerts | `packages/alerts/alerts_power.yaml` |
| `sensor.power_state` | Lighting | energy save mode automation |
| `sensor.energy_orchestrator_state` | Lighting | load scheduling automations |
| `binary_sensor.load_shedding_active` | Alerts | power alerts |
| `sensor.load_shedding_minutes_remaining` | Automations | water refill safety |
| `group.inverter_grid` | Legacy automations.yaml | grid_status_monitoring |
| `group.flexible_power_loads` | Alerts | `alerts_power.yaml` |

### Power Domain ← Inputs Consumed from Other Domains

| External Entity | Source Domain | Used In |
|---|---|---|
| `sensor.solcast_pv_forecast_forecast_today` | Solcast (custom_component) | `solar_forecast.yaml` |
| `sensor.load_shedding_stage_eskom` | Load Shedding integration | `load_shedding.yaml` |
| `sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark` | Load Shedding | `load_shedding.yaml`, `prepaid_strategy.yaml` |
| `binary_sensor.anyone_connected_home` | Presence | `energy_orchestrator_state` |
| `binary_sensor.night_confirmed` | Context | `battery_runtime.yaml`, `grid_risk.yaml` |

### Legacy automations.yaml Power Automations

These exist in the legacy file and should eventually be migrated to package files:

| Automation ID | Purpose | Status |
|---|---|---|
| `grid_status_monitoring` | Grid disconnect/reconnect notifications | Active but uses `sensor.inverter_power` (wrong name, should be `sensor.inverter_load_power`) |
| `inverter_pwer_monitoring` | Load/battery overload alerts | Active, same wrong entity |
| Load Shedding Inverter Scene Switcher | Changes inverter scene based on LS level | Active — see if duplicated by `solar_forecast.yaml` automation |

All legacy automations use `sensor.inverter_power` which does NOT exist — the correct entity is `sensor.inverter_load_power`. These notifications are silently broken.

---

## 11. Known Issues

Issues ordered by risk/impact. **P1 = breaks functionality. P2 = incorrect data. P3 = performance/cleanup.**

---

### Issue 1 — BROKEN: `battery_night_survival` References Non-Existent Entities
**Priority:** P1 — Sensor always unavailable  
**File:** `packages/power/energy_state.yaml:171`  
**Root cause:** References `sensor.battery_energy_available` and `sensor.average_night_consumption` — neither exists in any package file. These were never implemented or were removed.  
**Watchman:** Both confirmed missing.  
**Fix:** Either implement the underlying sensors (battery energy available = SOC% × 31776 Wh; average night consumption = statistics sensor over night hours) or remove this sensor and dashboard references.  
**Risk if left:** Always shows unavailable; dashboard cards that depend on it will fail.

---

### Issue 2 — ✅ RESOLVED 2026-04-21: `grid_charging_while_solar` + `grid_to_house_power` mis-classified grid import
**Priority:** P1 — Sensor misfired; live and daily grid→house sensors overcounted  
**File:** `packages/power/energy_state.yaml`, `packages/power/energy_core.yaml`  
**Root cause (2026-04-16 fix):** entity name errors fixed. Root cause (2026-04-21 fix): `grid_to_house_power` assigned ALL grid import to house instead of subtracting battery charge; `grid_charging_while_solar` fired on any `grid > 250` regardless of whether grid was actually charging the battery; `grid_used_by_house_today` returned raw `grid_energy_import_today` including battery charging kWh; `solar_to_battery_energy_today` and `grid_to_battery_energy_today` were missing (commented-out), making `solar_used_by_house_today` overcount and `house_power_losses_today` inflate by the battery charge amount.  
**Fix applied 2026-04-21:**  
- `grid_to_house_power` → `max(0, grid - grid_to_battery_power)` (subtracts battery charge component)  
- `grid_charging_while_solar` detection → `solar > 1000 and grid_to_battery_power > 200` (requires actual battery charging)  
- Added `sensor.solar_to_battery_energy_today` and `sensor.grid_to_battery_energy_today` using inverter-native today totals (restart-safe)  
- `grid_used_by_house_today` → `grid_energy_import_today - grid_to_battery_energy_today`  
- Notification for grid_charging_while_solar now includes solar/grid/load/SOC values

---

### Issue 3 — ✅ RESOLVED 2026-04-16: `prepaid_depletion_date` References Legacy Entity
**Priority:** P1 — Sensor always unavailable  
**File:** `packages/power/prepaid_strategy.yaml:140`  
**Root cause:** Referenced `sensor.prepaid_units_left` (removed entity) and `input_number.prepaid_units_left` (orphan helper).  
**Fix applied:** `sensor.prepaid_units_left` → `sensor.prepaid_units_left_safe` in depletion date template; `input_number.prepaid_units_left` removed from `prepaid_helpers.yaml`.

---

### Issue 4 — ✅ RESOLVED 2026-04-16: `prepaid_buy_decision_notify` Condition Never Matches
**Priority:** P1 — Buy notification automation never fires  
**File:** `packages/power/prepaid_strategy.yaml` (automation block)  
**Root cause:** Condition checked `trigger.to_state.state in ['buy_now','plan_topup','hold']` — `'hold'` is not a valid sensor state.  
**Fix applied:** Condition updated to `trigger.to_state.state in ['buy_now', 'buy_soon']` — fires only on actionable buy states.

---

### Issue 5 — ✅ RESOLVED 2026-04-16: Duplicate unique_id in `solar_core.yaml`
**Priority:** P2 — One of the two sensors silently ignored by HA  
**File:** `packages/power/solar_core.yaml`  
**Root cause:** Both `sensor.solar_surplus_available` and `binary_sensor.solar_surplus_available` shared `unique_id: solar_surplus_available`.  
**Fix applied:** Sensor unique_id changed to `solar_surplus_available_power`; binary_sensor retains `solar_surplus_available`. Both entities now register correctly.

---

### Issue 6 — ✅ RESOLVED 2026-04-16: `inverter_1_pv_voltage` Uses Wrong Entity in elif Branch
**Priority:** P2 — Falls through to wrong value when inv1_pv1_voltage is unavailable  
**File:** `packages/power/power_helpers.yaml` (contract incorrectly listed power_templates.yaml)  
**Root cause:** elif branch referenced `states('sensor.inverter_pv1_voltage')` — the combined cross-inverter sensor instead of the inv1-specific entity.  
**Fix applied:** `sensor.inverter_pv1_voltage` → `sensor.inverter_1_pv1_voltage` in the elif fallback branch.

---

### Issue 7 — DATA QUALITY: `sensor.inverter_today_energy_import` Uses Doubling Workaround
**Priority:** P2 — Structural data quality issue  
**File:** `packages/power/power_core.yaml`  
**Root cause:** Slave inverter (inv2) doesn't reliably report grid import. Workaround multiplies inv1 value by 2. If inv2 starts reporting, the doubling produces 2× overcounting.  
**Fix needed:** Add detection logic — if inv2 value > 0, use actual sum instead of doubling. Or add a config flag `input_boolean.use_grid_import_doubling` to allow easy toggle.

---

### Issue 8 — MISSING: `group.security_power_sensors`
**Priority:** P2 — `sensor.house_security_power` always returns 0/unavailable  
**File:** `packages/power/power_helpers.yaml`  
**Root cause:** The group is referenced in templates but was never defined in any package file. Watchman confirms missing.  
**Fix:** Either create the group with appropriate security device power sensors, or remove `sensor.house_security_power` if security load monitoring is not needed.

---

### Issue 9 — MISSING: Solar forecast helper entities for inverter mode automation
**Priority:** P2 — `solar_forecast_update_inverter_mode_daily` automation uses undefined thresholds  
**File:** `packages/power/solar_forecast.yaml`  
**Root cause:** `input_number.low_solar_forecast_trigger` and `input_number.high_solar_forecast_trigger` not defined in `solar_helpers.yaml` or anywhere else. `input_select.inverter_solar_mode_helper` also not confirmed defined.  
**Fix:** Add the missing helpers to `solar_helpers.yaml`:
```yaml
input_number:
  low_solar_forecast_trigger:
    name: Low Solar Forecast Trigger
    min: 0
    max: 50
    step: 0.5
    initial: 10
    unit_of_measurement: kWh
  high_solar_forecast_trigger:
    name: High Solar Forecast Trigger
    min: 0
    max: 50
    step: 0.5
    initial: 25
    unit_of_measurement: kWh
input_select:
  inverter_solar_mode_helper:
    name: Inverter Solar Mode
    options: [Low, Medium, High]
```

---

### Issue 10 — STALE: Wrong Solcast entity name
**Priority:** P2 — Solar forecast sensors always unavailable  
**File:** `packages/power/solar_forecast.yaml:36`  
**Root cause:** References `sensor.solcast_forecast_forecast_today` but the Solcast custom component produces `sensor.solcast_pv_forecast_forecast_today`.  
**Watchman:** Confirmed missing.  
**Fix:** Replace `solcast_forecast_forecast_today` → `solcast_pv_forecast_forecast_today` throughout `solar_forecast.yaml`.

---

### Issue 11 — LEGACY: `automations.yaml` uses `sensor.inverter_power` (wrong entity)
**Priority:** P2 — Grid disconnect notifications silently broken  
**File:** `automations.yaml:308, 319, 399, 413 (and many more)`  
**Root cause:** Legacy automations reference `sensor.inverter_power` which was renamed to `sensor.inverter_load_power`. All grid/load notifications report empty power values.  
**Fix:** Either update legacy automations or migrate them to package files with correct entity names. Use `sensor.inverter_load_power` everywhere.

---

### Issue 12 — BUG: `load_shedding_minutes_remaining` unique_id Has Typo
**Priority:** P3 — Functional but cannot be referenced by unique_id  
**File:** `packages/power/load_shedding.yaml:84`  
**Root cause:** `unique_id: load_hedding_inutes_emaining` — three letters missing (`s`, `m`, `r`). The entity works but HA's entity registry stores it under the wrong unique_id, making UI customisation of this entity unreliable.  
**Fix:** Correct unique_id to `load_shedding_minutes_remaining`. Note: changing a unique_id will create a new entity entry — update any entity customisations and dashboard references after fixing.

---

### Issue 13 — BUG: `grid_risk.yaml` uses `sensor.inverter_power` (wrong entity)
**Priority:** P2 — grid_risk sensors degrade silently when reading wrong/missing entity  
**File:** `packages/power/grid_risk.yaml`  
**Root cause:** Dependency analysis confirms `grid_risk.yaml` references `sensor.inverter_power` in its logic (same wrong-name pattern as legacy automations.yaml). The correct entity is `sensor.inverter_load_power`. This means grid risk calculations that incorporate load power are reading an unavailable entity and falling back to `float(0)`.  
**Fix:** Audit all `states('sensor.inverter_power')` calls in `grid_risk.yaml` and replace with `sensor.inverter_load_power`.

---

### Issue 15 — ✅ RESOLVED 2026-04-16: `sensor.prepaid_total_spent`
**Priority:** P3 — Dashboard shows unavailable  
**Root cause:** Dashboard referenced `sensor.prepaid_total_spent` which doesn't exist. The correct entity is `input_number.prepaid_total_spent` (defined in prepaid_helpers.yaml).  
**Fix applied:** Two dashboard references in `lovelace.dashboard_operations` (Energy Spend + Spend traces in plotly-graph cards) updated to `input_number.prepaid_total_spent`.

---

### Issue 16 — UNKNOWN STATE: Load groups
**Priority:** P3 — Power templates that sum group states show unknown  
**File:** `packages/power/power_helpers.yaml` and `power_templates.yaml`  
**Root cause:** `group.known_power_loads`, `group.flexible_power_loads`, `group.critical_power_loads` all show state "unknown" per watchman. These are populated by `pyscript/sync_power_groups.py` — if pyscript is not running or hasn't executed, groups are empty.  
**Fix:** Ensure pyscript runs on HA startup. Add a startup trigger. Monitor that pyscript completes successfully.

---

### Issue 17 — DESIGN DEBT: `power_helpers.yaml` violates layering convention
**Priority:** P3 — Maintenance clarity  
**Root cause:** File contains both `group:` definitions (helpers layer role) and `template: sensor:` blocks (templates layer role). Violates the `*_helpers.yaml` → only helpers rule from CODING_STANDARDS.md.  
**Fix (non-urgent):** Split into `power_groups.yaml` (group definitions) + merge templates into `power_templates.yaml`. Do not do this unless refactoring — splitting risks breaking references.

---

## 12. Error Signatures (Watchman-Confirmed)

These entities appear in watchman_report.txt as missing or unavailable. Map these to the issues above.

### Missing Entities (will produce errors in HA logs)

| Entity | Status | References | Issue |
|---|---|---|---|
| `sensor.battery_energy_available` | missing | energy_state.yaml:171 | Issue 1 |
| `sensor.average_night_consumption` | ⚠️ BUG: sampling_size:20 → ~100s window not 24h | energy_state.yaml | BUG-PWR-AVG01: needs throttled 5-min intermediary + sampling_size:288 |
| `sensor.grid_power` | ✅ FIXED 2026-04-16 | replaced with sensor.house_grid_power | Issue 2 |
| `sensor.battery_charge_power` | ✅ FIXED 2026-04-16 | replaced with sensor.grid_to_battery_power | Issue 2 |
| `sensor.prepaid_units_left` | ✅ FIXED 2026-04-16 | replaced with sensor.prepaid_units_left_safe | Issue 3 |
| `group.security_power_sensors` | missing | power_templates.yaml:182 | Issue 8 |
| `sensor.solcast_forecast_forecast_today` | missing | solar_forecast.yaml:36 | Issue 10 |
| `sensor.inverter_1_device_since_last_update` | missing | power_state.yaml:279 | Solarman entity name change |
| `sensor.inverter_2_device_since_last_update` | missing | power_state.yaml (similar) | Solarman entity name change |
| `sensor.prepaid_total_spent` | ✅ FIXED 2026-04-16 | dashboard now references input_number.prepaid_total_spent | Issue 15 |
| `sensor.inverter_2_pv3_voltage` | missing | dashboard | PV3/4 strings don't exist on inv2 |
| `sensor.inverter_2_pv4_voltage` | missing | dashboard | PV3/4 strings don't exist on inv2 |

### Unknown State Entities (group state not resolved)

| Entity | Status | Root Cause |
|---|---|---|
| `group.known_power_loads` | unknown | pyscript not populating groups |
| `group.flexible_power_loads` | unknown | pyscript not populating groups |
| `group.critical_power_loads` | unknown | pyscript not populating groups |
| `group.inverter_battery_state` | unknown | solarman group entity issue |
| `group.house_kitchen_power_sensors` | unknown | no entities in group? |
| `group.house_living_areas_power_sensors` | unknown | no entities in group? |
| `group.house_laundry_power_sensors` | unknown | no entities in group? |
| `group.house_entertainment_power_sensors` | unknown | no entities in group? |
| `group.house_outdoor_power_sensors` | unknown | no entities in group? |

### Dashboard-Only Missing Entities (not referenced in package files)

| Entity | Dashboard | Notes |
|---|---|---|
| `counter.power_critical_suppres` | dashboard_home | probably a typo; power alert suppression counter |
| `input_datetime.last_power_crit` | dashboard_home | power critical last-seen tracker |
| `number.inverter_1_program_1_ch` | dashboard_operations | Solarman program register may be named differently |
| `sensor.house_outdoor_power_fil` | dashboard_overview | probably `sensor.house_outdoor_power_throttled` with truncated name |

### Legacy automations.yaml Missing Entities

```
sensor.ssa_battery_soc_time_left     # legacy sensor, replaced by ss_soc_battery_time_left
sensor.ss_essential_power            # never implemented
sensor.ss_battery_power              # replaced by inverter_battery_power
sensor.markdown_ssa_battery_charge_time_left  # legacy markdown sensor
sensor.markdown_ssa_battery_discharge_time    # legacy markdown sensor
sensor.ssa_battery_capacity_shedding          # legacy
sensor.markdown_ssa_battery_discharge_time    # legacy
```

These are all from `automations.yaml:770` — a single legacy automation referencing many replaced sensors. Low priority to fix as long as the package-file equivalents work.

---

## 13. Optimization Recommendations

### Recommendation 1 — Fix broken strategy sensors first (Issues 1-4)
The buy decision notification (Issue 4), battery night survival (Issue 1), and grid charging detection (Issue 2) are strategy layer features that are entirely non-functional. These should be fixed before any new strategy features are added, as they create false confidence that the system is monitoring things it isn't.

### Recommendation 2 — Add Confidence Scoring to Prepaid Balance
`POWER_CONTEXT.md` already identifies this as next step. The current binary switch (drift > 1% → use manual) is fragile. A graduated confidence score (0-100%) would allow dashboard UI to reflect uncertainty without hard switching. Implement as `sensor.prepaid_balance_confidence` before adding new prepaid features.

### Recommendation 3 — Implement Buy Score v2 with Net Position
`sensor.prepaid_buy_score` currently uses a heuristic composite. The `sensor.prepaid_net_position_this_month` sensor exists but is unavailable. Fix and incorporate net position into the buy score formula — a negative net position month should lower the score even when days remaining are high.

### Recommendation 4 — Replace `inverter_today_energy_import` Workaround
The `* 2` doubling is a time bomb. When Inverter 2 Solarman polling stabilizes, energy import will double overnight. Add a conditional:
```yaml
{% set inv2 = states('sensor.inverter_2_today_energy_import') | float(-1) %}
{% if inv2 > 0 %}
  {{ inv1 + inv2 }}
{% else %}
  {{ inv1 * 2 }}
{% endif %}
```
And add a watchdog sensor that alerts when inv2 suddenly starts reporting.

### Recommendation 5 — Migrate legacy grid automations
`automations.yaml` grid monitoring automations are silently broken (`sensor.inverter_power` doesn't exist). The package files already have better state sensors (`sensor.house_power_health`, `sensor.grid_state_health`). Migrate `grid_status_monitoring` and `inverter_pwer_monitoring` to package files using the central notify script pattern.

### Recommendation 6 — Auto-reconciliation trigger
When `prepaid_drift_percentage` exceeds threshold and user has entered a new meter reading, automatically trigger `script.prepaid_realign_offset` and notify. Currently the reconciliation suggestion is shown in UI but never auto-applied — the script exists, just needs an automation trigger.

### Recommendation 7 — pyscript load group health check
Add a binary_sensor that checks if `group.known_power_loads` has any members. If it's empty (unknown), trigger the pyscript to re-sync. Current state: if pyscript fails silently at startup, all load visibility sensors return 0/unknown indefinitely.

---

## 14. Implementation Checklist

### Sprint 1 — Fix Silent Failures (Issues 1-5)

- [ ] Fix `battery_night_survival` — implement `sensor.battery_energy_available` and `sensor.average_night_consumption` OR remove and clean dashboard references
- [ ] Fix `grid_charging_while_solar` — correct entity names: `house_grid_power`, `grid_to_battery_power`
- [ ] Fix `prepaid_depletion_date` — replace `sensor.prepaid_units_left` → `sensor.prepaid_units_left_safe`
- [ ] Fix `prepaid_buy_decision_notify` automation — condition `'hold'` → `'delay'` (or match actual output states)
- [ ] Fix duplicate unique_id `solar_surplus_available` in solar_core.yaml — rename sensor version

### Sprint 2 — Fix Wrong Entities (Issues 6-13)

- [ ] Fix `inverter_1_pv_voltage` elif branch: `sensor.inverter_pv1_voltage` → `sensor.inverter_1_pv1_voltage`
- [ ] Add conditional to `inverter_today_energy_import` instead of `* 2`
- [ ] Create `group.security_power_sensors` or remove `sensor.house_security_power`
- [ ] Add missing solar helpers: `low_solar_forecast_trigger`, `high_solar_forecast_trigger`, `input_select.inverter_solar_mode_helper`
- [ ] Fix Solcast entity: `solcast_forecast_forecast_today` → `solcast_pv_forecast_forecast_today`
- [ ] Verify `sensor.inverter_1_device_since_last_update` — check actual Solarman entity name
- [ ] Fix `load_shedding_minutes_remaining` unique_id typo: `load_hedding_inutes_emaining` → `load_shedding_minutes_remaining`
- [ ] Audit `grid_risk.yaml` for all `sensor.inverter_power` references → replace with `sensor.inverter_load_power`

### Sprint 3 — Legacy Cleanup

- [ ] Migrate `grid_status_monitoring` from automations.yaml to `packages/power/power_automations.yaml` — fix entity names
- [ ] Migrate `inverter_pwer_monitoring` similarly
- [ ] Update dashboard references to use `input_number.prepaid_total_spent` instead of `sensor.prepaid_total_spent`
- [ ] Remove/comment stale `ssa_*` sensor references from automations.yaml

### Sprint 4 — Strategy Improvements

- [ ] Implement `sensor.prepaid_balance_confidence` (graduated confidence model)
- [ ] Implement `sensor.prepaid_net_position_this_month` (ensure it's available not just defined)
- [ ] Build Buy Score v2 with net position weighting
- [ ] Add auto-reconciliation trigger when drift exceeds threshold + meter reading entered
- [ ] Add pyscript load group health check + restart trigger

---

## Locked Design Decisions

These decisions are intentional and must NOT be changed without explicit review:

| Decision | Rationale |
|---|---|
| Battery SOC = average of both inverters | Both inverters share the same battery bank; avg is the correct representation |
| Inverter 1 = grid side, Inverter 2 = PV/load side | Physical wiring — do not swap |
| Prepaid balance uses usage-based model (not top-up based) | Top-up based double-counted purchases; usage model is accurate |
| `prepaid_units_left_safe` switches to manual when drift > 1% | Manual meter reading is authoritative; inverter cumulative drifts and resets |
| `solar_max_power_safe` hardcoded at 8640 W (not 10.8 kW) | Safety margin; actual achievable peak is below nameplate capacity |
| Energy flow sensors use `max(0, x)` clamps | Prevents negative flow values from contaminating integration sensors |
| Battery capacity = 31,776 Wh (not 31,800) | Physical measurement; do not adjust to round number |
| `inverter_battery_power` positive = discharge, negative = charge | Consistent with Solarman sign convention — do not invert |
| `house_grid_power` positive = import, negative = export | Standard HA convention |
| Load groups populated by pyscript, not static YAML | Labels-based dynamic grouping allows easy device reclassification |
| `sensor.prepaid_units_left_safe` is the ONLY entity safe to display | Authoritative and manual both feed into it; never display raw inverter grid totals directly |

---

---

## 15. Package Architecture & Dependency Analysis

> Source: `POWER_DEPENDENCY_ANALYSIS.md` — incorporated 2026-04-13.

### Key Architecture Finding

The power package is a **tightly coupled DAG** (directed acyclic graph — no circular dependencies). `power_core` is the single linchpin: every other subsystem has a direct or transitive dependency on it. `load_shedding` is the only fully independent subsystem.

### Subsystem Dependency Matrix

| Subsystem | Consumes From | Produces For |
|---|---|---|
| **power_core** | Hardware only (`inverter_1_*`, `inverter_2_*` via solarman) | Everything else — 50+ aggregated entities |
| **power_helpers** | power_core | power_state, energy_helpers, power_strategy (group.inverter_grid + load groups) |
| **power_templates** | power_core (inverter_load_power), itself (load groups) | energy_helpers (known/flexible/critical load power), dashboards |
| **power_state** | power_core (SOC, grid status), battery_runtime (confidence, minutes_safe), solar_state (system_status, confidence, stability, power_available_conservative) | house_power_health, power_state, power_flow_state → automations |
| **power_strategy** | power_core (inverter sensors), solar_clipping (solar_export_potential), power_helpers (group.inverter_grid) | power_strategy, power_strategy_severity → automations, conditions |
| **battery_runtime** | power_core (SOC, battery_power), power_helpers (group.inverter_grid), hardware (program registers) | battery_runtime_severity, battery_minutes_remaining_safe → power_state, grid_risk, energy_state |
| **battery_state** | power_core (inverter_battery_soc) | battery_state_health → dashboards |
| **energy_core** | power_core (inverter power + today totals) | solar_to_house/battery/grid power flow, solar_used_by_house_today → solar_state |
| **energy_helpers** | power_core (today totals), power_templates (load groups) | utility meters, energy_today series → dashboards |
| **energy_state** | power_core, battery_runtime (minutes_safe), solar_clipping (export_potential), power_templates (groups), solcast | energy_orchestrator_state, economy score, resilience hours → automations |
| **grid_risk** | battery_runtime (minutes_safe, severity, ss_soc_time_left), power_core (inverter sensors) | grid_risk_level, grid_risk_severity → grid_state |
| **grid_state** | power_helpers (group.inverter_grid), grid_risk (grid_risk_severity) | grid_state_health → dashboards |
| **solar_core** | power_core (inverter sensors) | solar_export_potential → energy_state, power_strategy |
| **solar_clipping** | power_core (solar, load, battery), solar_core (export_potential) | solar_unused_power, solar_opportunity_level → dashboards, automations |
| **solar_state** | power_core (inverter sensors), energy_core (solar_used_by_house_today) | solar_system_status, solar_generation_confidence, solar_stability → power_state |
| **solar_forecast** | solcast (external), load_shedding (external) | solar_forecast_available_conservative, inverter mode automation |
| **solar_helpers** | (none) | threshold helpers for solar_forecast automation |
| **prepaid_core** | energy_core (grid_energy_import_total), energy_helpers (solar_production_monthly), solcast (forecast_remaining) | prepaid balance sensors, drift model → prepaid_strategy |
| **prepaid_helpers** | (none) | input_number helpers for prepaid_core |
| **prepaid_strategy** | prepaid_core (balance, burn_rate), load_shedding (via state) | buy_score, buy_decision, depletion_date → dashboards, notifications |
| **load_shedding** | **External only** (load_shedding custom component) | load_shedding_active, countdown → solar_forecast, alerts |
| **power_contract** | (documentation only) | (none) |

### Dependency Graph

```
HARDWARE (solarman: inverter_1_*, inverter_2_*)
    │
    ▼
┌─────────────┐
│ power_core  │ ← linchpin — consumed by ALL subsystems
└──────┬──────┘
       │
   ┌───┴──────────────────────────────────┐
   │                                      │
┌──▼──────────┐                    ┌──────▼──────┐
│power_helpers│                    │battery_*    │
│(groups)     │                    │(runtime/    │
└──┬──────────┘                    │ state)      │
   │                               └──────┬──────┘
   │              ┌────────────────────────┘
   │              │
┌──▼──────────────▼──┐     ┌─────────────────┐
│ power_state        │     │ grid_risk        │
│ (health/flow)      │     │                 │
└──────┬─────────────┘     └────────┬────────┘
       │                            │
       │                    ┌───────▼────────┐
       │                    │ grid_state     │
       │                    └────────────────┘
       │
┌──────▼─────────┐   ┌──────────────────┐   ┌──────────────┐
│ power_strategy │   │ energy_*         │   │ solar_*      │
│ (decision)     │   │ (core/state/     │   │ (core/state/ │
└────────────────┘   │  helpers)        │   │  clipping/   │
                     └──────────────────┘   │  forecast)   │
                                            └──────────────┘
       │                    │                      │
       └────────────────────┴──────────────────────┘
                            │
              ┌─────────────▼──────────┐
              │  prepaid_*             │
              │  (core/strategy/       │
              │   helpers)             │
              └────────────────────────┘
                            │
              ┌─────────────▼──────────┐   ┌────────────────┐
              │  AUTOMATIONS / SCENES  │   │ load_shedding  │
              │  DASHBOARDS / ALERTS   │   │ (independent)  │
              └────────────────────────┘   └────────────────┘

External inputs:
  solcast → solar_forecast, solar_state, energy_state, prepaid_core
  load_shedding custom component → load_shedding.yaml → solar_forecast, alerts
  hardware registers (program SOC/time) → battery_runtime
```

### Modification Safety Matrix

Before modifying any of these files, audit the listed dependents:

| File to modify | Must audit before touching |
|---|---|
| `power_core.yaml` | ALL 21 other files — it is the foundation |
| `battery_runtime.yaml` | `power_state.yaml`, `grid_risk.yaml`, `energy_state.yaml` |
| `solar_state.yaml` | `power_state.yaml`, `energy_state.yaml`, `power_strategy.yaml` |
| `energy_core.yaml` | `solar_state.yaml` (consumes solar_used_by_house_today), `prepaid_core.yaml` |
| `solar_clipping.yaml` | `energy_state.yaml`, `power_strategy.yaml` |
| `power_helpers.yaml` | `power_state.yaml`, `energy_helpers.yaml`, `power_strategy.yaml`, `grid_state.yaml` |

### Safe Split / Restructure Candidates

| File | Status | Notes |
|---|---|---|
| `load_shedding.yaml` | ✅ Can be extracted | Zero internal dependencies. Candidate for own `eskom/` package |
| `power_contract.yaml` | ✅ Can be removed | Documentation only — no entities. Merge comments elsewhere |
| `grid_state.yaml` | ⚠️ Can be merged | Only depends on grid_risk + power_helpers; could merge with grid_risk.yaml |
| All others | ❌ Do not split | Tight coupling; splitting will break entity references |

---

*Last updated: 2026-04-13*  
*Updated by: Deep audit — all 22 power package files read, POWER_DEPENDENCY_ANALYSIS.md incorporated, watchman cross-referenced, legacy automations checked*
