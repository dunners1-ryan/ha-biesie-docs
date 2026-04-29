# Power Package Dependency Analysis

## Executive Summary

The power package consists of 20 interconnected files across 7 subsystems. Key finding: **load_shedding is completely independent** with zero internal dependencies. All other subsystems form a tightly coupled graph centered on `power_core`.

---

## 🔴 Critical Architecture: Dual Measurement Systems

### Physical Meter vs Inverter Tracking
```
┌─────────────────────────────────────────────────────────────┐
│ PREPAID METER (Empire CIU EV-KP)                            │
│ ✓ Physically separate from inverter                         │
│ ✓ NOT networked or integrated                               │
│ ✓ AUTHORITATIVE for financial truth                         │
│ ✗ Manual script updates only (no real-time sync)            │
│                                                              │
│ Codes:                                                       │
│ • 1.8.0 = Lifetime total import (kWh)                       │
│ • 1.9.0 = Monthly import (kWh)                              │
│ • 98.2.1 = Yesterday import (kWh)                           │
│ • C.51.0 = Current remaining balance (kWh)                  │
└─────────────────────────────────────────────────────────────┘
                            ↕️ (manual sync via script)
┌─────────────────────────────────────────────────────────────┐
│ INVERTER TRACKING (Solarman + grid_energy_import_total)     │
│ ✓ Real-time grid power measurement                          │
│ ✓ Feeds into HA via Solarman integration                    │
│ ✗ NOT source of truth (no connection to physical meter)     │
│ ✗ Subject to sync lag, firmware quirks, restart drift       │
│                                                              │
│ Effective sensors:                                           │
│ • sensor.grid_energy_import_total ≈ 5,192 kWh              │
│ • sensor.grid_energy_import_today ≈ 40 kWh                 │
│ • prepaid_import_monthly (utility meter source)             │
└─────────────────────────────────────────────────────────────┘
```

### Measurement Reconciliation
The system uses **three-value alignment** to handle drift:

```
Source of Truth Hierarchy:
  1. input_number.prepaid_meter_lifetime_import      ← REQUIRED (manual entry)
  2. sensor.grid_energy_import_total                 ← OPERATIONAL (inverter, high-frequency)
  3. input_number.prepaid_alignment_offset           ← RECONCILIATION (auto-calculated)

Calculation:
  prepaid_units_used = (grid_import_total + alignment_offset) - prepaid_meter_baseline
  alignment_offset = prepaid_meter_lifetime - grid_import_total
```

### Known Drift Pattern
Daily reconciliation required because:
- **Physical meter updates** infrequently (daily or on-demand)
- **Inverter records** continuously (subject to resets, reboots, lag)
- **Measurement mismatch**: Inverter may measure AC-side, meter measures after losses
- **Expected variance**: ±1-3% monthly (grid losses, meter type differences)

**Monitoring**: Check `sensor.prepaid_grid_meter_drift` and `sensor.prepaid_drift_percentage` regularly. If drift > 5%, realign using "Realign Prepaid Offset" script button.

---

## Per-Subsystem Analysis

### 1. **PREPAID_*** (3 files: prepaid_core, prepaid_helpers, prepaid_strategy)

#### What it CONSUMES (states() calls)
- `sensor.grid_energy_import_total` ← From **energy_core**
- `sensor.grid_energy_import_today` ← From **energy_core**
- `sensor.solar_production_monthly` ← From **energy_helpers**
- `sensor.solcast_pv_forecast_forecast_remaining_today` ← From **solcast integration** (external)
- `sensor.prepaid_rolling_daily_burn` ← From **prepaid_strategy**
- `sensor.prepaid_adaptive_burn_rate` ← From **prepaid_strategy**
- `input_number.*` helpers from **prepaid_helpers**
- State attributes like `state_attr('sensor.prepaid_import_monthly','last_period')`

