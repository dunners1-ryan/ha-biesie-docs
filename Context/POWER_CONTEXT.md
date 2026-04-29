# HABiesie — Power & Energy Domain Context
> **Living document.** Update after every change to the power package.  
> Paste alongside `PROJECT_STATE.md` when working on power/energy/prepaid.

---

## 🎯 Intended Design

A two-layer measurement model that separates operational data (inverter) from financial truth (prepaid meter), with a decision/strategy layer on top:

```
Inverter (real-time) ──┐
                        ├──→ Aligned Sensors ──→ Strategy Layer ──→ Buy Decision
Prepaid Meter (truth) ──┘
```

---

## 📁 Package Files

```
packages/power/
  power_helpers.yaml         # input_numbers for thresholds, offsets, settings
  power_templates.yaml       # all derived power/energy sensors
  power_state.yaml           # health states, risk levels, strategy sensors
  power_automations.yaml     # alerts, load management, charging programs
  power_core.yaml            # solarman integration config, utility meters
  power_prepaid.yaml         # prepaid tracking, cost model, buy score
  power_loads.yaml           # load group definitions and tracking
  power_solar.yaml           # solar-specific sensors and forecasting
  power_battery.yaml         # battery runtime, health, and state sensors
  power_grid.yaml            # grid state, import tracking, risk sensors
```

---

## ⚡ Hardware Architecture

### Dual Inverter Setup
| Inverter | Role | Measures |
|---|---|---|
| Inverter 1 (Master) | Grid interface | Grid power, system losses, BMS |
| Inverter 2 (Slave) | Generation | PV power, load, battery |

Both aggregated in `power_core.yaml` into unified sensors.

### Solar
- 4x MPPT strings (2 per inverter)
- Max PV: ~10.8kW
- Battery: ~31.8kWh
- Forecast: Solcast integration

### Grid
- Prepaid meter (Eskom)
- Load shedding integration (SePush API)

---

## 🔴 Critical Sensors (DO NOT RENAME)

### Real-time Power
```
sensor.inverter_load_power          # current house load (W)
sensor.inverter_grid_power          # grid import/export (W, negative = export)
sensor.inverter_pv_power            # total solar generation (W)
sensor.inverter_battery_power       # battery charge/discharge (W)
sensor.inverter_battery_soc         # battery state of charge (%)
sensor.inverter_battery_voltage     # battery voltage (V)
sensor.inverter_battery_current     # battery current (A)
sensor.house_solar_power            # alias/aggregate for solar
group.inverter_grid                 # on = grid connected, off = offline
group.inverter_battery_state        # battery group state
```

### Inverter Sub-sensors
```
sensor.inverter_1_pv1_power         # string 1 power
sensor.inverter_1_pv2_power         # string 2 power
sensor.inverter_2_pv1_power         # string 3 power
sensor.inverter_2_pv2_power         # string 4 power
sensor.inverter_1_battery_state     # charging/discharging/idle
sensor.inverter_1_battery_temperature
sensor.inverter_ac_temperature
sensor.inverter_dc_temperature
```

### Daily Energy
```
sensor.inverter_today_production        # solar kWh today
sensor.inverter_today_load_consumption  # load kWh today
sensor.inverter_today_battery_charge    # battery charged kWh today
sensor.inverter_today_battery_discharge # battery discharged kWh today
sensor.grid_energy_import_today         # grid imported kWh today
sensor.inverter_today_energy_export     # exported kWh today
```

