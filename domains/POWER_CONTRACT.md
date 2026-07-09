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
| Battery bank | ~48,230 Wh total (48.2 kWh) — 3× Greenrich AF1600 (314Ah, 51.2V, 16S1P) |
| Grid | Eskom prepaid meter — **Empire CIU EV-KP**, Johannesburg City Power Area 3-11 Weltevredenpark |
| Solar forecast | Solcast integration |
| Load shedding | SePush/load_shedding integration |

---

## 2. File Inventory

**Package path:** `packages/power/` — 26 files

| File | Role | Layer |
|---|---|---|
| `power_core.yaml` | Dual-inverter aggregation — all unified sensors | Aggregation |
| `power_helpers.yaml` | Load groups (groups) + area/load power templates + solar season factor helpers (P6) | Mixed (violates layering) |
| `power_templates.yaml` | Per-inverter PV, battery, grid sub-sensors | Derived |
| `power_state.yaml` | house_power_health, power_state, inverter health, freq/temp | State |
| `power_statistics.yaml` | Rolling average sensors + solar forecast accuracy + 4-state weather correlation + season factor (P6 2026-06-14) | Derived |
| `power_strategy.yaml` | power_strategy, power_strategy_status, severity | Decision |
| `power_automations.yaml` | Power domain automations — migrated from automations.yaml | Automation |
| `geyser_automations.yaml` | Geyser heat pump scheduling — 4 automations + script.geyser_manual_run (E2 2026-06-14; E4 retrofit: orchestrator gate, midday solar-gated, at-temp proxy, window helpers, emergency off) | Automation |
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
| `solar_clipping.yaml` | Unused power, opportunity level, efficiency loss, clipping; solar_production_vs_capacity (P6) | Derived |
| `solar_core.yaml` | Solar export potential, surplus sensor + binary_sensor | Core |
| `solar_forecast.yaml` | Forecast sensors, inverter mode automation (3h) | Derived + Automation |
| `solar_helpers.yaml` | solar_surplus_threshold, solar_export_tariff | Helpers |
| `solar_state.yaml` | Solar status, ramp rate, confidence, stability, window; solar_vs_forecast_ratio_today (P6) | State |

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

**Edge case — meter reading taken AFTER the voucher was already applied** (e.g. missed/late entry, or a voucher that timed out on the meter and was re-keyed later, 2026-07-03 incident): step 2's "Update Prepaid Units" button ADDS the purchase kWh on top of the current `prepaid_units` balance — correct only if the balance you Realigned to in step 1 does NOT already include that purchase. If your fresh meter reading already reflects the purchase (post-application), do NOT click Update Prepaid Units as normal — it will double-count the kWh into the balance. Instead: Realign as normal (safe, only touches drift/offset), then manually set `input_number.initial_prepaid_grid_import` to the live `sensor.grid_energy_import_total` (mirrors what Update would have snapshotted), then add the purchase's units/cost/fixed-charge directly to `prepaid_total_units`/`prepaid_total_spent`/`prepaid_fixed_cost_paid` (and the month/year equivalents below) without touching `prepaid_units` again. Verify first via meter math: `balance_delta + consumption_since_last_reading` should equal the purchased kWh, confirming the voucher actually landed before backfilling.

**Prepaid diagnostic (added 2026-04-23):** `pyscript.prepaid_diagnostic` (mirrors `power_snapshot.py` pattern) — creates persistent_notification with full reconciliation state, drift trend, balance cross-check, and realign preview. Callable via `script.prepaid_diagnostic` dashboard button or Developer Tools → Services.

**`sensor.prepaid_net_position_this_month`**: Always available (no guard, all inputs use `| float(0)`). Returns 0 at month start before utility meters accumulate. Formula: `solar_savings_this_month - prepaid_spend_this_month`.

### Reliability Classification

| Tier | Sensors | Trust | Notes |
|---|---|---|---|
| AUTHORITATIVE | `prepaid_meter_lifetime_import`, `prepaid_units_left_safe` (when drift low) | High | Manual entry is ground truth |
| OPERATIONAL | `grid_energy_import_total`, `inverter_battery_soc`, `house_solar_power` | High | High frequency, reset risk |
| DERIVED/RELIABLE | `prepaid_units_used_authoritative`, `prepaid_drift_percentage`, `energy_independence` | High | Clean formulas, verified |
| ESTIMATED | `prepaid_estimated_days_remaining`, `prepaid_adaptive_burn_rate`, `battery_runtime` | Medium | Depends on rolling averages + thresholds |
| OPERATIONAL | `battery_night_survival` | Medium | Verified working 2026-06-19 — battery_energy_available + average_night_consumption both implemented |
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
sensor.inverter_battery_capacity Wh  (48230 * SOC/100, rough estimate)
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
sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c
```

**BUG FIXED 2026-06-28:** `load_shedding_templates.yaml` (all three derived sensors above)
was still reading the stale `sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark`
entity (state `None`, no attributes — a prior area-code/integration version), not the live
`sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c` this contract already documented
above (and which `load_shedding_automations.yaml` was already using correctly). `binary_sensor.
load_shedding_active` happened to still read "off" by coincidence (no shedding scheduled while
broken), which is why it went unnoticed. Found while wiring load-shedding awareness into
`inverter_p4_grid_charge_control` — see "P4 grid charge" below.

### Load Power (power_helpers.yaml)

```
sensor.house_kitchen_power         W  (group.house_kitchen_power_sensors — unknown state)
sensor.house_living_areas_power    W  (group.house_living_areas_power_sensors — unknown)
sensor.house_laundry_power         W  (group.house_laundry_power_sensors — unknown)
sensor.house_entertainment_power   W  (group.house_entertainment_power_sensors — unknown)
sensor.house_outdoor_power         W  (group.house_outdoor_power_sensors — unknown)
sensor.house_security_power        W  (group.house_security_power_sensors — defined in power_templates.yaml; member entities TBD)
sensor.house_known_power           W  sum of group.known_power_loads
sensor.house_flexible_power        W  sum of group.flexible_power_loads
sensor.house_critical_power        W  sum of group.critical_power_loads
sensor.house_unknown_power         W  house_load - known_power (unmonitored load)
sensor.load_visibility_score       %  known_power / house_load_power
sensor.flexible_load_percent       %  flexible_power / house_load_power
sensor.house_outdoor_power_throttled W  10s debounce version of outdoor
```

### Plug Device Change Log

| Date | Old Device / Entity | New Device / Entity | Reason |
|------|---------------------|---------------------|--------|
| 2026-05-25 | Water Cooler Plug (`switch.water_cooler_plug`) | LG Combo Washer Plug (`switch.lg_combo_washer_plug`) | Plug relocated to LG combo washer for energy tracking; higher consumption load. ZHA device renamed, all `water_cooler_plug_*` entities renamed to `lg_combo_washer_plug_*`. Config: added `sensor.lg_combo_washer_plug_power` to `known_power_loads`, `flexible_power_loads`, `house_laundry_power_sensors` in `power_templates.yaml`. UI: integral sensor source + utility meters + Energy Dashboard updated manually. |

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
sensor.prepaid_monthly_usage_true        kWh  "Prepaid Calendar Month Usage" — ✅ RENAMED 2026-07-08 (was mislabeled
                                                "Rolling 30-Day Grid Usage", flagged 2026-07-03; entity_id/unique_id left
                                                untouched to avoid orphaning dashboard bindings, only the friendly name
                                                and doc entry were wrong). Mirrors the prepaid_import_monthly utility
                                                meter's `last_period` attribute (last COMPLETED calendar month's total,
                                                frozen until next month boundary) or current running total if
                                                last_period is 0. This is genuinely calendar-month based, not rolling —
                                                the name now matches the behavior.
sensor.prepaid_rolling_30day_usage       kWh  NEW 2026-07-08 — the actual sliding 30-day window that was missing.
                                                `platform: statistics`, source sensor.grid_energy_import_total,
                                                `state_characteristic: change`, `max_age: days: 30` (change = newest
                                                minus oldest sample in the buffer — correct way to get a rolling delta
                                                off a total_increasing energy sensor). Kept as a separate entity from
                                                prepaid_monthly_usage_true above, deliberately — don't conflate rolling
                                                and calendar-month semantics in one sensor.

sensor.prepaid_energy_cost_per_kwh       R/kWh  last purchase cost
sensor.prepaid_true_cost_per_kwh         R/kWh  lifetime blended cost
sensor.prepaid_true_cost_per_kwh_monthly R/kWh  monthly blended cost
sensor.prepaid_spend_this_month          R      "(Estimate)" — prepaid_import_monthly (kWh) × prepaid_true_cost_per_kwh
                                                (LIFETIME blended rate). NOT a sum of real purchases this month — will
                                                diverge whenever this month's purchases are priced differently to the
                                                lifetime average. Kept for mid-month projection before topping up.
sensor.prepaid_actual_spend_this_month   R      (added 2026-07-03) REAL sum of purchases this calendar month =
                                                input_number.prepaid_month_spent + prepaid_month_fixed_paid. Use this
                                                over the estimate above for actual budget tracking. Long-term
                                                statistics backfilled 2026-07-09 for Dec 2024 - Jun 2026 (see below).
sensor.prepaid_month_energy_spend        R      (added 2026-07-09) = input_number.prepaid_month_spent alone — the
                                                energy-only half of the combined sensor above, split out so the
                                                dashboard can graph energy vs. fixed-charge trend separately, not
                                                just the combined bar. Same reset cadence
                                                (prepaid_month_counters_reset, 1st of month).
sensor.prepaid_month_fixed_charge        R      (added 2026-07-09) = input_number.prepaid_month_fixed_paid alone —
                                                the fixed/service-charge half. Same reset cadence as above.
sensor.prepaid_actual_spend_this_year    R      (added 2026-07-03) same as above, annual — prepaid_year_spent +
                                                prepaid_year_fixed_paid. Only accurate from 2026-07-03 onward (not
                                                backfilled — annual figure wasn't part of the 2026-07-09 backfill,
                                                only the 3 monthly sensors were).

# Historical spend backfill (2026-07-09, corrected 2026-07-09b) —
# sensor.prepaid_actual_spend_this_month, sensor.prepaid_month_energy_spend,
# sensor.prepaid_month_fixed_charge, sensor.prepaid_month_block1_kwh/_block2_kwh/
# _block3_kwh all have real long-term statistics for Dec 2024 - Jun 2026 (one
# point/month: state = that month's real total, sum = running cumulative),
# reconciled from City Power Discovery-app receipts + a personal Tymebank purchase
# spreadsheet (75 real data points — every purchase has an actual receipt, no
# estimates — two independent parallel purchase streams, no overlapping dates).
# July 2026 onward is live-tracked normally, not part of the backfill. IMPORTANT: recorder.import_statistics
# is NOT a callable HA service in this install (only recorder.purge/purge_entities/
# enable/disable/get_statistics are registered) — the backfill required a one-off
# pyscript script with allow_all_imports/hass_is_global TEMPORARILY set true
# (.storage/core.config_entries, pyscript entry), reverted immediately after (confirmed
# false/false post-revert). Editing .storage/* while HA is running does not persist —
# HA's in-memory copy overwrites the file on its next save (e.g. during a restart's
# shutdown phase); any future .storage edit of this kind must use ha core stop → edit →
# ha core start, not ha core restart.

# Month/year real-purchase counters (prepaid_helpers.yaml, added 2026-07-03)
# Reset by automation.prepaid_month_counters_reset (00:00:10, day==1; year counters
# only reset when day==1 AND month==1). Incremented per-purchase by
# script.update_prepaid_units alongside the never-reset lifetime prepaid_total_*.
input_number.prepaid_month_units/_spent/_fixed_paid/_discount   this calendar month
input_number.prepaid_year_units/_spent/_fixed_paid/_discount    this calendar year

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
sensor.prepaid_current_tariff_block      1/2/3  (added 2026-07-09) City Power inclining-block
                                                position for this month's real purchases so far
                                                (input_number.prepaid_month_units vs
                                                prepaid_tariff_block1_threshold/_block2_threshold).
                                                Attributes: kwh_this_month, kwh_to_next_block,
                                                current_rate. Feeds tariff context into the
                                                prepaid_buy_decision_notify message below.
```