#### What it PRODUCES (unique_id definitions)
- `prepaid_units_used_since_last_update`
- `prepaid_units_used_authoritative`
- `prepaid_units_left_authoritative`
- `prepaid_units_left_safe`
- `prepaid_estimated_days_remaining`
- `prepaid_grid_meter_drift`
- `prepaid_drift_percentage`
- `prepaid_rolling_daily_burn`
- `prepaid_adaptive_burn_rate`
- `prepaid_topup_strategy`
- `prepaid_buy_decision`
- `prepaid_buy_score`
- (17 total sensors)

#### Internal Dependencies
```
prepaid_* 
  ↓ CONSUMES
  ├─ energy (grid_energy_import_*) 
  ├─ solar (solar_production_monthly via energy)
  └─ solcast (forecast data via states())
```

**Can be split?** Conditionally. Depends on whether `grid_energy_import_total` stays in `energy_core` or moves to `power_core`.

---

### 2. **SOLAR_*** (5 files: solar_core, solar_forecast, solar_helpers, solar_state, solar_clipping)

#### solar_core / solar_forecast / solar_state CONSUMES
- `sensor.house_solar_power` ← From **power_core** (aggregated inverter PV)
- `sensor.solcast_pv_forecast_forecast_today` ← From **solcast** (external)
- `sensor.solcast_pv_forecast_forecast_remaining_today` ← From **solcast**
- `sensor.inverter_pv_power`, `sensor.inverter_battery_soc`, `sensor.inverter_load_power` ← From **power_core**
- `sensor.solar_export_potential` ← From **solar_clipping** (internal)
- `sensor.solar_used_by_house_today` ← From **energy_core**
- `sensor.inverter_load_power` ← From **power_core**
- `sensor.load_shedding_stage_eskom`, `sensor.load_shedding_area_*` ← From **load_shedding** (external)

#### PRODUCES (unique_id definitions)
- `solar_system_status`
- `solar_power_available_conservative`
- `solar_forecast_available_conservative`
- `solar_forecast_next_hour`
- `solar_generation_confidence`
- `solar_stability`
- `solar_ramp_rate_kw_per_min`
- `solar_export_potential` ← **Consumed by energy_state, power_strategy**
- `solar_export_potential_today`
- `solar_export_value_today`
- `solar_surplus_available`
- `solar_unused_power`
- `solar_opportunity_level`
- `solar_efficiency_loss_percent`
- `solar_clipping_today` (integration sensor)
- (23 total +automations)

#### Internal Dependencies
```
solar_*
  ↓ CONSUMES
  ├─ power_core (house_solar_power, inverter_pv_power, inverter_battery_soc, inverter_load_power)
  ├─ energy_core (solar_used_by_house_today)
  ├─ load_shedding (forecast data for mode selection)
  └─ solcast (external forecast)
  
  ↓ PRODUCES → CONSUMED_BY
  ├─ solar_export_potential → energy_state, power_strategy
  ├─ solar_system_status → power_state
  ├─ solar_generation_confidence → power_state
  ├─ solar_stability → power_state
```

**Can be split?** NO — tight coupling to power_core (inverter sensors) and energy_core (solar_used_by_house).

---

### 3. **BATTERY_*** (2 files: battery_state, battery_runtime)

#### What it CONSUMES
- `sensor.inverter_battery_soc` ← From **power_core**
- `sensor.inverter_battery_power` ← From **power_core**
- `is_state('group.inverter_grid','on')` ← From **power_helpers** (group definition)
- `number.inverter_1_battery_shutdown_soc` ← Hardware (inverter registers)
- `number.inverter_1_program_*_time`, `number.inverter_1_program_*_soc` ← Hardware (program settings)

#### What it PRODUCES
- `battery_state_health`
- `ss_battery_capacity`
- `ss_soc_battery_time_left` ← **Consumed by grid_risk, energy_state**
- `ss_soc_battery_time_left_friendly`
- `battery_runtime_status_card`
- `battery_runtime_severity` ← **Consumed by grid_risk**
- `battery_minutes_remaining_safe` ← **Consumed by power_state, grid_risk, energy_state**
- `battery_night_survival`
- `markdown_ss_discharge_time`
- (15 total)