### Prepaid / Financial
```
sensor.prepaid_units_left_safe          # conservative balance (use this for display)
sensor.prepaid_units_left_authoritative # exact balance from meter reading
sensor.prepaid_units_used_authoritative # units consumed since last top-up
sensor.prepaid_grid_meter_drift         # inverter vs meter drift (kWh)
sensor.prepaid_drift_percentage         # drift as percentage
sensor.prepaid_estimated_days_remaining # days at current burn rate
sensor.prepaid_adaptive_burn_rate       # rolling burn rate (kWh/day)
sensor.prepaid_energy_cost_per_kwh      # current purchase cost
sensor.prepaid_true_cost_per_kwh        # lifetime blended cost
sensor.prepaid_true_cost_per_kwh_monthly # monthly blended cost
sensor.prepaid_spend_this_month         # R spent this month
sensor.prepaid_buy_score                # 0-100 score (>70 = buy now)
sensor.prepaid_buy_decision             # human readable buy recommendation
sensor.prepaid_topup_strategy           # strategic timing recommendation
sensor.prepaid_optimal_topup_size       # suggested units to buy
```

### Strategy / Decision Layer
```
sensor.power_state                  # grid/solar/battery_safe/battery_risk/battery_critical
sensor.house_power_health           # grid_stable/solar_powered/battery_resilient/battery_risk/critical
sensor.house_power_source           # dominant source: grid/solar/battery
sensor.power_flow_state             # solar_only/solar_and_battery/grid_only/battery_only/offgrid_critical
sensor.power_strategy               # current recommended strategy
sensor.energy_orchestrator_state    # run_heavy_loads/run_medium_loads/reduce_load/conserve_energy
sensor.solar_opportunity_level      # high/medium/low/none
sensor.solar_generation_confidence  # high/medium/low
sensor.solar_stability              # rapidly_improving/improving/stable/declining/collapsing
sensor.battery_state_health         # strong/healthy/low/critical
sensor.grid_state_health            # stable/risk/unstable/offline
```

### Input Controls
```
input_number.prepaid_alignment_offset         # manual reconciliation offset
input_number.prepaid_meter_lifetime_import    # meter reading (kWh, entered manually)
input_number.prepaid_units                    # current balance on meter display
input_number.prepaid_last_topup_units         # units in last purchase
input_number.prepaid_last_topup_cost          # cost of last purchase (R)
input_number.prepaid_last_topup_fixed_deducted # fixed charge deducted
input_number.prepaid_last_topup_discount      # discount received
input_number.prepaid_total_spent              # running total spend this month
input_number.prepaid_daily_burn_average       # expected daily usage (kWh)
input_number.prepaid_fixed_monthly_cost       # fixed monthly network charge (R)
input_number.grid_tariff_nominal              # Eskom tariff R/kWh for comparison
input_number.solar_export_tariff              # notional export value R/kWh
input_number.grid_import_usage_low_warning_trigger    # daily import low threshold
input_number.grid_import_usage_medium_warning_trigger
input_number.grid_import_usage_high_warning_trigger
input_number.inverter_battery_soc_warning_trigger
```

---

## 📐 Measurement Model

### Source of Truth Hierarchy
```
1. input_number.prepaid_meter_lifetime_import  ← AUTHORITATIVE (manual entry)
2. sensor.grid_energy_import_total             ← OPERATIONAL (inverter, high frequency)
3. input_number.prepaid_alignment_offset       ← RECONCILIATION (manual correction)

Authoritative usage = inverter_total + alignment_offset - prepaid_baseline
```

### Reliability Classification
| Tier | Sensors | Trust |
|---|---|---|
| ✅ Reliable | `grid_energy_import_total`, `prepaid_meter_lifetime_import`, `prepaid_units_used_authoritative`, `prepaid_drift_percentage` | High |
| 🟡 Semi-reliable | `prepaid_units_left_safe`, `prepaid_estimated_days_remaining`, `prepaid_adaptive_burn_rate` | Medium — depends on thresholds |
| ✅ Fixed (was broken) | Monthly spend (now usage×rate, not top-up based), drift (now %, not absolute) | High |

---

## 💡 Load Management

### Load Groups
```
group.real_power_loads      # all monitored loads
group.flexible_power_loads  # can be shed during battery risk
group.critical_power_loads  # never shed
group.known_power_loads     # loads with power sensors
```

Managed by `python_scripts/sync_load_group.py` — syncs entities by HA labels.