# City Power Inclining Block Tariff (added 2026-07-09)
Residential Prepaid High, FY2025/26 (joburg.org.za Consolidated-Tariffs-FY20252026.
FINAL.pdf) — Block 1 0-350kWh/month @ R2.6645/kWh, Block 2 350-500kWh @
R3.0564/kWh, Block 3 >500kWh @ R3.4826/kWh, Fixed R200/month (R70 service + R130
capacity — matches this dataset's empirical R200 flat rate exactly). Thresholds
unchanged from FY2024/25 (only per-block rates increased annually) — stored as
editable input_numbers, not hardcoded, since they change every fiscal year:
```
input_number.prepaid_tariff_block1_threshold/_block2_threshold   kWh (350/500)
input_number.prepaid_tariff_block1_rate/_block2_rate/_block3_rate  R/kWh
```
script.update_prepaid_units (prepaid_core.yaml) computes the block breakdown for
every real purchase entered (using input_number.prepaid_month_units BEFORE this
purchase as the starting point) — stored in
input_number.prepaid_last_topup_block1_kwh/_block2_kwh/_block3_kwh, and
accumulated into input_number.prepaid_month_block1_kwh/_block2_kwh/_block3_kwh
(reset by prepaid_month_counters_reset, same cadence as prepaid_month_units). The
"Prepaid Top-Up Recorded" notification lists the block breakdown for that
purchase and escalates to severity: warning if any kWh landed above Block 1.
Graphable wrappers (same pattern as prepaid_month_energy_spend):
sensor.prepaid_month_block1_kwh/_block2_kwh/_block3_kwh — "Monthly kWh by Tariff
Block" plotly card, lovelace.dashboard_operations view 11 section 2. Historical
long-term statistics backfilled Dec 2024 - Jun 2026 (2026-07-09) alongside the
spend sensors — see below.

**Gotcha (found 2026-07-09): modern `template:` platform does NOT support
`object_id`** — using it silently drops the entire sensor definition (only
visible in core logs: `'object_id' is an invalid option for 'template'`). To
control entity_id, adjust the `name:` field instead — HA's slugify inserts an
extra underscore for a space-before-digit (e.g. "Block 1" → `block_1`, not
`block1`), so name sensors "Block1" (no space) if a specific entity_id suffix is
needed. If an entity was already created under the wrong slugified id, a
`template.reload` alone will NOT rename it — the entity registry's entity_id
assignment is sticky per unique_id; the stale row must be removed from
`.storage/core.entity_registry` (core stopped) before the corrected name/config
can register under the intended entity_id.

### Battery (battery_runtime.yaml, battery_state.yaml)

```
sensor.ss_battery_capacity              Wh   program-aware target SOC × 48230
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

# Energy orchestrator (energy_state.yaml) — rebuilt Session E1 2026-06-14
# 6-state priority ladder (first match wins, evaluated top-to-bottom):
#   loadshedding_critical  load shedding active AND SOC < emergency threshold (15%)
#   loadshedding           load shedding active OR (upcoming within pre_shed_hours_warning AND SOC < pre_shed_soc_threshold)
#   critical               battery_state_health == 'critical' OR SOC < critical_threshold (25%)
#   conserve               battery_state_health == 'low' OR SOC < conserve_threshold (50%)
#                          OR solar_weather_correlation == 'degraded' AND SOC < degraded_soc_threshold (60%)
#   surplus                solar_export_potential > surplus_threshold (1000W) AND battery strong/healthy
#   normal                 default
# Gated by: input_boolean.orchestrator_enabled — when OFF, always returns 'normal'
# Consumers: binary_sensor.water_refill_allowed (blocks on critical/loadshedding/loadshedding_critical)
#            lighting_energy_saving.yaml ✅ 2026-06-19 — auto_enable/disable automations wired
#            automation.energy_saving_mode_auto_enable: SOC < threshold OR orchestrator critical/loadshedding
#            automation.energy_saving_mode_auto_disable: clears when BOTH SOC > recovery AND orch normal/surplus
#            pool_pump_solar_control ✅ E3 2026-06-14 — orchestrator is now primary gate
sensor.energy_orchestrator_state        loadshedding_critical/loadshedding/critical/conserve/surplus/normal
sensor.orchestrator_decision_reason     human-readable string explaining why current state was chosen
#                                       e.g. "SOC 23% below critical threshold 25%" or "Load shedding active"
sensor.energy_economy_score             0-100
sensor.house_energy_resilience_hours    h  how long house can run on current energy
sensor.house_energy_resilience_status       text description
sensor.energy_loss_percent_today        %
sensor.energy_loss_root_cause               text diagnosis
sensor.power_loss_root_cause_live           live version
sensor.energy_loss_state                    ok/minor_loss/significant_loss/major_loss
sensor.battery_night_survival               % battery_energy_available / (average_night_consumption × 10h) × 100
                                              VERIFIED WORKING 2026-06-19 (was stale BROKEN label)
sensor.solar_sufficiency_tomorrow           based on solcast forecast
binary_sensor.grid_charging_while_solar     not a binary_sensor — see sensor below
sensor.grid_charging_while_solar            True when pv_power > 1000W AND grid_to_battery_power > 200W
                                              VERIFIED WORKING 2026-06-19 (was stale BROKEN label)
sensor.energy_model_integrity               ok/warning/error (consistency check)

# Grid risk (grid_risk.yaml)
sensor.grid_risk_level        unknown/critical/warning/moderate/safe
                              Classification logic (updated 2026-06-17):
                              Grid ON  branch: <30min=warning, <120min=moderate, else=safe
                                        Critical is suppressed when grid is available —
                                        runtime to SOC floor is not a genuine emergency.
                              Grid OFF branch: battery_runtime_severity==critical→critical,
                                        <180min→warning, <360min→moderate, else safe.
sensor.grid_risk_status       emoji + text status (🟢/🟡/🟠/🔴)
sensor.grid_dependency_eta    clock ETA (HH:MM) when battery reaches target SOC floor
sensor.grid_risk_severity     normal/critical/warning/information

# Grid state (grid_state.yaml)
sensor.grid_state_health      stable/risk/unstable/offline

# Battery state (battery_state.yaml)
sensor.battery_state_health   strong/healthy/low/critical

# Rolling averages & weather correlation (power_statistics.yaml — added 2026-04-22; upgraded P6 2026-06-14; fixed 2026-06-18)
# NOTE (2026-06-18): 7d/30d_mean rewritten as template sensors. Old implementation used platform:statistics
# on inverter_today_production with sampling_size:7 = mean of last 3.5 minutes of readings (wrong).
# New implementation: change statistic on inverter_total_production (lifetime, no resets) ÷ days.
# Both _mean sensors now INCLUDED in recorder. Intermediate _total sensors EXCLUDED.
sensor.inverter_production_7d_total   kWh  intermediate: change/7d on inverter_total_production (EXCLUDED recorder)
sensor.inverter_production_30d_total  kWh  intermediate: change/30d on inverter_total_production (EXCLUDED recorder)
sensor.inverter_production_7d_mean    kWh  7-day rolling mean of daily PV production (template: 7d_total/7)
sensor.inverter_production_30d_mean   kWh  30-day rolling mean of daily PV production (template: 30d_total/30)
sensor.inverter_production_7d_stdev   kWh  7-day std dev of daily production (CV signal for solar_weather_correlation)
sensor.house_load_24h_mean            W    24h rolling mean load power (used by P4 grid charge heuristic)
sensor.house_load_7d_mean             kWh  7-day mean of daily load consumption (seasonal comparison)
sensor.solar_vs_forecast_ratio_today  %    Today actual / Solcast forecast today. 10am guard: returns 100 before 10am.
                                           Distinct from solar_vs_forecast_ratio_7d (7d mean/forecast) and
                                           solar_vs_forecast_ratio in energy_helpers.yaml (no 10am guard).
sensor.solar_forecast_accuracy_7d     %    7-day statistics mean of solar_vs_forecast_ratio_today.
                                           Primary ratio_7d input to solar_weather_correlation.
sensor.solar_vs_forecast_ratio_7d     %    Legacy: 7d_mean / today's Solcast forecast. Retained for
                                           backward compat — dashboards may reference it.
sensor.solar_weather_correlation           4-state: excellent / good / poor / degraded
                                           Classification (worst-match-wins):
                                             degraded:   ratio_7d < 65 OR (ratio_today < 50 AND hour ≥ 14) OR cv > 40
                                             poor:       ratio_7d < 80 OR (ratio_today < 65 AND hour ≥ 12) OR cv > 25
                                             excellent:  ratio_7d ≥ 95 AND ratio_today ≥ 90 AND cv < 15
                                             good:       default
                                           Consumers: energy_state.yaml checks (== 'degraded') → conserve branch.
                                           'ratio' attribute kept for energy_state.yaml decision_reason string.
                                           Backward compat: 'degraded' state preserved from 2-state (2026-04-22) version.
sensor.solar_season_efficiency_factor      Float multiplier (dimensionless). Season from sensor.season:
                                             summer 1.0, autumn 0.75, winter 0.55, spring 0.85 (JHB defaults)
                                           UI-adjustable via input_number.solar_factor_{summer|autumn|winter|spring}.
                                           Review after 3+ months: calibrate to actual winter/summer production ratio.
                                           Used by P4 logbook context (expected_daily = 30d_mean × season_factor).

# Solar real-time efficiency (solar_clipping.yaml — added P6 2026-06-14)
sensor.solar_production_vs_capacity   %    PV output / system rated capacity. Sustained >95% peak = clipping risk.
                                           Capacity from input_number.system_solar_capacity_w (default 10800W).

# Solar forecast daily ratio (solar_state.yaml — added P6 2026-06-14)
sensor.solar_vs_forecast_ratio_today  %    (See above — also defined in this section for cross-reference)

# All statistics/correlation sensors excluded from recorder (diagnostic — recalculated from history).
# Source entities: inverter_today_production, inverter_load_power, inverter_today_load_consumption,
#                  inverter_pv_power, solcast_pv_forecast_forecast_today, sensor.season

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
input_number.grid_import_usage_low_warning_trigger    ← UI helper (Settings → Helpers)
input_number.grid_import_usage_medium_warning_trigger ← UI helper
input_number.grid_import_usage_high_warning_trigger   ← UI helper
input_number.inverter_battery_soc_warning_trigger
```

**Grid Import Daily Warning** (migrated 2026-06-18 from automations.yaml id 1742385109757):
```
automation.grid_import_daily_warning    power_automations.yaml
  Triggers: numeric_state above each threshold (low / medium / high)
  Fires: once per threshold crossing per day (sensor resets at midnight)
  Severity: information (low) / warning (medium) / warning (high)
  Route: script.notify_power_event — quiet hours respected
  Message: grid_kwh, SOC, solar_today, orchestrator_state, solar_weather_correlation
  Fix vs original: numeric_state triggers replace state-change+choose (original
    re-fired on every state update while within a band). Dead entity refs removed
    (inverter_solar_appliance_mode_helper, accuweather_hours_of_sun_day_0).
```

### Solar Controls (solar_helpers.yaml)

```
input_number.solar_surplus_threshold    W  threshold for surplus opportunity (default: 800)
input_number.solar_export_tariff        R/kWh  notional export value (default: 1.30)
```

### Solar Statistics + Season Factor Helpers (power_helpers.yaml — added P6 2026-06-14)

```
input_number.system_solar_capacity_w      W  10800  rated PV capacity — used by solar_production_vs_capacity
input_number.solar_factor_summer          ×   1.00  JHB summer production factor (Dec–Feb)
input_number.solar_factor_autumn          ×   0.75  JHB autumn production factor (Mar–May)
input_number.solar_factor_winter          ×   0.55  JHB winter production factor (Jun–Aug)
input_number.solar_factor_spring          ×   0.85  JHB spring production factor (Sep–Nov)
# Calibrate season factors after 3+ months of data by comparing actual winter/summer production.
# All factors are UI-adjustable from dashboard without editing YAML.
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

**Grid-offline and last-sun-slot helpers (added 2026-05-25):**
```
input_number.grid_offline_soc_min_pool       %   min SOC to run pool during grid outage (default: 60)
input_number.grid_offline_soc_min_borehole   %   min SOC for borehole during grid outage (default: 50)
input_number.grid_offline_soc_min_geyser     %   min SOC for geyser non-critical window (default: 45)
input_number.geyser_grid_offline_critical_soc % min SOC for geyser even in critical window (default: 35)
input_number.last_sun_soc_target_summer      %   overnight charge target summer (default: 80)
input_number.last_sun_soc_target_winter      %   overnight charge target winter (default: 90)
input_number.last_sun_slot_start_hour        h   hour when last-sun-slot starts (default: 14.0)
input_number.geyser_target_hours_summer      h   geyser daily solar target summer (default: 2.0)
input_number.geyser_target_hours_winter      h   geyser daily solar target winter (default: 3.5)
sensor.last_sun_soc_target                   %   season-aware (80% summer / 90% other)
sensor.geyser_target_run_hours_today         h   season-aware geyser target
```

**Control logic summary (updated E3 2026-06-14 — orchestrator as primary gate):**

`sensor.energy_orchestrator_state` is the primary gate for all pool pump decisions from E3 onwards. This is the reference pattern all appliance automations will follow — geyser and borehole control will adopt the same structure.

- Active window: 08:00–16:00. Gate: `input_boolean.load_control_pool_enabled = on`.
- **Turn on (Branch 5)**: `energy_orchestrator_state in ['surplus', 'normal']` + headroom > 1200W + daily target not reached + local gates below. Replaces raw battery_state_health / load_shedding_stage / upcoming-shed SOC checks.
- **Turn on local gates (preserved — not covered by orchestrator)**:
  - Grid-offline SOC floor: `grid_offline_soc_min_pool` (60%) — blocks when grid down and SOC too low
  - Last-sun-slot: blocks from `last_sun_slot_start_hour` (14:00) when SOC < `last_sun_soc_target` (80%/90% by season)
  - Winter morning hold: `pool_winter_start_hour` (10:00) — no pool before 10:00 in winter
- **Turn off (16:00 hard stop — Branch 0)**: unconditional — no minimum run time guard.
- **Turn off (Branch 2a — grid offline + low SOC)**: immediately shuts off if grid is offline AND SOC < `grid_offline_soc_min_pool` (60%). Bypasses minimum run time.
- **Turn off (Branch 2b — last sun slot)**: shuts off from `last_sun_slot_start_hour` when SOC < `last_sun_soc_target`. Bypasses minimum run time.
- **Turn off (Branch 2 — orchestrator hard block)**: `energy_orchestrator_state in [loadshedding, loadshedding_critical, critical]`. Enforces minimum run time EXCEPT `loadshedding_critical` (emergency — turns off immediately). Replaces raw load_shedding_stage + upcoming-shed checks. Notifies at severity: warning.
- **Turn off (Branch 3 — conserve / solar dropped)**: `energy_orchestrator_state == conserve` OR `solar_available_surplus < 800W` (hysteresis floor). Enforces minimum run time. Replaces raw battery_state_health low/critical + headroom checks.
- **Turn off (target met — Branch 4)**: daily run hours >= season target — enforces minimum run time.
- Minimum run time guards prevent short cycles (pump must run `pool_minimum_run_minutes` before most turn-offs). Does NOT apply to 16:00 hard stop, grid-offline, last-sun, or loadshedding_critical branches.
- Season logic: `sensor.season == 'summer'` → summer target; all other seasons → winter target.
- `sensor.pool_pump_control_status` updated E3: reads orchestrator state as primary reason string (priority 1→11 ordered display).
- `sensor.solar_available_surplus` (power_state.yaml) is the canonical solar headroom sensor — Watts (pv_power − load_power). Note: `sensor.solar_surplus_available` (solar_core.yaml) is a boolean string True/False — NOT a Watts value. Use `solar_available_surplus` in all appliance automations.
- All notifications via `script.notify_power_event` (severity: information/warning, subsystem: energy).
- `mode: queued` — ensures Branch 1 (records `pool_pump_last_on`) is not dropped when Branch 5 turns on the pump.

**Design note — incident 2026-05-25 (Eskom outage):**
Grid failed ~01:00. Battery drained from 45% to 38% by dawn. Pool pump turned on at 07:25 drawing ~3.9kW from battery (grid still offline). BMS protection tripped at 07:45 (SOC crashed from ~36% to 12%), tripping the house. House off 07:50–09:45. Grid-offline gate (60% threshold) would have prevented the pool pump from starting at 38% SOC. Battery also showing possible health issue — 24% SOC drop in 5 minutes under load is not consistent with rated capacity.

**Replaced automations (removed 2026-04-22):**
```
1742227789739 — Pool Pump Turn Off Daily PM
1742294033785 — Pool Pump Turn on Early - Daily AM 9-1pm
1742294477609 — Pool Pump Turn on - Daily AM 11-1pm
1742295582190 — Pool Pump Turn on - Daily PM 1-4pm
```

### Energy Orchestrator Controls (power_helpers.yaml — added E1 2026-06-14)

Master switch:
```
input_boolean.orchestrator_enabled      default: true — when OFF, orchestrator returns 'normal' always
```

SOC threshold ladder (all UI-controllable, no hardcoded values in templates):
```
input_number.orchestrator_emergency_soc_threshold     %  15  — halt almost everything
input_number.orchestrator_critical_soc_threshold      %  25  — battery critical floor
input_number.orchestrator_conserve_soc_threshold      %  50  — shed non-essential loads
input_number.orchestrator_pre_shed_soc_threshold      %  80  — pre-load-shedding reserve gate
input_number.orchestrator_conserve_degraded_soc_threshold % 60 — SOC floor for degraded-solar conserve branch
```

Other thresholds:
```
input_number.orchestrator_surplus_export_threshold    W  1000 — solar surplus to enter surplus state
input_number.orchestrator_target_soc_by_sunset        %  90   — charge target by end of solar window (future use)
input_number.orchestrator_load_first_soc_threshold    %  65   — crossover: battery-first → load-first (future use)
input_number.orchestrator_pre_shed_hours_warning      h  3    — warning window before upcoming shed
```

Pool winter scheduling:
```
input_number.pool_winter_start_hour                   h  10   — earliest hour to run pool pump in winter
```

### Force Charge (power_helpers.yaml + power_automations.yaml — added 2026-06-18)

Temporarily enables Grid charging on all P1–P6 programs to force battery to a target SOC without
manually changing inverter programme settings. Saves and fully restores previous settings on completion.

```
input_boolean.force_charge_active        ON while a force charge run is in progress; gates restore automation
input_number.force_charge_target_soc     target SOC % (50–100, step 5, default 100)
input_text.force_charge_saved_charging   JSON: saved P1–P6 charging options + SOC numbers from INV1

script.force_charge_batteries            Entry point. Fields: target_soc (optional, default reads input_number).
                                           1. Saves P1–P6 charging+SOC as JSON to force_charge_saved_charging
                                           2. Pauses inverter_programme_auto_enabled
                                           3. Sets Battery First on INV1+INV2
                                           4. Sets Grid on P1–P6 INV1; SOC targets = target_soc
                                           5. Calls force_inverter_sync (INV1→INV2)
                                           6. Notifies warning

script.force_charge_restore              Internal restore (called by monitor + cancel).
                                           Restores P1–P6 options + SOC numbers on INV1 with 2s inter-write
                                           delays (Sunsynk drops rapid register writes). Syncs INV2 before
                                           re-enabling programme_auto (prevents race with pattern automation).

script.force_charge_cancel               Dashboard cancel → calls force_charge_restore immediately.

automation.force_charge_monitor          Template trigger: SOC ≥ force_charge_target_soc AND force_charge_active=on
                                           → calls force_charge_restore automatically.
```

Design notes:
- `inverter_programme_auto_enabled` is paused during force charge so the pattern automation doesn't fight the override.
- Restore order: restore INV1 → sync INV2 → wait 10s → re-enable programme auto (prevents pattern automation racing the sync).
- All restore actions have `continue_on_error: true` — partial restore preferred over full stop.
- JSON in force_charge_saved_charging includes both charging select states AND SOC number values.

### Inverter Sync (power_helpers.yaml + power_automations.yaml — E1 2026-06-14; expanded E5 2026-06-14)

All programme entities confirmed in scenes.yaml E5 pre-flight. Now fully implemented.

```
input_boolean.inverter_sync_status     ON = in sync; OFF = mismatch; written by inverter_sync_check

automation.inverter_sync_check         triggers on change to any of:
                                         select.inverter_1_energy_pattern
                                         select.inverter_1_program_1_charging through _6_charging
                                       90s delay → compares energy_pattern + all 6 charging selects
                                       sets sync status; notifies on mismatch with detail string

script.force_inverter_sync             copies INV1 → INV2:
                                         energy_pattern
                                         program_1_charging through _6_charging (select)
                                         program_1_soc through _6_soc (number)
                                       dashboard-callable for manual re-sync; called by E5-2 after pattern changes
```

### Inverter Programme Automation (power_helpers.yaml + power_automations.yaml — E5 2026-06-14)

**Controls inverter programme settings dynamically. Writes to physical inverter registers.**
Master gate: `input_boolean.inverter_programme_auto_enabled` — when OFF, makes no inverter changes.

**Helpers:**
```
input_boolean.inverter_programme_auto_enabled   default: true — master gate
input_boolean.use_legacy_solar_scenes           default: false — when ON, solar_forecast.yaml scene
                                                system resumes (rollback path for E5 automations)
input_number.orchestrator_p4_charge_trigger_soc %  70  — future use (pre-flight SOC gate for P4)
input_number.orchestrator_solar_gap_threshold  kWh  5.0 — P4 only enables if solar falls this far
                                                         short of target (updated from 2.0 on 2026-06-17 for 48 kWh bank)
input_number.battery_capacity_kwh              kWh 48.0 — 3× Greenrich AF1600 (48.23 kWh nominal — updated 2026-06-17)
input_number.p4_grid_charge_solar_gate_w       W   1500 — added 2026-06-23. Grid only enables if current
                                                         solar has dropped below this, even when the
                                                         forecast-shortfall math says it's needed.
input_number.p4_deadline_zone_start_hour       h   16.0 — added 2026-06-24. From this hour, abandons the
                                                         solar-aware/shortfall projection and forces Grid
                                                         on if soc_gap still exceeds the deadline threshold.
input_number.p4_deadline_gap_threshold         %   3   — added 2026-06-24. Gap to target below which the
                                                         deadline push doesn't bother forcing grid.
input_number.p4_max_grid_charge_rate_kw       kW   5.0 — added 2026-06-28. Observed achievable net
                                                         SOC-closing rate with Grid forced on continuously
                                                         (battery charge net of concurrent house load).
                                                         Used to detect when the remaining kWh shortfall
                                                         can no longer close in the time left — see
                                                         "Rate-urgency + load shedding" below.
```

**Programme time slots (confirmed E5 pre-flight):**
```
P1: 01:00–05:00   P2: 05:00–09:00   P3: 09:00–14:00
P4: 14:00–17:00   P5: 17:00–21:00   P6: 21:00–01:00
```

**E5-2: automation.inverter_energy_pattern_control**
```
Triggers: time_pattern /30min + orchestrator state change + homeassistant start + time 16:30
  (was /5min — reduced E8 fix 2026-06-14: 5min polling caused notification floods when
   inverter reverted to own schedule between polls; 30min is sufficient re-enforcement cadence)
Gate: inverter_programme_auto_enabled ON AND orchestrator_enabled ON

Notification behaviour (E8 fix 2026-06-14):
  Notifications suppressed on periodic trigger (trigger.id == 'periodic').
  Battery First and Load First push notifications only fire on REAL TRANSITIONS
  (orchestrator_changed, homeassistant start, evening_return triggers).
  Logbook entries still written on periodic re-enforcement for auditability.

Branch 1 (highest priority):
  orchestrator in [loadshedding, loadshedding_critical, critical, conserve]
  AND energy_pattern != 'Battery First'
  → set Battery First; notify if trigger != periodic (warning for emergency states, info for conserve)
  → call force_inverter_sync after 5s

Branch 2 (morning below threshold, silent):
  trigger != evening_return AND hour < 10 AND SOC < load_first_soc_threshold (65%)
  AND energy_pattern != 'Battery First'
  → set Battery First; logbook only

Branch 3 (solar hours, threshold met):
  trigger != evening_return AND orchestrator in [normal, surplus]
  AND SOC >= load_first_soc_threshold AND 09:00–16:00
  AND energy_pattern != 'Load First'
  → set Load First; notify info
  → call force_inverter_sync after 5s

Branch 4 (evening return, 16:30 time trigger):
  trigger == evening_return AND energy_pattern != 'Battery First'
  → set Battery First; notify info
  → call force_inverter_sync after 5s

Default: logbook only (no change)
mode: single (max_exceeded: silent — 5min ticks don't queue)
```

**E5-3: automation.inverter_p4_grid_charge_control** (rate-urgency + load-shedding awareness added 2026-06-28)
```
Triggers: time 14:00/14:30/15:00/15:30/16:00/16:30 (evaluate) + time 17:00 (restore, unconditional)
          + numeric_state sensor.inverter_battery_soc above orchestrator_target_soc_by_sunset (target_reached, real-time)
Gate: inverter_programme_auto_enabled ON

BUG FIXED 2026-06-28: the 17:00 `restore` trigger above did not actually exist
in code prior to this date — Branch 2 referenced `trigger.id == 'restore'` but
no trigger ever emitted that id, so Branch 2 was dead code (added trigger now
present). Had no charging-behaviour impact in practice — the inverter's own
P4 program window (14:00-17:00 hardware schedule) already cuts grid import at
17:00 regardless of the HA select state (confirmed via recorder:
`inverter_grid_power` drops to near-zero at 17:05 even when the select stayed
"Grid") — but the select entity was misleadingly stuck on "Grid" all
evening/overnight until the next day's 14:00 evaluate.

Branch 0 — Target reached (real-time, added 2026-06-23):
  Fires the INSTANT SOC crosses target_soc_by_sunset while P4 = Grid.
  → Disable P4 Grid immediately on both inverters. Does not wait for the
    next 30-min evaluate checkpoint or the 17:00 restore.
  Reason: previously grid charging could continue well past target SOC until
  the next fixed checkpoint — user flagged heavy grid consumption from this gap.

Branch 1 — Evaluate (14:00–16:30, every 30 min):
  Calculates:
    soc_gap = target_soc_by_sunset - current_soc  (floor 0)
    kwh_needed = (soc_gap / 100) × battery_capacity_kwh
    solar_for_battery = solcast_remaining - (house_load_24h_mean_kW × hours_to_17)
    kwh_shortfall = kwh_needed - solar_for_battery
    solar_tapered = sensor.inverter_pv_power < input_number.p4_grid_charge_solar_gate_w (1500W default)

  Enable P4 Grid (both INV1 + INV2 directly) if:
    soc_gap > 5%  AND  kwh_shortfall > orchestrator_solar_gap_threshold
    AND solar_tapered  AND  P4 not already Grid
    — added 2026-06-23: solar_tapered gate. Previously enabled purely on the
    forecast-shortfall projection, which could fire at 14:00 while solar was
    still producing strongly (observed: grid importing 6.8kW while solar
    produced 4kW simultaneously). Now grid charging is delayed until current
    production has actually dropped below the gate, even if the forecast
    says the day overall won't be enough — trades a small risk of missing
    the sunset target for materially less grid usage.

  Hold off (added 2026-06-23) if:
    soc_gap > 5% AND kwh_shortfall > threshold AND NOT solar_tapered AND P4 != Grid
    → logbook only, re-evaluated at the next 30-min checkpoint.

  Disable P4 Grid if:
    P4 is currently Grid AND (soc_gap ≤ 5% OR shortfall ≤ threshold OR NOT solar_tapered)
    — `OR NOT solar_tapered` added 2026-06-23: NOT sticky. Even if Grid was
    enabled at one 30-min checkpoint, if solar recovers above the gate by the
    next checkpoint (typically true 14:00-15:00 when sun is normally still
    strong), it releases back to solar immediately rather than staying on
    Grid until the shortfall math alone says so. Re-toggles every 30 min
    based on current conditions, not a one-shot decision for the rest of
    the window.

  Sets both inverters directly (not via force_inverter_sync — P4 only change).

Branch 2 — Restore 17:00 (always):
  Set P4 = Disabled on both INV1 and INV2.
  P5 window begins; grid charging in P5 may be active per existing programme.

Note: sensor.house_load_24h_mean (W, 24h rolling mean) used for load estimate —
more stable than instantaneous load_power for multi-hour heuristic. This sensor
was silently broken (returned `unavailable`) from a YAML structural bug in
power_statistics.yaml until fixed 2026-06-23 — see Known Issues. While broken,
the load term defaulted to 0 via `|float(0)`, making solar_for_battery slightly
optimistic (shortfall computed smaller than true value).

Forecast ratio adjustment (added 2026-07-01): incident — 2026-07-01 was a bad
solar day (Solcast forecast 46 kWh, actual much lower), but SOC only reached 81%
vs 90% target. Root cause: `sensor.solcast_pv_forecast_forecast_remaining_today`
reflects forecasted production for the remaining hours — on an overcast day where
actual output is 30% of forecast, the remaining_forecast stays optimistic all
afternoon. This kept `solar_for_battery` high, `kwh_shortfall` below the 5 kWh
`gap_threshold`, and blocked both the enable condition AND `rate_urgent` (both gate
on `kwh_shortfall > gap_threshold`). Only the 16:00 deadline_zone fired, leaving
just 1 hour to close the gap. Fix: multiply remaining_forecast by
`sensor.solar_vs_forecast_ratio_today / 100` before the shortfall calculation.
By 14:00 the ratio reflects 7+ hours including peak solar window (11:00-14:00)
— a highly reliable signal. On bad days (ratio 30%) adjusted remaining goes from
13.5 → 4 kWh, making kwh_shortfall accurate and triggering grid at 14:00-15:00
instead of 16:00. Good days (ratio ~100%) are unaffected. The ratio sensor has a
10am guard (returns 100 before 10am), so no risk of premature scaling.
Variables added: `remaining_forecast_raw` (raw Solcast value), `ratio_today`
(ratio / 100), `remaining_forecast` (adjusted = raw × ratio). Logbook messages
updated to show "raw X kWh × ratio Y% = Z kWh" for auditability.

input_number.p4_grid_charge_solar_gate_w  W  1500 default (power_helpers.yaml)
  Lower = waits longer/closer to sunset before resorting to grid (more
  efficient, more risk of not closing the gap in time).

Deadline push (added 2026-06-24): incident — SOC plateaued ~78-80% on both
2026-06-23 and 2026-06-24, never reaching the 90% target. Root cause: P4
enabled Grid at 16:00 (SOC 72%), which itself pushed SOC to 78% by the 16:30
checkpoint — that jump made the shortfall recalculation (using the same live
SOC) dip just under threshold, so "Disable — solar on track" fired and
cancelled grid for good, with no evaluate trigger between 16:30 and the
unconditional 17:00 restore to catch the mistake. From
`input_number.p4_deadline_zone_start_hour` (16:00 default) onwards, the
automation stops trusting the solar-aware projection: if `soc_gap >
input_number.p4_deadline_gap_threshold` (3% default), it forces/keeps Grid on
regardless of `solar_tapered` or the shortfall calc. The "Disable — solar on
track" branch is guarded with `and not (deadline_zone and soc_gap >
deadline_gap_threshold)` so it can't undo the deadline push. Branch 0
(real-time target-reached disable, added 2026-06-23) still takes priority and
clears it immediately once SOC actually reaches target — the deadline push
only matters for genuinely closing a real gap in the time remaining.

Rate-urgency + load shedding (added 2026-06-28): SOC history for 06-24
through 06-27 (recorder) showed the window still consistently plateauing
76-85%, never reaching the 90% target, even after the deadline-push fix
above. Root cause: `solar_tapered` reacts to instantaneous solar watts, not
the accumulating kWh shortfall — on 06-26, solar sat above the 1500W gate
from 14:00 to past 15:30 while `kwh_shortfall` grew from 4.1→8.6 kWh (logbook:
"P4 hold — shortfall X kWh > threshold but solar still YW"), leaving only the
fixed 16:00 deadline_zone to close a hole that had already outgrown what the
inverter can physically charge in the time left.

Added `rate_urgent`: computed each evaluate as
`required_rate_kw = kwh_shortfall / hours_to_deadline` vs
`input_number.p4_max_grid_charge_rate_kw` (5.0 kW default — the observed net
SOC-closing rate from forcing Grid on continuously on 2026-06-25: SOC
66%→83%, +8.2 kWh of the 48.2 kWh bank, over 1.5h ≈ 5.5 kW net, well below the
raw 6-7kW grid import on `inverter_grid_power` since load draws from the same
import). If closing the shortfall by the effective deadline would need a
sustained rate above this, the "Deadline/rate-urgency push" branch fires
immediately regardless of `solar_tapered` or the fixed 16:00 hour — same
forcing action as the deadline push, just triggered earlier when the math
says waiting will make the target unreachable.

The effective deadline is no longer always 17:00: if
`binary_sensor.load_shedding_active` is on, the deadline becomes "now" (grid
already unavailable — no benefit to waiting). If the area sensor's `forecast`
attribute shows a shed starting before 17:00, the deadline pulls forward to
that start time (grid won't be chargeable during the outage either). Both
collapse `hours_to_deadline` toward 0, which drives `required_rate_kw` up and
trips `rate_urgent` — this is how "work around load shedding in the P4
window" is implemented: charge before the outage rather than waiting for a
deadline that assumed grid would still be there. The "Disable — solar on
track" branch is guarded against undoing this the same way it's guarded
against the deadline push (`not (rate_urgent and soc_gap > deadline_gap_threshold)`).

Functional check (E5 2026-06-14):
  energy_pattern changed Load First → Battery First immediately on reload (orchestrator=conserve).
  P4=Disabled (17:00 window already closed, correct).
```

**Solar scene legacy gate:**
`input_boolean.use_legacy_solar_scenes = off` (default) → solar_forecast.yaml automation is disabled.
Flip to `on` to restore the scene-based system (Low/Medium/High Solar Forecast scenes). E5 automations
continue to run but solar_forecast.yaml resumes overriding them every 3 hours.

**Confirmed select entity option strings (from scenes.yaml):**
```
select.inverter_1_energy_pattern:       "Battery First" | "Load First"
select.inverter_1_program_N_charging:   "Disabled" | "Grid" | "Generator" | "Both"
select.inverter_1_work_mode:            "Export First" | "Zero Export To Load" | "Zero Export To CT"
Service call: select.select_option with option: "<exact string>"
```

### Air Fryer Control + Unknown Draw Detection (power_helpers.yaml + power_automations.yaml — E7 2026-06-14)

**Appliance entity correction**: spec called these `switch.air_fryer_switch` / `sensor.air_fryer_power` — actual entities are:
```
switch.philips_airfryer_plug     ← controllable smart plug switch (~1.5–2 kW)
sensor.philips_airfryer_plug_power ← live power reading
input_boolean.load_control_airfryer_enabled ← default: true — auto-cut gate
```

**Load visibility sensors (power_templates.yaml — already existed, E7-2 was no-op):**
```
sensor.known_load_power          kW  sum of group.known_power_loads (all monitored appliances)
sensor.unknown_load_power        kW  inverter_load_power/1000 - known_load_power (floor 0)
sensor.load_visibility_score     %   100 - (unknown/total × 100); 100 when idle
sensor.active_high_loads_power   W   known appliances drawing > 500W
```

**Unknown draw detection helpers (power_helpers.yaml — added E7):**
```
input_boolean.tier4_warnings_enabled         default: true — gate for unknown_draw_warning automation
input_number.unknown_draw_warning_threshold  W  1500 — warning level (W; sensor.unknown_load_power is kW)
input_number.unknown_draw_critical_threshold W  3000 — critical level
input_number.unknown_draw_duration_trigger   min  3 — minutes sustained above threshold before alert
input_datetime.unknown_draw_last_alert           cooldown stamp (30 min between alerts)
```

**Automations (power_automations.yaml — added E7):**
```
automation.airfryer_critical_cut
  Trigger: group.inverter_grid → off  OR  sensor.inverter_battery_soc below orchestrator_critical_soc_threshold
  Condition: load_control_airfryer_enabled ON + switch ON + grid OFF
             + SOC < critical_threshold + SOC > 5%
  Action: turn off switch + notify warning "Grid offline + low battery — air fryer cut"
  Design rule: grid ON → air fryer always allowed regardless of orchestrator state.
               grid OFF + SOC genuinely low (6–20%) → cut immediately.
               SOC ≤ 5% is treated as a startup/BMS-init glitch and is IGNORED — BMS
               physically shuts down at 10% so any reading ≤5% is sensor noise during
               inverter restart or battery swap. Guard added 2026-06-17 after confirmed
               false cuts during battery swap (SOC showed 0% with inverters in bypass).
  No auto-restart: cooking appliance safety rule.

automation.airfryer_restore_on_recovery
  Trigger: group.inverter_grid → on FROM off
  Condition: load_control_airfryer_enabled ON
  Action: notify info "Grid restored — air fryer safe to restart manually"

automation.unknown_draw_warning
  Trigger: sensor.unknown_load_power above 0, sustained for duration_trigger minutes
  Condition: tier4_warnings_enabled ON + orchestrator in [conserve, critical, loadshedding*]
             + unknown_draw_W > warning_threshold + 30-min cooldown gate
  Branch 1 (critical): unknown_draw > critical_threshold AND orch critical/loadshedding_critical
             → script.notify_power_event severity: critical
  Branch 2 (warning): unknown_draw > warning_threshold → severity: warning
  NOTE: does NOT fire on surplus/normal days — unknown draw acceptable then.
  Unit conversion: unknown_load_power (kW) × 1000 compared to threshold (W).
```

**Honor tablet notifications**: no separate chime/TTS mechanism exists. Tablets (Honor 10 Dash, Honor X7 Dash) receive push notifications via `mobile_app_honor10_dash` / `mobile_app_honorx7_dash` which are members of all STD_* groups. `script.notify_power_event` at severity critical already reaches both tablets. Individual services available: `notify.honor_10_dash_mobile_app`, `notify.honor_x7_dash_mobile_app`.

### Geyser Scheduling (power_helpers.yaml + geyser_automations.yaml — E2 2026-06-14; upgraded E4, timing/minimum-check 2026-06-17)

Migrated E2 from automations.yaml. E4 retrofit: orchestrator gate, midday solar gate, at-temperature proxy, window time helpers, morning hard-off, emergency off. 2026-06-17: all hard-off times shifted earlier; adaptive evening start added; daily minimum check added; battery SOC thresholds recalibrated for 48 kWh bank.

Switch entity: `switch.geyser_heat_pump_switch` (~1.25 kW draw).

**Helpers (power_helpers.yaml):**
```
# Control booleans
input_boolean.geyser_sports_night              ← Tue/Thu auto-set 17:00; clears 00:01 daily
                                                  sports_night only extends evening hard-off
input_boolean.geyser_morning_override          ← bypasses load_control_geyser_enabled for morning
                                                  Midday (12:00) auto-clear added 2026-06-29
                                                  (geyser_morning_override_midday_clear, geyser_automations.yaml)
input_boolean.geyser_manual_run_active         ← ON while manual run in progress
input_boolean.geyser_reached_temp_today        ← set when geyser_at_temperature → on; reset midnight
                                                  Used by geyser_daily_minimum_check (2026-06-17)
input_select.geyser_manual_run_duration        ← "30" or "60" minutes for manual run

# Window timing (added E4) — triggers are fixed at default values; see TRIGGER FLOOR NOTE
input_number.geyser_morning_start_weekday  h  4.5  (Mon–Sat trigger 04:30 non-winter, 04:00 winter)
input_number.geyser_morning_start_weekend  h  5.5  (Sunday trigger 05:30 non-winter, 05:00 winter)
input_number.geyser_morning_end_weekday    h  7.5  (trigger 07:30 non-winter, 08:00 winter)
input_number.geyser_morning_end_weekend    h  8.5  (trigger 08:30 non-winter, 09:00 winter)
input_number.geyser_winter_start_offset    min 30  (subtracts from start, adds to end in winter)

# Midday solar gate (added E4)
input_number.geyser_midday_surplus_threshold W 300  (geyser draws 1.25 kW; 300W = solar covers most)
input_number.geyser_last_heat_up_minutes    min  0  elapsed minutes from switch-on to at-temperature

# Daily minimum and adaptive evening (added 2026-06-17; fixed 2026-06-18; raised 2026-06-24)
input_number.geyser_adequate_daily_energy_by_midday  kWh 3.0   gate for evening early start.
                                                               IMPORTANT: condition compares DELTA
                                                               (midday_end − morning_end), not raw midday_end.
                                                               Raw midday_end includes morning run energy →
                                                               condition was always false on normal days.
                                                               Fixed 2026-06-18 (1.5→1.25 kWh, ~1h runtime).
                                                               Raised 2026-06-24: 1.25 kWh was less than half
                                                               a real reheat cost. geyser_last_heat_up_minutes
                                                               history (5 days, excl. resets): median 136 min,
                                                               mean 147 min → 1.25kW × ~2.3-2.45h = 2.8-3.1 kWh
                                                               per full heat-up. Now 3.0 kWh — biases toward
                                                               requiring midday (cheap/solar window) to do
                                                               nearly all of that work before deferring to the
                                                               likely grid-funded 18:30 evening fallback. Also
                                                               relevant given 200L tank capacity vs 3-person
                                                               shower blocks (morning + evening) already near
                                                               capacity — see geyser_heat_up_duration_capture.
input_number.geyser_thursday_high_usage_extra_kwh    kWh 1.5   added to the threshold above on Thursdays only
                                                               (added 2026-07-03). Thursday is a maid day
                                                               (presence_trust.yaml, weekday: [mon, thu],
                                                               10:00-17:45) — extra daytime hot-water use draws
                                                               the tank down more than normal, so the same
                                                               midday kWh delta means less actual heat left by
                                                               evening. Incident: 2026-07-02 midday delta 4.25
                                                               kWh read "Adequate" against the flat 3.0 kWh
                                                               threshold, early start (17:00) didn't fire,
                                                               18:30 fallback left tank not hot by ~20:00
                                                               showers. Effective Thursday threshold: 4.5 kWh
                                                               (would have caught the 4.25 kWh incident).
                                                               User confirmed scope: Thursday only, not Monday
                                                               (the other maid day) — raise threshold approach,
                                                               not an unconditional bypass.
input_number.geyser_min_daily_energy_kwh             kWh 2.0  trigger threshold for 20:00 backup check
input_number.geyser_energy_at_morning_end            kWh      snapshot of energy_day at morning hard-off
input_number.geyser_energy_at_midday_end             kWh      snapshot at 15:00 (midday hard-off)

# Midday forced minimum (added 2026-06-20) — see "Midday forced minimum" note below
input_number.geyser_midday_forced_minutes_winter           min  60  winter, normal weekday
input_number.geyser_midday_forced_minutes_winter_extended  min  90  winter, staff_on_site OR weekend
input_number.geyser_midday_forced_minutes_summer           min  30  any non-winter day

# Morning extension (added 2026-06-21, reworked 2026-07-06) — see "Morning extension" note below
input_boolean.geyser_morning_extend_enabled    master toggle, default on
input_boolean.geyser_morning_extend_override   manual same-day trigger, default off — reset 00:01
input_number.geyser_morning_extend_max_hour    h    9.0   safety cap, cold/poor-solar trigger, non-maid day
input_number.geyser_morning_extend_maidday_hour h   10.0  safety cap, cold/poor-solar trigger, maid day (Mon/Thu)
input_number.geyser_holiday_extend_max_hour    h    13.0  safety cap, holiday_mode OR manual override trigger
input_number.geyser_cold_ambient_threshold_c   °C   14  default
input_boolean.geyser_morning_extended_today    internal flag — reset 00:01
input_boolean.holiday_mode                     (presence_trust.yaml) — alternate extension trigger
```

**Derived sensors (power_state.yaml):**
```
sensor.geyser_control_status         13-state priority display
binary_sensor.geyser_at_temperature  ON after 5 min sustained geyser_heat_pump_power < 50W while switch on
sensor.geyser_daily_status           reached_temp / heating / low_energy / no_run  (added 2026-06-17)
                                     Attributes: energy_today_kwh, morning/midday/evening_kwh,
                                     reached_temp_today, at_temperature_now, min_energy_kwh,
                                     midday_adequacy (added 2026-07-01):
                                       Before 15:00: "In progress — X.X kWh (need Y.Y)"
                                       After 15:00, adequate: "Adequate (X.XX / Y.Y kWh) — early start 17:00/17:30"
                                       After 15:00, low: "Low (X.XX / Y.Y kWh) — 18:30 fallback"
                                       Season-aware: winter early = 17:00, non-winter = 17:30.
                                       Threshold: input_number.geyser_adequate_daily_energy_by_midday (3.0 kWh).
```

**Full evening schedule (updated 2026-06-24):**
```
Evening turn-on — ADAPTIVE + SEASON-AWARE (updated 2026-06-24):
  Condition: (midday_end − morning_end) < adequate_threshold (3.0 kWh)
    BUG FIX 2026-06-18: was comparing raw midday_end (cumulative daily) → morning run alone
    exceeded threshold → 17:00/17:30 never fired. Now compares midday-window DELTA only.
    THRESHOLD RAISED 2026-06-24: 1.25→3.0 kWh — grounded in real heat-up cost (see helper note above).

  17:00 (evening_early_winter)  WINTER only: if NOT at_temperature AND midday delta < threshold
                                → extra time needed: showers at 19:30 require 2.5h heat-up
  17:30 (evening_early)         NON-WINTER: if NOT at_temperature AND midday delta < threshold
                                → start early, accept 17:30–18:30 cooking-peak overlap
  18:30 (evening_late)          ALL seasons: if NOT at_temperature AND switch off — fallback
                                → fires if 17:00/17:30 branch didn't run (adequate midday)

Evening hard-off:
  Normal winter     : 21:00 (9pm)
  Normal non-winter : 20:30 (8:30pm)
  Sports winter     : 22:00 (10pm)   ← sports_night extends by 1h
  Sports non-winter : 21:30 (9:30pm) ← sports_night extends by 1h
```

**Full schedule (all windows):**
```
Morning  Mon–Sat winter    : 04:00 on / 08:00 off
         Mon–Sat non-winter: 04:30 on / 07:30 off
         Sunday winter     : 05:00 on / 09:00 off
         Sunday non-winter : 05:30 on / 09:00 off
Midday   (solar-gated)     : 12:00/13:00/14:00 on → 15:00 off
Evening  (adaptive)        : 17:00 winter / 17:30 non-winter / 18:30 fallback → hard-off varies by season/sports
```

**Automations (geyser_automations.yaml):**
```
automation.geyser_turn_on  (6 branches — morning ×3, midday, evening_early_winter, evening_early, evening_late)
  Morning          : NON-NEGOTIABLE — only blocked at loadshedding_critical
                     geyser_morning_override bypasses load_control_geyser_enabled
  Midday           : SOLAR-GATED — orchestrator [surplus, normal], solar > 300W, NOT at temp, before 15:00
  Evening early winter (17:00): winter only — fires if NOT at_temp AND midday delta < 3.0 kWh
                     (4.5 kWh on Thursdays — maid-day high-usage bump, added 2026-07-03)
  Evening early (17:30): non-winter — fires if NOT at_temp AND midday delta < 3.0 kWh
                     (4.5 kWh on Thursdays — same bump applies regardless of season)
  Evening late (18:30) : fires if NOT at_temperature AND switch off (all-seasons fallback)

automation.geyser_turn_off  (7 branches + default, mode: queued)
  AM protection    : 05:30/06:30/07:30/08:30 — grid off + SOC < prog2_soc + shedding (safety floor)
  Morning hard-off : weekday 07:30 non-winter / 08:00 winter; Sunday 09:00 both seasons
  Midday hard-off  : 15:00 unconditional
  Evening winter   : 21:00 — winter + sports_night off
  Evening standard : 20:30 — non-winter + sports_night off
  Sports night win : 22:00 — winter + sports_night on
  Sports night std : 21:30 — non-winter + sports_night on
  Emergency off    : orchestrator → loadshedding_critical (outside morning window only)

automation.geyser_reached_temp_tracker  (added 2026-06-17)
  Trigger: geyser_at_temperature → on → set geyser_reached_temp_today ON
  Trigger: 00:01 daily → reset geyser_reached_temp_today OFF

automation.geyser_period_energy_snapshot  (added 2026-06-17; alerts added 2026-06-18)
  Captures geyser_heat_pump_energy_day into:
    geyser_energy_at_morning_end at morning hard-off (08:00 winter / 07:30 non-winter)
    geyser_energy_at_midday_end at 15:00
  Feeds per-period breakdown in sensor.geyser_daily_status
  Morning alert (added 2026-06-18): if morning_kwh < 0.5 AND NOT reached_temp_today
    → severity: warning "Geyser morning run produced almost nothing (X kWh)"
    → tells user tank is cold and evening early start will fire
  Midday alert (added 2026-06-18): if midday_delta < adequate_threshold AND NOT reached_temp_today
    → severity: information "Poor midday — early evening start at 17:00/17:30"
    → midday_delta = midday_end − morning_end (midday-window energy only)

automation.geyser_daily_minimum_check  (added 2026-06-17)
  Trigger: 20:00 daily
  If NOT reached_temp AND energy_day < min_daily_energy_kwh AND switch off → turn on + notify warning
  If NOT reached_temp AND switch already on → notify info (still heating, runs to hard-off)
  If reached_temp → logbook only (all good)

automation.geyser_heat_up_duration_capture  (added 2026-06-15 — unchanged)
automation.geyser_sports_night_scheduler    (ON: Tue + Thu 17:00. OFF: 00:01 daily)
automation.geyser_manual_run               (timed 30/60 min run via script.geyser_manual_run)
```

**Orchestrator gate policy:**
- Morning + evening windows: blocked ONLY at `loadshedding_critical`.
- Midday: blocked when orchestrator NOT in `[surplus, normal]`. Forced-minimum branch (below) ignores this gate — only blocked at `loadshedding_critical`, same as morning/evening.
- Emergency off: fires on `loadshedding_critical` OUTSIDE morning window.
- 20:00 minimum check: NOT blocked by orchestrator (hot water essential) — only skips on `loadshedding_critical`.

**Midday forced minimum (added 2026-06-20):** Incident — poor-solar winter day (2026-06-20) left the tank cold; manual run needed at ~15:30. Branch 2 (solar-gated) can go all midday without firing on bad-solar days. Branch 2b backstops this: fixed triggers at 13:30/14:00/14:30 force the geyser on (bypassing the solar gate) if, by the time needed to guarantee the required runtime before the 15:00 hard-off, the switch is still off and not at temperature.

Required minutes (`input_number.geyser_midday_forced_minutes_*`, power_helpers.yaml):
```
geyser_midday_forced_minutes_winter            min  60  default — winter, normal weekday
geyser_midday_forced_minutes_winter_extended   min  90  default — winter AND (staff_on_site OR weekend)
geyser_midday_forced_minutes_summer            min  30  default — any non-winter day (raise to 60 via UI if needed)
```
Force-on time = 15:00 − required minutes (90→13:30, 60→14:00, 30→14:30), matched against the fixed trigger slots with ±15 min tolerance (TRIGGER FLOOR pattern). Uses `binary_sensor.staff_on_site` (derived, never the manual `input_boolean`) and `now().weekday() >= 5` for weekend. Notifies severity: warning (explicitly flags solar gate was bypassed).

**Sticky-flag bug fix (same day, 2026-06-20):** `geyser_period_energy_snapshot`'s midday alert and `geyser_daily_minimum_check`'s 20:00 "all good" branch both used to gate on `input_boolean.geyser_reached_temp_today`, which only resets at midnight. A geyser that reached temperature once overnight/early-morning stayed flagged on for the rest of the day even after the tank cooled from daytime use — silently suppressing both checks (root cause of a missing midday notification despite the tank being cold by 15:30). Both now use `binary_sensor.geyser_at_temperature` (real-time) for the actual decision; the sticky flag is kept only in log/notification text for context.

**Midday safety-net auto-trigger (added 2026-06-20):** The midday alert in `geyser_period_energy_snapshot` (15:00) is escalated from severity `information` to `critical` and now auto-triggers a forced 60-minute run — via the existing manual-run mechanism (`input_select.geyser_manual_run_duration` = "60" + `input_boolean.geyser_manual_run_active` → on) — when the tank is not at temperature and the midday delta is below threshold. This is a backstop for the case where Branch 2b (forced minimum, above) doesn't end up running the geyser. Gated by: switch off, `load_control_geyser_enabled` on, `geyser_manual_run_active` off, orchestrator not `loadshedding_critical`.

**Morning extension (added 2026-06-21, reworked 2026-07-06):** Incident — cold (11°C), heavily overcast winter day (`solar_weather_correlation` degraded), more than one person home all day — the morning run alone wasn't enough; tank cold again by 11am showers. `geyser_turn_off` Branch 1 checks, at the normal morning hard-off, whether the geyser is STILL ACTUALLY HEATING (`binary_sensor.geyser_at_temperature == 'off'`, i.e. power draw still ≥ 50W) AND any of: (a) the cold trigger — `sensor.season == 'winter'` AND `state_attr('weather.openweathermap','temperature') < input_number.geyser_cold_ambient_threshold_c` (14°C default) AND `sensor.solar_weather_correlation in ['poor','degraded']` AND **more than one** family member in a home AP zone (inline count over `sensor.{ryan,vicky,luke,tayla}_ap_location`, not the universal-quantifier `binary_sensor.all_family_home` — a single person home alone doesn't need the extra capacity); (b) `input_boolean.holiday_mode == 'on'`; (c) `input_boolean.geyser_morning_extend_override == 'on'` (manual, same-day equivalent of holiday_mode for this logic only — doesn't touch security escalation or bedtime scheduling). If so (and `input_boolean.geyser_morning_extend_enabled` is on), it skips the turn-off and sets `input_boolean.geyser_morning_extended_today`.

2026-07-06 rework: the actual turn-off is no longer a fixed timer — Branch 1b now turns the geyser off as soon as it actually stops heating (`binary_sensor.geyser_at_temperature` → `on`), triggered directly off that state change. Three safety-cap time triggers are fallbacks only, in case the sensor never trips (e.g. an element fault) — hitting one fires a **warning**-severity notification instead of the normal information one, since it likely means something's wrong rather than just a slow reheat:
- **09:00** (`geyser_morning_extend_max_hour`) — cold/poor-solar trigger, normal day. Covers a later morning gym-then-shower slot or a longer winter shower without running all the way to 11:00; the dynamic stop-on-heating-complete mechanism already prevents wasting grid power once the tank is actually hot, so this is purely how late the fallback can run.
- **10:00** (`geyser_morning_extend_maidday_hour`) — cold/poor-solar trigger, on a maid day (Mon/Thu, `presence_trust.yaml` schedule) — extra household hot-water demand those days can need the extra hour.
- **13:00** (`geyser_holiday_extend_max_hour`) — holiday_mode or manual override. Later because there's no wake-up/school run — mornings behave more like a lazy weekend, where a shower could land mid-morning or even early afternoon; the incident that prompted this path (2026-07-06) was a cold bath ~10am on a holiday morning.

`geyser_morning_extended_today` and `geyser_morning_extend_override` both reset at 00:01 alongside `geyser_reached_temp_today`. The extended window is NOT treated as "sacred" by the orchestrator-emergency branch (Branch 7) — a `loadshedding_critical` event during the extension still cuts power, unlike the true morning window. Note: `input_boolean.holiday_mode` is a shared, multi-domain toggle (already used by `security_logic.yaml` for threat escalation and `lighting_bedtime.yaml` for bedtime scheduling) — turning it on for a holiday affects those too. `geyser_morning_extend_override` was added so a single day's extension doesn't require flipping that shared toggle.

**Earlier opportunistic midday trigger (added 2026-06-21):** Added an 11:00 trigger to `geyser_turn_on`'s midday triggers (alongside existing 12:00/13:00/14:00), reusing Branch 2's solar-gate logic unchanged — on bad winter days the Solcast peak can land ~10-11am rather than midday, so this catches a brief surplus window that the original 12:00 start would miss.

**Canonical entity — solar surplus:**
`sensor.solar_available_surplus` (Watts, power_state.yaml) — used for midday solar gate.
Do NOT use `sensor.solar_surplus_available` (solar_core.yaml — returns `"True"`/`"False"` string, useless for thresholds).

**Trigger floor limitation:**
HA time triggers cannot read input_number at runtime. Triggers are fixed at values matching helper defaults. Adjusting a helper within ±15 min of its trigger time works. Larger adjustments also require updating the matching trigger line in geyser_automations.yaml.

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

### Energy Simulator (pyscript/energy_simulator.py + power_helpers.yaml — E8 2026-06-14)

Read-only scenario simulator — evaluates orchestrator and appliance decisions using simulated inputs without touching any live entity. Output goes to `persistent_notification.energy_simulator` and fires `event.energy_simulator_result`.

**Safety gate:** `input_boolean.simulator_active` must be ON to run. Defaults to OFF.

**Helpers (power_helpers.yaml):**
```
input_boolean.simulator_active              default: false — safety gate
input_select.simulator_scenario             8 presets + "Custom"
input_number.simulator_soc_override         % (0–100, step 5) — used in Custom mode
input_number.simulator_solar_override       W (0–15000, step 100)
input_number.simulator_load_override        W (0–8000, step 100)
input_number.simulator_hour_override        h (0–23, step 1)
input_number.simulator_shed_stage_override  0–6
```

**Scenario presets:**
| Scenario | SOC | Solar | Load | Hour | Shed |
|---|---|---|---|---|---|
| Normal sunny day | 75% | 5000W | 2000W | 12:00 | 0 |
| Poor solar — rain | 45% | 800W | 2500W | 13:00 | 0 |
| Load shedding stage 2 | 65% | 3000W | 2000W | 14:00 | 2 |
| Load shedding stage 4 | 40% | 1000W | 2500W | 16:00 | 4 |
| Critical battery morning | 18% | 0W | 1800W | 05:00 | 0 |
| Tennis night — low solar | 55% | 0W | 2000W | 21:00 | 0 |
| Post battery upgrade — surplus | 85% | 8000W | 2500W | 11:00 | 0 |
| Custom | simulator_*_override inputs | | | | |

**What it simulates:**
- Orchestrator 6-state priority logic (mirrors energy_state.yaml)
- Geyser: which window is active at sim_hour, orchestrator gate, midday solar gate
- Pool: solar surplus headroom, window gate, winter morning hold, last-sun-slot
- Borehole: water_refill_allowed equivalent (grid + SOC + orchestrator + solar window)
- Air fryer: grid-off + SOC-low cut condition
- Inverter programme: Battery/Load First crossover, P4 grid charge evaluation

**Live threshold reads:** All `input_number.orchestrator_*` thresholds are read at runtime — changing a threshold immediately affects simulation output without editing the pyscript.

**Call via:** `Developer Tools → Services → pyscript.energy_simulator`

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
| `sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c` | Load Shedding | `load_shedding_automations.yaml`, `prepaid_strategy.yaml` |
| `binary_sensor.anyone_connected_home` | Presence | `energy_orchestrator_state` |
| `binary_sensor.night_confirmed` | Context | `battery_runtime.yaml`, `grid_risk.yaml` |

### Legacy automations.yaml Power Automations

**✅ Resolved 2026-06-18 — `automations.yaml` reduced to 0 active automations.**

All legacy power automations were migrated or deleted in the 2026-06-18 session:

| Automation ID | Purpose | Outcome |
|---|---|---|
| `grid_status_monitoring` | Grid disconnect/reconnect notifications | **Deleted** — wrong entity (`sensor.inverter_power`); superseded by package sensors |
| `inverter_pwer_monitoring` | Load/battery overload alerts | **Deleted** — same wrong entity; superseded |
| Load Shedding Inverter Scene Switcher | Changes inverter scene based on LS level | **Deleted** — `select` actions were `enabled:false`, scene targets never existed; superseded by `automation.inverter_energy_pattern_control` (E5-2) |
| Grid Import Warning per day | Daily grid kWh threshold alerts | **Migrated** → `automation.grid_import_daily_warning` in `power_automations.yaml` (dead entity refs cleaned) |

---

## 11. Known Issues

Issues ordered by risk/impact. **P1 = breaks functionality. P2 = incorrect data. P3 = performance/cleanup.**

---

### Issue 0 — ✅ RESOLVED 2026-06-23: `power_statistics.yaml` duplicate `template:` key silently dropped 6 sensors
**Priority:** P1 — silently broke functionality with no config-check error  
**File:** `packages/power/power_statistics.yaml`  
**Symptom:** Dashboard "Solar Performance Stats" card showed `Unavailable` for 7-Day Accuracy %, 7d Mean Production, 30d Mean Production, 7d Std Dev. Initially misdiagnosed as a post-restart warm-up delay (entities had `restored: true` and a stale `last_changed` timestamp matching a recent reload) — turned out to be permanent, not transient.  
**Root cause:** The file had **two top-level `template:` keys**. PyYAML (and HA's loader) silently keeps only the **last** value for a duplicate mapping key — the entire first `template:` block was dropped from the loaded config on every reload, with no validation error since the YAML was syntactically valid. Casualties: `sensor.inverter_production_7d_mean` / `30d_mean` (defined in the dropped block), plus four `platform: statistics` sensors (`inverter_production_7d_stdev`, `house_load_24h_mean`, `house_load_7d_mean`, `solar_forecast_accuracy_7d`) that were **also** incorrectly nested inside that same dropped `template:` list using `platform:` keys — invalid under the `template:` integration schema (that syntax belongs to the legacy `sensor:` platform-list, used correctly elsewhere in the same file).  
**Downstream impact:** `sensor.solar_weather_correlation`'s 7-day trend inputs (`ratio_7d`, `stdev`, `cv`) all silently defaulted to neutral values (100/0/0) via Jinja `|float()` fallbacks — it could only ever reach `degraded` via the same-day `ratio_today` branch, never via sustained 7-day under-performance, for as long as this was broken. `house_load_24h_mean` being unavailable also fed directly into the P4 grid-charge shortfall calc (`inverter_p4_grid_charge_control`), defaulting load to 0 and making the shortfall estimate slightly optimistic. Unknown exact duration — file's changelog comments suggest the split happened across the 2026-06-14 P6 session and the 2026-06-18 daily-snapshot edit.  
**Fix:** Restructured into ONE `sensor:` list (6 `platform: statistics` sensors) and ONE `template:` list (5 derived template sensors) — no behavioural change, just makes the config load as originally written. Verified via `yaml.safe_load()` that all 11 sensors now parse into the correct top-level key.  
**Lesson:** A second `!include`'d file defining the same top-level key as an earlier one in the same package is fine (HA merges those at the package level) — but **within a single file**, a repeated top-level key is a silent data-loss bug, not a merge. `python3 -c "import yaml; yaml.safe_load(open(f))"` validates syntax but will NOT catch this — checking `len(d['template'])` / `len(d['sensor'])` against the expected sensor count would have caught it.  
**Deploy gotcha #1 (2026-06-23/24):** `platform: statistics` sensors (legacy `sensor:` syntax) are **not** in HA's hot-reloadable set (unlike `automation:`/`script:`/`template:`/`input_*:`) — they seed from recorder history at startup only, so a "Check Configuration" + partial reload left `7d_mean`/`30d_mean` stuck `unavailable` (with `restored: true` and a frozen `last_changed`) for over a day even though the fix was already on disk and validated. Needed a full HA restart to actually take effect.  
**Deploy gotcha #2 (2026-06-24, user-fixed):** after the restart, the corrected `inverter_production_7d_mean`/`30d_mean` came back registered as `_2` duplicates (e.g. `sensor.inverter_production_7d_mean_2`) instead of reclaiming their original `entity_id`. Cause: moving a `unique_id` between platform sections (legacy `sensor:` ↔ `template:`) while the entity registry still has a row for that `unique_id` tied to the old platform can make HA register it as a new entity rather than reclaiming the old `entity_id`. Fixed by removing the orphaned registry entries (Settings → Devices & Services → Entities → search, delete the stale non-`_2` rows) so the correct entities reclaimed their proper names.

---

### Issue 0b — ✅ RESOLVED 2026-06-24: `battery_runtime_status_card` showed "Battery charging – target unknown%"
**Priority:** P2 — wrong dashboard text, no functional impact  
**File:** `packages/power/battery_runtime.yaml`  
**Symptom:** Dashboard tile read "Battery charging – target unknown%" whenever the battery was charging.  
**Root cause:** The "charging" branch read `{{ charge_eta }}%`, where `charge_eta = states('sensor.markdown_ss_battery_charge_time_left')` — an entity that doesn't exist, so it always resolved to `unknown`. The correct value, `target_soc` (= `sensor.ss_battery_capacity`, the program-aware target SOC — e.g. reads 90% during the P4 window if that's what's configured), was already computed one line above and used correctly in the *other* branch of the same template — just never wired into this one. Classic copy-paste/wrong-variable bug.  
**Fix:** Changed to `{{ target_soc }}%`. Removed the now-dead `charge_eta` variable (its only consumer was this bug).

---

### Issue 1 — ✅ RESOLVED 2026-06-19: `battery_night_survival` stale BROKEN label
**Priority:** P1 — was believed unavailable  
**File:** `packages/power/energy_state.yaml`  
**Resolution:** Both dependency sensors ARE implemented in energy_state.yaml: `sensor.battery_energy_available` (SOC% − shutdown%) × 48,230 Wh and `sensor.average_night_consumption` (platform:statistics mean 24h on inverter_load_power, sampling_size removed). Bug label was stale — sensors implemented in Session A 2026-04-21 and sampling_size bug fixed subsequently. Sensor is operational.

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

### Issue 8 — RENAMED: `group.security_power_sensors` → `group.house_security_power_sensors`
**Priority:** P2 — ⚠️ Group defined but may have no member entities  
**File:** `packages/power/power_templates.yaml`  
**Status (2026-06-19):** Group `group.house_security_power_sensors` IS defined in `power_templates.yaml:106`
and referenced at `power_templates.yaml:187`. The contract previously reported the wrong group name
(`security_power_sensors`). The group exists but whether it has member entities (smart plugs on
security devices) is not confirmed — `sensor.house_security_power` may still return 0.  
**Remaining action:** Populate `group.house_security_power_sensors` with actual security device
power sensors (NVR, IP cameras, alarm panel) or remove `sensor.house_security_power` if security
load monitoring is not needed.

---

### Issue 9 — ✅ RESOLVED 2026-06-19: Missing solar forecast helper entities added
**Priority:** P2 — `solar_forecast_update_inverter_mode_daily` automation uses undefined thresholds  
**File:** `packages/power/solar_helpers.yaml`  
**Resolution:** `input_number.low_solar_forecast_trigger` (default 10 kWh),
`input_number.high_solar_forecast_trigger` (default 25 kWh), and
`input_select.inverter_solar_mode_helper` (options: Low/Medium/High, default Medium) added to
`solar_helpers.yaml`. Watchman noise cleared. The `solar_forecast.yaml` inverter mode automation
remains gated off by `input_boolean.use_legacy_solar_scenes = off` (default) — helpers are defined
but the automation is dormant unless the legacy scenes gate is enabled.

---

### Issue 10 — ✅ RESOLVED 2026-06-19: Wrong Solcast entity name
**Priority:** P2 — Solar forecast sensors always unavailable  
**File:** `packages/power/solar_forecast.yaml`  
**Resolution:** `solar_forecast.yaml` now uses `sensor.solcast_pv_forecast_forecast_today` throughout
(verified at lines 37, 147, 174, 189, 203, 230, 243, 251, 277, 290). All references corrected.

---

### Issue 11 — ✅ RESOLVED 2026-06-18: `automations.yaml` wrong entity — moot (automations deleted)
**Priority:** P2 — Grid disconnect notifications silently broken  
**Resolution:** `automations.yaml` was fully cleaned out in the 2026-06-18 session (all 7 active
automations migrated or deleted — see PROJECT_STATE session log). `grid_status_monitoring` and
`inverter_pwer_monitoring` (which referenced `sensor.inverter_power`) were among the deleted
automations. The file now contains 0 active automations. This issue is moot.

---

### Issue 12 — ✅ RESOLVED 2026-04-21: `load_shedding_minutes_remaining` unique_id typo corrected
**Priority:** P3 — Functional but cannot be referenced by unique_id  
**File:** `packages/load_shedding/load_shedding_templates.yaml` (content migrated from `load_shedding.yaml`)  
**Resolution:** `unique_id` corrected from `load_hedding_inutes_emaining` to `load_shedding_minutes_remaining`
on 2026-04-21. Comment in file at line 117–118 confirms the correction. Entity now registered under
the correct unique_id.

---

### Issue 13 — ✅ RESOLVED 2026-06-19: `grid_risk.yaml` wrong entity corrected
**Priority:** P2 — grid_risk sensors degrade silently when reading wrong/missing entity  
**File:** `packages/power/grid_risk.yaml`  
**Resolution:** `grid_risk.yaml` now uses `sensor.inverter_load_power` (verified at line 79).
No references to `sensor.inverter_power` remain in the file.

---

### Issue 15 — ✅ RESOLVED 2026-04-16: `sensor.prepaid_total_spent`
**Priority:** P3 — Dashboard shows unavailable  
**Root cause:** Dashboard referenced `sensor.prepaid_total_spent` which doesn't exist. The correct entity is `input_number.prepaid_total_spent` (defined in prepaid_helpers.yaml).  
**Fix applied:** Two dashboard references in `lovelace.dashboard_operations` (Energy Spend + Spend traces in plotly-graph cards) updated to `input_number.prepaid_total_spent`.

---

### Issue 16 — ✅ RESOLVED 2026-05-27: Load groups wiped by pyscript at startup
**Priority:** P3 — Power templates that sum group states show unknown  
**File:** `pyscript/sync_power_groups.py`, `packages/power/power_templates.yaml`  
**Root cause:** `group.flexible_power_loads` and `group.critical_power_loads` are defined in YAML (power_templates.yaml) but pyscript wiped them to empty `[]` on every startup. YAML-defined members were correct; pyscript overwrote them. `group.known_power_loads` "unknown" state is expected — HA groups of numeric sensors always report "unknown" state, but `expand()` in templates still iterates members correctly.  
**Fix applied 2026-05-27:** Removed the two wipe lines from pyscript. pyscript now only maintains `group.real_power_loads` (auto-detected). YAML owns `flexible_power_loads` and `critical_power_loads` — members survive restarts.

---

### Issue 17 — DESIGN DEBT: `power_helpers.yaml` violates layering convention
**Priority:** P3 — Maintenance clarity  
**Root cause:** File contains both `group:` definitions (helpers layer role) and `template: sensor:` blocks (templates layer role). Violates the `*_helpers.yaml` → only helpers rule from CODING_STANDARDS.md.  
**Fix (non-urgent):** Split into `power_groups.yaml` (group definitions) + merge templates into `power_templates.yaml`. Do not do this unless refactoring — splitting risks breaking references.

---

### Issue 18 — ✅ RESOLVED 2026-07-06: power critical/Telegram notifications silently failing + power warning-tier alerts had zero delivery
**Priority:** P1 — critical alerts (incl. prepaid-critical) were not reaching any phone
**Files:** `packages/notifications/notify_power_event.yaml`, `packages/alerts/alerts_power.yaml`

**Bug 1 — critical branch silently broken:** `notify_power_event.yaml`'s critical branch
still called `notify.send_message` with a nested `data: {push, channel, ttl, priority}`
block — the "extra keys not allowed @ data['data']" bug fixed everywhere else on
2026-07-01 (security/water/system/lighting), but power was missed. `continue_on_error: true`
swallowed the HTTP 400 silently. Real impact: `power_battery_soc_critical_alert`
(power_automations.yaml) and both prepaid-critical automations
(`prepaid_critical_night_protection_alert` in prepaid_core.yaml,
`prepaid_buy_decision_notify` in prepaid_strategy.yaml) had been failing to reach any phone
for critical severity. **Fix:** converted to the legacy per-device `notify.mobile_app_*`
pattern already proven in `notify_security_events.yaml`.

**Bug 2 — Telegram inline_keyboard also rejected:** discovered live while testing Bug 1's
fix — the Telegram mirror for critical severity called `notify.send_message` targeting
`notify.telegram_bot_5527` with `inline_keyboard` at top level of `data:`, the same
rejection SECURITY_CONTRACT.md's BUG-S61 already fixed for security, just never applied to
power. Confirmed firing live on both prepaid-critical automations via `ha core logs`. Mobile
push was unaffected (runs before this step); only the Telegram Acknowledge button/message
was silently missing. **Fix:** switched to `telegram_bot.send_message` directly.

**Bug 3 — `alert.power_alert` (STD_Alerts) zero delivery for warning tiers:** `notify.STD_Alerts`
has been broken since 2026-06-28 (see NOTIFICATIONS_CONTRACT.md §7). `alert.power_alert`'s
own entity_id anchor is `binary_sensor.power_grid_offline_alert_active` ONLY — so
`sensor.power_alert_context`'s warning-tier scenarios (evening/night risk while grid is
down but SOC still OK) never got a phone push at all, and battery-low-while-grid-up (a
distinct warning scenario the context sensor computes) could never even reach the alert
entity, since it isn't anchored on that binary sensor either (architectural gap, related to
the still-open "Entity Reference" note re: `power_alert_context.devices` — see historical
Issue tracking). Only the specific SOC-critical/low-solar combination had separate coverage
via `power_battery_soc_critical_alert`. **Fix:** new automation `Route Power Alert`
(`alerts_power.yaml`) triggers on grid-offline OR battery-low binary sensors going `on`, or
the context sensor reaching `critical`, and calls `script.notify_power_event` directly —
same pattern already proven for the temperature domain.