#### Internal Dependencies
```
battery_*
  ↓ CONSUMES
  └─ power_core (soc, battery_power, grid status)
  
  ↓ PRODUCES → CONSUMED_BY
  ├─ battery_runtime_severity → grid_risk
  ├─ battery_minutes_remaining_safe → power_state, grid_risk, energy_state
  └─ ss_soc_battery_time_left → grid_risk
```

**Can be split?** NO — provides critical safety signals (runtime, severity) to power_state and grid_risk.

---

### 4. **ENERGY_*** (3 files: energy_core, energy_helpers, energy_state)

#### What it CONSUMES
**energy_core:**
- `sensor.inverter_pv_power`, `sensor.inverter_load_power`, `sensor.inverter_battery_power`, `sensor.inverter_grid_power` ← From **power_core**
- `sensor.inverter_today_production`, `sensor.inverter_today_battery_charge`, `sensor.inverter_today_load_consumption`, etc. ← From **power_core**
- `sensor.solar_to_battery_energy`, `sensor.grid_to_battery_energy` ← From itself (utility meter inputs)
- `sensor.solar_to_battery_energy_today`, `sensor.solar_used_by_house_today` ← From itself

**energy_state:**
- `sensor.solar_export_potential` ← From **solar_clipping**
- `sensor.battery_minutes_remaining_safe` ← From **battery_runtime**
- `sensor.solcast_pv_forecast_forecast_remaining_today`, `sensor.solcast_pv_forecast_forecast_next_hour` ← From **solcast** (external)
- `sensor.inverter_battery_soc`, `sensor.inverter_pv_power`, `sensor.inverter_load_power`, `sensor.inverter_grid_power` ← From **power_core**
- `number.inverter_1_battery_shutdown_soc` ← Hardware
- Various inverter power/battery sensors

**energy_helpers:**
- `sensor.inverter_today_load_consumption` ← From **power_core** (utility meter source)
- `group.known_power_loads`, `group.flexible_power_loads`, `group.critical_power_loads` ← From **power_templates**

#### What it PRODUCES
- `solar_to_house_power`, `solar_to_battery_power`, `solar_to_grid_power`
- `battery_to_house_power`, `grid_to_house_power`, `grid_to_battery_power`
- `house_power_losses`, `house_power_losses_today`
- `energy_balance_error_today`
- `solar_used_by_house_today` ← **Consumed by solar_state**
- `grid_used_by_house_today`
- `battery_used_by_house_today`
- `energy_economy_score`
- `energy_orchestrator_state` ← **Consumed by automations**
- `house_energy_resilience_hours`, `house_energy_resilience_status`
- `energy_loss_percent_today`, `energy_loss_state`
- `battery_night_survival`
- `solar_sufficiency_tomorrow`
- Utility meters: `house_energy_daily`, `house_energy_monthly`, `house_grid_energy_daily`, etc.
- `known_energy_today`, `flexible_energy_today`, `critical_energy_today` ← Depend on **power_templates** groups
- (40+ total entities)

#### Internal Dependencies
```
energy_*
  ↓ CONSUMES
  ├─ power_core (inverter sensors, today energy totals)
  ├─ battery_runtime (battery_minutes_remaining_safe)
  ├─ solar_clipping (solar_export_potential)
  ├─ power_templates (groups: known/flexible/critical_power_loads)
  └─ solcast (forecast)
  
  ↓ PRODUCES → CONSUMED_BY
  ├─ solar_used_by_house_today → solar_state
  ├─ energy_economy_score → (dashboard/UI)
  ├─ energy_orchestrator_state → (automations)
  └─ house_energy_resilience_* → (dashboard/UI)
```

**Can be split?** Partially. `energy_core` could stay; `energy_state` creates circularity (consumes solar_export_potential to compute solar sufficiency).

---

### 5. **POWER_*** (6 files: power_core, power_helpers, power_state, power_templates, power_strategy, power_contract)