### Key Load Entities
```
sensor.geyser_heat_pump_power       switch.geyser_heat_pump_switch_1
sensor.pool_pump_power              switch.pool_pump_switch_1
sensor.water_pressure_pump_power    switch.water_pressure_pump
sensor.borehole_pump_power          switch.borehole_pump
sensor.philips_airfryer_plug_power  switch.philips_airfryer_plug
sensor.dishwasher_samsung_plug_power switch.dishwasher_samsung_plug
```

---

## ✅ What Works Well

- Dual inverter measurement and aggregation
- Prepaid financial model (clean, usage-based, no double-counting)
- Solar forecast integration (Solcast)
- Buy score and decision logic
- Energy flow visualization (sunsynk + power-flow-card-plus)
- Load visibility scoring
- Grid import tracking with thresholds
- Battery runtime estimation

---

## 🔴 Prepaid Meter Drift Management

### Architecture
The prepaid system measures two separate things:
1. **Physical meter** (Empire CIU EV-KP) — manually read, AUTHORITATIVE
2. **Inverter grid import** (via Solarman) — continuous, OPERATIONAL

These naturally diverge over time due to:
- **Measurement points**: Inverter (AC side) vs meter (after losses)  
- **Update frequency**: Meter (daily/manual) vs inverter (real-time)
- **Resets/reboots**: Inverter may drift during power loss
- **Measurement type differences**: Grid losses, power factor, harmonics

### Tolerance Thresholds
| Drift % | Status | Action |
|---------|--------|--------|
| < 1% | Aligned | ✅ Normal operation |
| 1-2% | Monitor | 🟡 Weekly verification recommended |
| 2-5% | Caution | 🟠 Schedule reconciliation soon |
| > 5% | Alert | 🔴 Reconciliation required immediately |

**Monthly expectation**: ±1-3% drift is normal for systems like this.

### Drift Monitoring System
Three automatic sensors track drift health:
- `sensor.prepaid_grid_meter_drift` — absolute kWh difference
- `sensor.prepaid_drift_percentage` — normalized percentage  
- `sensor.prepaid_drift_status` — reconciliation_required / monitor_closely / aligned
- `sensor.prepaid_days_since_verification` — days since last manual check

### Automatic Processes
- **Weekly reminder** (every Sunday 19:00) — prompts manual meter verification
- **Auto-reconciliation** (when drift > 5% AND usage < 1 kWh) — automatically realigns offset
- **Alert** (when drift > 2%) — notifies to check meter readings

### Manual Reconciliation Workflow
When you update the physical meter reading:
1. Read the meter **1.8.0** code (lifetime total import)
2. Enter in HA field: "Enter Meter Lifetime (1.8.0)"
3. Click **"Realign Prepaid Offset"** button
4. System auto-calculates: `new_offset = meter_reading - grid_import_total`
5. Verify remaining balance matches meter **C.51.0** on display

The large offset (typically 100k+) is **normal** — it represents usage before the inverter started tracking.

---

## 🎯 Next Steps (Agreed Priority)

1. **Buy Score v2** — add confidence weighting, net position, solar trend (not just today)
2. **Confidence layer** — replace binary safe/authoritative with 0-100% confidence score
3. **Historical learning** — track burn rate accuracy vs actuals over time

---

## 🚫 Deferred

- Export tariff optimization (no export capability currently)
- Time-of-use arbitrage automation (manual programs set on inverter)
- Predictive load scheduling beyond current orchestrator

---

## 🔗 Dependencies

- **Water system:** `binary_sensor.water_refill_allowed` reads battery SOC + grid state
- **Lighting:** Energy saving mode reads `sensor.power_state`
- **Alerts:** `sensor.house_power_health` feeds power alert context
- **Presence:** Orchestrator considers occupancy for load decisions

---

*Last updated: <!-- DATE -->*  
*Updated by: <!-- CHANGE SUMMARY -->*