**Live incident during rollout (self-contained, fixed same session):** the first version of
`Route Power Alert` (and 12 sibling automations added for other domains — see
ALERTS_CONTRACT.md BUG-A10) used bare `to: "on"`/`to: "critical"` triggers with no
`from`/`not_from` guard. Reloading `template:` to activate them caused a wave of false
CRITICAL pushes — fixed same session with `from: "off"` / `not_from: ["unknown",
"unavailable"]` guards. Separately, the SAME reload also caused pre-existing
`prepaid_buy_decision_notify` and `prepaid_critical_night_protection_alert` to fire falsely
despite already having `not_from`/`not_to` guards — the real mechanism there is a
`template:` reload causing interdependent sensors to transiently read a default (e.g.
`| float(0)`) from an upstream sensor that hasn't recomputed yet, a valid-looking-but-wrong
value that `not_from` guards can't catch. **NOT fixed** (pre-existing, out of scope this
session) — would need an audit of both automations' upstream sensor chains for reload-safe
defaults. Operational takeaway: don't reload `template:` unless a `template:` sensor
actually changed.

---

## 12. Error Signatures (Watchman-Confirmed)

These entities appear in watchman_report.txt as missing or unavailable. Map these to the issues above.

### Missing Entities (will produce errors in HA logs)