#### power_core CONSUMES
- Direct hardware only: `sensor.inverter_1_*`, `sensor.inverter_2_*` (from solarman custom component)

#### power_core PRODUCES
- `house_grid_power` ← **Core aggregation** (used everywhere)
- `house_solar_power` ← **Core aggregation** (used everywhere)
- `house_load_power` ← **Core aggregation** (used everywhere)
- `house_battery_power` ← **Core aggregation**
- `inverter_battery_power` ← **Core signal**
- `inverter_battery_soc` ← **Core signal**
- `inverter_pv_power`, `inverter_grid_power`, `inverter_load_power`
- `inverter_today_production`, `inverter_today_battery_charge`, `inverter_today_load_consumption`, etc.
- `inverter_total_battery_charge`, `inverter_total_battery_discharge`, etc.
- (50+ aggregated entities)

#### power_helpers CONSUMES
- `sensor.inverter_1_battery_current`, `sensor.inverter_2_battery_current` ← From **power_core** inputs (hardware)
- Groups and templates derived from inverter 1/2

#### power_helpers PRODUCES
- Groups: `inverter_battery_soc_group`, `inverter_grid_frequency`, `inverter_battery_temperature`, etc.
- Templates: `inverter_battery_current`, `inverter_battery_voltage`, `inverter_pv1_power`, `inverter_pv2_power`, etc.

#### power_state CONSUMES
- `sensor.inverter_battery_soc` ← From **power_core**
- `group.inverter_grid` ← From **power_helpers**
- `sensor.solar_power_available_conservative` ← From **solar_state**
- `sensor.battery_runtime_confidence` ← From **battery_runtime**
- `sensor.solar_system_status` ← From **solar_state**
- `sensor.solar_generation_confidence` ← From **solar_state**
- `sensor.solar_stability` ← From **solar_state**
- `sensor.battery_minutes_remaining_safe` ← From **battery_runtime**

#### power_state PRODUCES
- `house_power_health` ← **Used by automations**
- `power_state` ← **Decision sensor**
- `power_flow_state`
- `house_power_source`
- `grid_runtime_remaining` ← Uses **prepaid data** (units_left_safe)
- `inverter_health`
- (15+ state entities)

#### power_templates CONSUMES
- `sensor.inverter_load_power` ← From **power_core**
- `group.known_power_loads`, `group.flexible_power_loads`, `group.critical_power_loads` ← Defined in itself

#### power_templates PRODUCES
- `power_battery_soc` ← Abstraction of `inverter_battery_soc`
- `house_power` ← Direct wrap of `inverter_load_power`
- `house_kitchen_power`, `house_living_areas_power`, `house_laundry_power`, `house_outdoor_power` ← Area aggregations
- `active_high_loads_power`, `active_low_loads_power`
- `known_load_power`, `flexible_load_power`, `critical_load_power` ← **Used by energy_helpers**
- `unknown_load_power`

#### power_strategy CONSUMES
- `is_state('group.inverter_grid','on')` ← From **power_helpers**
- `sensor.inverter_pv_power`, `sensor.inverter_load_power`, `sensor.inverter_battery_soc` ← From **power_core**
- `sensor.solar_export_potential` ← From **solar_clipping**

#### power_strategy PRODUCES
- `power_strategy` ← **Decision engine** (solar_surplus, solar_ok, battery_critical, normal, etc.)
- `power_strategy_status` (human-readable)
- `power_strategy_severity`

#### power_contract
- **Documentation file only** — NO entities, NO unique_ids
- Contains design commentary on state types and inheritance