| Entity | Status | References | Issue |
|---|---|---|---|
| `sensor.battery_energy_available` | ✅ implemented | energy_state.yaml | Issue 1 — RESOLVED 2026-06-19; SOC×capacity formula verified working |
| `sensor.average_night_consumption` | ✅ implemented | energy_state.yaml | Issue 1 — RESOLVED 2026-06-19; sampling_size removed, platform:statistics on inverter_load_power |
| `sensor.grid_power` | ✅ FIXED 2026-04-16 | replaced with sensor.house_grid_power | Issue 2 |
| `sensor.battery_charge_power` | ✅ FIXED 2026-04-16 | replaced with sensor.grid_to_battery_power | Issue 2 |
| `sensor.prepaid_units_left` | ✅ FIXED 2026-04-16 | replaced with sensor.prepaid_units_left_safe | Issue 3 |
| `group.security_power_sensors` | missing | power_templates.yaml:182 | Issue 8 |
| `sensor.solcast_forecast_forecast_today` | ✅ fixed | solar_forecast.yaml | Issue 10 — RESOLVED 2026-06-19; now uses `solcast_pv_forecast_forecast_today` |
| `sensor.inverter_1_device_since_last_update` | missing | power_state.yaml:279 | Solarman entity name change |
| `sensor.inverter_2_device_since_last_update` | missing | power_state.yaml (similar) | Solarman entity name change |
| `sensor.prepaid_total_spent` | ✅ FIXED 2026-04-16 | dashboard now references input_number.prepaid_total_spent | Issue 15 |
| `sensor.inverter_2_pv3_voltage` | missing | dashboard | PV3/4 strings don't exist on inv2 |
| `sensor.inverter_2_pv4_voltage` | missing | dashboard | PV3/4 strings don't exist on inv2 |