#### Internal Dependencies
```
power_core
  ↓ CONSUMES
  └─ hardware only (inverter_1_*, inverter_2_*)
  
  ↓ CONSUMED_BY
  └─ (everything else)

power_helpers
  ↓ CONSUMES
  └─ power_core
  
  ↓ CONSUMED_BY
  ├─ power_state (group.inverter_grid)
  ├─ energy_helpers (group.known/flexible/critical_power_loads)
  └─ power_strategy (group.inverter_grid)

power_state
  ↓ CONSUMES
  ├─ power_core (inverter_battery_soc, grid status)
  ├─ battery_runtime (runtime_confidence, minutes_remaining_safe)
  └─ solar_state (solar_power_available, system_status, confidence, stability)
  
  ↓ PRODUCES → CONSUMED_BY
  └─ house_power_health, power_state, power_flow_state → (automations, cards)

power_templates
  ↓ CONSUMES
  ├─ power_core (inverter_load_power)
  └─ itself (power load groups)
  
  ↓ PRODUCES → CONSUMED_BY
  ├─ known_load_power, etc. → energy_helpers
  └─ house_kitchen_power, etc. → (UI/dashboards)

power_strategy
  ↓ CONSUMES
  ├─ power_core (inverter sensors)
  ├─ solar_clipping (solar_export_potential)
  └─ power_helpers (group.inverter_grid)
  
  ↓ PRODUCES → CONSUMED_BY
  └─ power_strategy, power_strategy_severity → (automations, conditions)

power_contract
  ↓ (Documentation only)
```

**Can be split?** NO — power_core is the linchpin. Everything else is layered abstraction on top.

---

### 6. **GRID_*** (2 files: grid_state, grid_risk)

#### grid_state CONSUMES
- `is_state('group.inverter_grid','on')` ← From **power_helpers**
- `sensor.grid_risk_severity` ← From **grid_risk** (internal)

#### grid_state PRODUCES
- `grid_state_health` ← State indicator (stable, risk, unstable, offline)

#### grid_risk CONSUMES
- `sensor.battery_minutes_remaining_safe` ← From **battery_runtime**
- `sensor.battery_runtime_severity` ← From **battery_runtime**
- `sensor.battery_runtime_confidence` ← From **battery_runtime**
- `sensor.inverter_1_battery` ← Raw hardware (inverter_1)
- `sensor.inverter_pv_power`, `sensor.inverter_power` ← From **power_core**
- `is_state('binary_sensor.inverter_1_grid','on')` ← Raw hardware sensor
- `sensor.ss_soc_battery_time_left` ← From **battery_runtime**

#### grid_risk PRODUCES
- `grid_risk_level` ← severity indicator (safe, moderate, warning, critical)
- `grid_risk_status` ← Human-readable with ETA
- `grid_dependency_eta` ← Time estimate
- `grid_risk_severity` ← **Used by grid_state**

#### Internal Dependencies
```
grid_risk
  ↓ CONSUMES
  ├─ battery_runtime (battery_minutes_remaining_safe, battery_runtime_severity, ss_soc_battery_time_left)
  ├─ power_core (inverter_1_battery, inverter_pv_power, inverter_power)
  └─ hardware (inverter_1_grid binary sensor)

grid_state
  ↓ CONSUMES
  ├─ grid_risk (grid_risk_severity)
  └─ power_helpers (group.inverter_grid)
```

**Can be split?** Partially. `grid_risk` is essential (battery safety). `grid_state` is just a wrapper.

---

### 7. **LOAD_SHEDDING** (1 file: load_shedding.yaml)

#### What it CONSUMES
- `sensor.load_shedding_stage_eskom` ← From **load_shedding custom component** (external, Eskom schedule)
- `sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark` ← From **load_shedding custom component** (external)
- State attributes like `state_attr('sensor.load_shedding_area_*', 'starts_in')`

#### What it PRODUCES
- `load_shedding_status_card` ← Display template
- `load_shedding_active` ← Binary sensor (true/false)
- `load_shedding_severity` ← Severity state (error/ok)
- `load_hedding_inutes_emaining` ← Numeric countdown (note: typo in unique_id)

#### Internal Dependencies
```
load_shedding
  ↓ CONSUMES
  └─ load_shedding custom component (external integration ONLY)
  
  ↓ PRODUCES → CONSUMED_BY
  ├─ load_shedding_active → (solar_forecast, alerts, automations)
  └─ load_shedding_* → (UI, dashboards)

  ⚠️ NO CIRCULAR DEPENDENCIES
  ⚠️ NO INTERNAL POWER PACKAGE DEPENDENCIES
```

**Can be split?** ✅ **YES — COMPLETELY INDEPENDENT** — zero internal dependencies.

---

## Dependency Matrix (Consolidated)

| Subsystem | CONSUMES | PRODUCES | DEPENDS_ON |
|-----------|----------|----------|-----------|
| **prepaid_*** | grid_energy_import_total, solar_production_monthly, solcast forecast | prepaid_units_*, prepaid_burn_*, prepaid_strategy | energy, solar, solcast |
| **solar_*** | house_solar_power, solcast forecast, inverter sensors, solar_used_by_house_today, load_shedding data | solar_export_potential, solar_system_status, solar_generation_confidence, solar_stability, solar_ramp_rate, solar_clipping | power_core, energy_core, load_shedding, solcast |
| **battery_*** | inverter_battery_soc, inverter_battery_power, inverter_grid, hardware configs | battery_state_health, battery_runtime_severity, battery_minutes_remaining_safe | power_core |
| **energy_*** | inverter sensors (power/energy), solar_export_potential, battery_minutes_remaining_safe, power load groups, solcast forecast | solar_used_by_house_today, house_power_losses, energy_economy_score, house_energy_resilience_*, energy_orchestrator_state, known_energy_today, flexible_energy_today, critical_energy_today | power_core, battery_runtime, solar_clipping, power_templates, solcast |
| **power_core** | hardware only (inverter_1_*, inverter_2_*) | house_grid_power, house_solar_power, inverter_battery_soc, inverter_pv_power, inverter_today_production, (50+ aggregated entities) | inverter hardware (custom component solarman) |
| **power_helpers** | power_core, hardware | inverter_battery_current, inverter_battery_voltage, inverter_pv*_power, groups | power_core |
| **power_state** | power_core sensors, group.inverter_grid, solar_power_available_conservative, battery_runtime_confidence, solar_system_status, solar_generation_confidence, battery_minutes_remaining_safe | house_power_health, power_state, power_flow_state, house_power_source, grid_runtime_remaining | power_core, battery_runtime, solar_state |
| **power_templates** | inverter_load_power, known/flexible/critical_power_loads groups | power_battery_soc, house_power, house_kitchen_power, known_load_power, flexible_load_power, critical_load_power, unknown_load_power | power_core |
| **power_strategy** | group.inverter_grid, inverter_pv_power, inverter_load_power, inverter_battery_soc, solar_export_potential | power_strategy, power_strategy_status, power_strategy_severity | power_core, solar_clipping, power_helpers |
| **power_contract** | (documentation only) | (none — documentation) | (none) |
| **grid_state** | group.inverter_grid, grid_risk_severity | grid_state_health | power_helpers, grid_risk |
| **grid_risk** | battery_minutes_remaining_safe, battery_runtime_severity, battery_runtime_confidence, inverter_1_battery, inverter_pv_power, inverter_power | grid_risk_level, grid_risk_status, grid_dependency_eta, grid_risk_severity | battery_runtime, power_core |
| **load_shedding** | load_shedding_stage_eskom, load_shedding_area_* (external) | load_shedding_active, load_shedding_severity, load_shedding_* | load_shedding custom component (external ONLY) |

---

## Dependency Graph (Simplified)