### Unknown State Entities (group state not resolved)

| Entity | Status | Root Cause |
|---|---|---|
| `group.known_power_loads` | unknown (expected) | HA groups of numeric sensors always report "unknown" state; expand() still works correctly |
| `group.flexible_power_loads` | ✅ FIXED 2026-05-27 | pyscript was wiping to [] on startup; fix in sync_power_groups.py — YAML members now survive |
| `group.critical_power_loads` | ✅ FIXED 2026-05-27 | pyscript was wiping to [] on startup; fix in sync_power_groups.py — YAML members now survive |
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

### Sprint 1 — Fix Silent Failures ✅ ALL DONE (verified 2026-06-14)

- [x] Fix `battery_night_survival` — ✅ uses `sensor.battery_energy_available` + `sensor.average_night_consumption` (energy_state.yaml:292)
- [x] Fix `grid_charging_while_solar` — ✅ uses `sensor.house_grid_power` + `sensor.grid_to_battery_power` (energy_state.yaml:318)
- [x] Fix `prepaid_depletion_date` — ✅ uses `sensor.prepaid_units_left_safe` (prepaid_strategy.yaml:141)
- [x] Fix `prepaid_buy_decision_notify` — ✅ condition checks `['buy_now', 'buy_soon']`; 'hold' state never existed
- [x] Fix duplicate unique_id `solar_surplus_available` — ✅ sensor renamed to `solar_surplus_available_power` (solar_core.yaml:47)