```
┌─────────────────────────────────────────────────────────────────┐
│                    HARDWARE LAYER                               │
│           (solarman custom component reads)                     │
│  inverter_1_*, inverter_2_* from Sunsynk inverter registers    │
│             + inverter_1 BMS program settings                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                    ┌────▼────────────────────┐
                    │  power_core             │
                    │  (aggregation hub)      │
                    └────┬────────────────────┘
                         │
            ┌────────────┼────────────┬─────────────┐
            │            │            │             │
        ┌───▼──────┐  ┌──▼────────┐  │         ┌───▼──────┐
        │power_    │  │power_     │  │         │battery_  │
        │helpers   │  │state      │  │         │*         │
        │(groups)  │  │(health)   │  │         │(runtime) │
        └──────────┘  └──┬────────┘  │         └─────┬────┘
                         │            │               │
                    ┌────▼──────────┐ │         ┌─────▼─────┐
                    │power_strategy │ │         │grid_risk  │
                    │(decision)     │ │         │(warnings) │
                    └───────────────┘ │         └──────┬────┘
                                      │                │
            ┌─────────────┬───────────▼──────────┬────▼─────┐
            │             │                      │          │
        ┌───▼────┐   ┌────▼──────┐          ┌───▼──────┐  ┌──
▼────┐
│solar_*           │energy_*   │          │grid_state│  │prep\
aid_*
│ (export,         │(balances) │          │          │  │    \
  |
│  clipping,       │           │          └──────────┘  └──────┤
│  state)          │           │                               │
└───┬────┘   ┌─────▼──────┐    │                               │
    │        │power_      │    │                         ┌─────▼─────┐
    │        │templates   │    │                         │load_      │
    │        │(areas)     │    │                         │shedding   │
    │        └────────────┘    │                         │(external) │
    │                          │                         └───────────┘
    └──────────────────────────┴──────────────────────────────┘
              ALL FEED INTO: automations, scenes, UI
```

---

## Safe Split Candidates

### ✅ **CAN SPLIT OUT (Zero interdependencies)**

1. **load_shedding.yaml** 
   - Depends only on external `load_shedding` custom component
   - Consumed by: solar_forecast automations, other automations (external to power package)
   - **Status**: Candidate for external own package or `eskom/` package

### ⚠️ **PARTIAL CANDIDATES (Minimal internal deps)**

2. **power_contract.yaml**
   - Pure documentation, zero entities
   - Can be deleted or merged into comments
   - **Status**: Safe to remove

3. **grid_state.yaml**
   - Only depends on `grid_risk.yaml` + `power_helpers`
   - Could be merged with `grid_risk.yaml`
   - **Status**: Could combine: `grid_*.yaml` → single `grid_state.yaml`

### ❌ **CANNOT SPLIT (Circular or critical)**

- **power_core**: Foundation. Powers everything.
- **battery_runtime**: Provides critical `battery_minutes_remaining_safe` signal to power_state, grid_risk, energy_state.
- **power_state**: Depends on solar_state + battery_runtime; decision hub for automations.
- **solar_state**: Depends on power_core sensors; produces solar_export_potential consumed by power_strategy, energy_state.
- **energy_core/energy_state**: Depends on power_core + battery_runtime + solar_clipping. Produces solar_used_by_house_today consumed by solar_state.
- **prepaid_***: Depends on energy (grid_energy_import_total) + solar (solcast forecast).
- **power_strategy/power_templates**: Abstraction layers on power_core; no independent purpose.

---

## Recommendation Summary

### Current State
- **Highly coupled**: All subsystems except `load_shedding` have interdependencies
- **Power_core is linchpin**: Direct or transitive dependency from every other subsystem
- **No circular dependencies**: DAG is acyclic (good)
- **Clear layering**:
  1. Hardware → power_core (aggregation)
  2. power_core → power_state, battery_runtime, energy_core (analysis)
  3. Analysis → decision sensors (power_strategy, grid_risk)
  4. Decision sensors → automations

### Best Practices for Modification
1. **Before modifying power_core**: Audit all 15+ dependents
2. **Before modifying battery_runtime**: Check power_state, grid_risk, energy_state
3. **Before modifying solar_state**: Check power_state, energy_state, power_strategy
4. **Refactoring prepaid**: Carefully decouple from energy module (grid_energy_import_total dependency)

### If Splitting is Required
1. **Extract load_shedding first** (zero internal deps)
2. **Then grid_state + grid_risk** (can be merged or isolated)
3. **Never split**: power_core, battery_runtime, solar_state, energy_core, power_state