### Sprint 2 — Fix Wrong Entities ✅ ALL DONE (verified 2026-06-14)

- [x] Fix `inverter_1_pv_voltage` elif branch — ✅ already uses `sensor.inverter_1_pv1_voltage` (power_helpers.yaml:1023)
- [x] `inverter_today_energy_import * 2` — ✅ acknowledged workaround for Solarman Slave quirk; direct grid meter integration deferred (PROJECT_STATE)
- [x] `group.security_power_sensors` — ✅ group is `house_security_power_sensors`; template uses it correctly (power_templates.yaml:187)
- [x] Solar helpers — ✅ `low_solar_forecast_trigger`, `high_solar_forecast_trigger`, `input_select.inverter_solar_mode_helper` all defined and wired (solar_forecast.yaml)
- [x] Fix Solcast entity — ✅ all automation triggers/actions use `solcast_pv_forecast_forecast_today`; stale template reference at solar_forecast.yaml:37 fixed 2026-06-14 → `sensor.solar_forecast_available_conservative` now reads correct entity
- [x] `sensor.inverter_1_device_since_last_update` — ✅ renamed to `sensor.inverter_device_1_since_last_update` (2026-04-21, power_state.yaml:315)
- [x] `load_shedding_minutes_remaining` typo — ✅ unique_id corrected 2026-04-21 (load_shedding_templates.yaml:121)
- [x] `grid_risk.yaml inverter_power` audit — ✅ no stale `sensor.inverter_power` references found

### Sprint 3 — Legacy Cleanup ✅ ALL DONE (verified 2026-06-14)

- [x] Migrate `grid_status_monitoring` — ✅ not in automations.yaml (migrated/removed)
- [x] Migrate `inverter_pwer_monitoring` — ✅ not in automations.yaml
- [x] Update dashboard references to `input_number.prepaid_total_spent` — ✅ fixed 2026-06-14: two plotly graph cards (Prepaid Usage & Cost + Solar Savings vs Cost) were still referencing non-existent `sensor.prepaid_total_spent`; corrected to `input_number.prepaid_total_spent`
- [x] Remove stale `ssa_*` sensor references from automations.yaml — ✅ none found

### Sprint 4 — Strategy Improvements (4/5 done — verified 2026-06-15)

- [x] Implement `sensor.prepaid_balance_confidence` — ✅ implemented (prepaid_core.yaml:370)
- [x] Implement `sensor.prepaid_net_position_this_month` — ✅ implemented (prepaid_strategy.yaml:230)
- [x] Build Buy Score v2 — ✅ `sensor.prepaid_buy_score` implemented with multi-factor scoring (days, solar, burn_rate, fixed_remaining); `sensor.prepaid_buy_decision` downstream (prepaid_strategy.yaml:149)
- [x] Add auto-reconciliation trigger — ✅ `automation.prepaid_auto_reconcile` (prepaid_core.yaml:524): triggers on `prepaid_meter_lifetime_import` change; gates on `prepaid_drift_percentage > prepaid_drift_threshold`; calls `script.prepaid_realign_offset` + notify
- [ ] Add pyscript load group health check + restart trigger — ⬜ OPEN: `sync_power_groups.py` only maintains `real_power_loads` on startup; no health check or restart trigger implemented

### Sprint 5 — Geyser Solar Control ✅ IMPLEMENTED E2+E4 2026-06-14

**Context:** Added 2026-05-25. Helpers and template sensors were already in place. Automation
implemented in E2 (migration) and upgraded in E4 (orchestrator gate + midday solar gate).
See Geyser Scheduling section in Section 8 for full documentation.

**Helpers already added:**
```
input_number.grid_offline_soc_min_geyser        %   non-critical window min (default: 45)
input_number.geyser_grid_offline_critical_soc   %   critical window min (default: 35)
input_number.geyser_target_hours_summer         h   daily solar target summer (default: 2.0)
input_number.geyser_target_hours_winter         h   daily solar target winter (default: 3.5)
sensor.geyser_target_run_hours_today            h   season-aware target
```

**UI helper needed (before implementing):**
```
sensor.geyser_run_hours_today — History Stats helper (same pattern as pool pump):
  Settings → Helpers → History Stats
  Entity: switch.geyser_heat_pump_switch, State: on, Type: Time
  Start: {{ now().replace(hour=0,minute=0,second=0,microsecond=0) }}, End: {{ now() }}
```

**Proposed control windows:**
- Morning critical (06:00–09:00): pre-heat for morning showers — run if grid ON OR SOC >= 35%
- Solar daytime (09:00–17:00): run on solar surplus >3000W (geyser ~2kW), all gates applied
- Evening critical (17:00–20:30): pre-heat for evening showers — run if grid ON OR SOC >= 35%
- Hard stop 20:30: unconditional off

**Non-critical window gates (solar daytime only):**
1. Grid offline AND SOC < 45% → block
2. Last sun slot (14:00+) AND SOC < last_sun_soc_target → block (battery charging priority)
3. Solar headroom < 3000W → block
4. Daily solar target already met → block (critical windows exempt from this)

**Design questions that need review before implementing:**
- Morning window: if grid is offline at 06:00 and SOC is e.g. 35% (borderline), do we run geyser?
  The 2kW heat pump for 2h = 4kWh from battery. At 35% SOC (11kWh available) that's 36% of battery.
  Probably correct NOT to run if SOC < 45% even in morning window — but this conflicts with shower need.
- Evening window: if Eskom is still out at 17:00, SOC might be higher (solar charged). Easier to gate.
- Integration with geyser schedule inverter programs — the inverter may already control geyser via
  Prog2/Prog3 time programs. Need to confirm whether switch.geyser_heat_pump_switch overrides those
  or conflicts with them. May need to disable inverter timer program first.
- Minimum run time: geyser heat pump takes ~15 min to reach operating temperature — add min_run_minutes?
- Geyser temperature sensor: if available, could replace time-based with temperature-based target.

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
| Battery capacity = 48,230 Wh (3 × 314Ah × 51.2V) | Calculated from label specs; used in battery_runtime.yaml + energy_state.yaml |
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

*Last updated: 2026-06-14 (Session F — dashboard pass)*  

---

## 16. Dashboard Architecture (Session F — 2026-06-14)

All power dashboards are in HA storage mode (`lovelace: mode: storage`). Files live in `/config/.storage/lovelace.*`. Edit via JSON while HA is stopped (stop → edit → start); the WebSocket API is the preferred method but requires a browser-based LLAT.

### Dashboard File Map

| Storage File | Dashboard | URL | Views |
|---|---|---|---|
| `lovelace.dashboard_operations` | Operations | /dashboard-operations | Power, Power Control, Appliance Control, Geyser Control, Inverter Control, Prepaid, Security, Presence, Lights, Water, Network, Media, Weather, Office, Mobile |
| `lovelace.operations_debug` | Debug | /operations-debug | Energy (power-history), Water debug, Energy Cost, Presence debug, Security debug |
| `lovelace.dashboard_overview` | Overview | /dashboard-overview | Home overview (power flow, water, presence, alerts) |
| `lovelace.dashboard_system` | System | /dashboard-system | System health, alerts |

### Power-Relevant Views

| View | Path | Purpose | Last Updated |
|---|---|---|---|
| Power Control | /dashboard-operations#power-control | Orchestrator brain, geyser+pool+borehole controls, inverter sync, forecast, load shedding | F 2026-06-14 |
| Appliance Control | /dashboard-operations#appliance-control | Pool pump detail (mushroom + 4-card grid + controls) | F 2026-06-14 |
| Geyser Control | /dashboard-operations#geyser-control | Geyser status, 4-card grid, controls with sliders, 48h logbook | F 2026-06-14 |
| Inverter Control | /dashboard-operations#inverter-control | Inv1 programme, Inv2 sync, load shedding scenes, orchestrator context, diagnostics, solar stats, season calibration | F 2026-06-14 |
| Energy / Power History | /operations-debug#power-history | Load vs SOC apexchart, solar vs weather charts, solar performance H1-H4, simulator panel, orchestrator debug | F 2026-06-14 |

### Entity Name Corrections (spec vs actual)

| Spec Name | Actual Entity | Notes |
|---|---|---|
| `sensor.house_known_load_power` | `sensor.known_load_power` | In power_templates.yaml |
| `sensor.house_unknown_load_power` | `sensor.unknown_load_power` | In power_templates.yaml |
| `sensor.house_load_visibility_percent` | `sensor.load_visibility_score` | In power_templates.yaml |
| `sensor.water_tank_level_percent` | `sensor.water_tank_level` | In water package |
| `sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark` | `sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c` | load_shedding integration entity ID. BUG FIXED 2026-06-28: `load_shedding_templates.yaml` was still hardcoded to the old name (3 templates) — this row documented the rename but the code never caught up. See "Load Shedding" section above. |
| `sensor.load_shedding_next_start` | `sensor.load_shedding_minutes_remaining` | No next_start entity in this integration version |

### Navbar

All power-related views use `custom:navbar-card` (footer position) with routes: Home → /dashboard-overview, Operations → /dashboard-operations, Debug → /operations-debug, System → /dashboard-system, Alerts → /dashboard-system/alerts.

### Known P6 Statistics Sensor Issue

Three statistics sensors defined in `power_statistics.yaml` are NOT appearing in the HA entity registry after two restarts:
- `sensor.inverter_production_7d_stdev` (standard_deviation of inverter_today_production)
- `sensor.house_load_7d_mean` (mean of inverter_today_load_consumption)
- `sensor.solar_forecast_accuracy_7d` (mean of solar_vs_forecast_ratio_today)

The YAML parses correctly. Pre-existing statistics sensors load as 'unavailable'. HA logs show no error. Investigation needed: check if statistics platform rejects standard_deviation characteristic in HA 2026.6, or if source entity incompatibility is the cause. Dashboard cards that reference these sensors will show "unavailable" — acceptable until resolved.
*Updated by: Deep audit — all 22 power package files read, POWER_DEPENDENCY_ANALYSIS.md incorporated, watchman cross-referenced, legacy automations checked*
*Last updated: 2026-06-19 — Issues 10/11/12/13 closed (code verified); Issue 8 corrected (group renamed house_security_power_sensors); watchman table updated; legacy automations table updated (all deleted/migrated 2026-06-18)*
