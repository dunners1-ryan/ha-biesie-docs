# HABiesie — Water Domain Context
> **Living document.** Update after every change to the water package.  
> Before modifying ANY water file, read `packages/water/a_water_lifecycle_contract.yaml` first.  
> Paste alongside `PROJECT_STATE.md` when working on water.

---

## 🎯 Intended Design

A deterministic refill lifecycle system that observes physical reality (depth sensor), not device state (switch status):

```
Truth = water level movement, NOT switch state
```

### Lifecycle States
```
Idle ──→ Running ──→ Completed
           ↓              
        Aborted (safety)
```

### Core Contract Rules
1. **Start = physical event** — pump ON and depth increasing
2. **Active = managed cycle** — `input_boolean.water_refill_cycle_active` is ON
3. **End = physical completion** — pump OFF after valid run (not a block/abort)
4. **Abort = safety intervention** — `input_boolean.water_refill_aborted_due_to_safety` is ON
5. **Capture is observational only** — it records what happened, does not make control decisions
6. **Safety is independent and absolute** — safety aborts cannot be blocked by any other logic
7. **Control checks first** — pump only starts if `binary_sensor.water_refill_allowed` is ON

---

## 📁 Package Files

```
packages/water/
  a_water_lifecycle_contract.yaml  # SPECIFICATION DOCUMENT — read before anything else
  water_helpers.yaml               # all input helpers and thresholds
  water_templates.yaml             # depth validation, level %, rate sensors
  water_state.yaml                 # water_state aggregation, refill permission
  water_automations.yaml           # lifecycle capture, safety aborts, scheduling
  water_core.yaml                  # Tuya integration config for sensors/pumps
  water_notifications.yaml         # water event notifications
  water_debug.yaml                 # diagnostic sensors for debugging
```

---

## 🔴 Critical Entities (DO NOT RENAME)

### Core State
```
sensor.water_state                          # ok/low/critical/safety
sensor.water_tank_level                     # 0-100% (calculated from depth)
sensor.water_tank_depth_validated           # depth in metres (spike-filtered)
sensor.water_tank_level_sensor_depth        # raw depth from Tuya sensor
sensor.water_tank_level_sensor_liquid_level # raw % from Tuya sensor
sensor.water_tank_depth_rate                # m/h rate of change
sensor.water_tank_consumption_rate          # m/h consumption when not refilling
sensor.water_refill_cycle_avg_flow_rate     # avg fill rate during active cycle
```

### Lifecycle Flags
```
input_boolean.water_refill_cycle_active              # true during valid refill
input_boolean.water_refill_aborted_due_to_safety     # true after safety abort
input_boolean.water_tank_refill_enabled              # master enable/disable
binary_sensor.water_refill_allowed                   # composite permission gate
binary_sensor.water_tank_refilling                   # pump on + depth rising
binary_sensor.water_solar_window_active              # in solar refill window
binary_sensor.water_tank_depth_sensor_stable         # sensor not spiking
```

### Pumps
```
switch.borehole_pump          # primary refill pump
switch.water_pressure_pump    # house pressure pump (separate system)
sensor.borehole_pump_power    # W — used for dry-run detection
```

### Safety Counters
```
counter.water_borehole_faults_today
counter.water_borehole_faults_this_week
input_number.water_emergency_runs_today
input_datetime.water_last_emergency_refill
input_boolean.water_emergency_acknowledged
input_text.borehole_last_fault                # description of last fault
```

### Thresholds (all in metres unless noted)
```
input_number.water_target_depth_full          # ~1.95m — stop refill at this depth
input_number.water_target_depth_partial       # partial fill target
input_number.water_target_depth_normal        # normal operating target
input_number.water_depth_low                  # trigger low alert
input_number.water_depth_minimum_safety       # dry-run protection floor
input_number.water_depth_critical             # trigger critical alert

input_number.water_battery_soc_sufficient     # min SOC to allow refill
input_number.water_battery_soc_hard_stop      # SOC that aborts active refill
input_number.water_refill_max_runtime_minutes # max pump runtime per cycle

input_datetime.water_refill_solar_start       # solar window start time
input_datetime.water_refill_solar_stop        # solar window end time
```

---

## ⚙️ Safety Abort Logic (IN PLACE — DO NOT REMOVE)

| Protection | Trigger | Action |
|---|---|---|
| Max depth stop | depth ≥ `water_target_depth_full` | Stop pump |
| Dry run protection | pump ON + power < threshold | Stop pump, log fault |
| No-rise protection | pump running + depth not increasing after timeout | Stop pump, log fault |
| Battery hard stop | SOC drops below `water_battery_soc_hard_stop` | Stop pump |
| Max runtime cutoff | pump on > `water_refill_max_runtime_minutes` | Stop pump, log fault |

---

## ⚠️ Known Problems

### 1. False Cycle Detection (ROOT ISSUE)
- **Cause:** Tuya sensor instability causes `unavailable → on` transitions
- **Symptom:** System logs false refill start/end cycles
- **Fix needed:** Add `from: "off"` constraint to all pump triggers + 10s stability window
- **Status:** Partially implemented, needs verification

### 2. Control Loop Fighting Safety
- **Old pattern (broken):** Control → start pump → Safety → stop pump → Capture → logs cycle
- **Correct pattern:** Check permission BEFORE starting → only start if allowed
- **Fix needed:** Pre-check `binary_sensor.water_refill_allowed` before any pump start
- **Status:** `binary_sensor.water_refill_allowed` exists but not consistently checked first

### 3. Notification Spam
- **Cause:** 5-minute control loop retrying + pump being blocked each time
- **Symptom:** Repeated START → END → START → END notifications
- **Fix:** Pre-check pattern above eliminates the artificial cycles
- **Status:** Linked to issue #2

---

## ✅ What Works Well

- Depth sensor spike filtering (`water_tank_depth_validated`)
- Safety abort logic (all 5 protections in place)
- Solar window scheduling
- Fault counter tracking (daily + weekly)
- Manual run scripts (5/10/15/30/60 min options)
- Water state aggregation (ok/low/critical/safety)
- Dashboard visual (shimmer on refill, color gradient by level)

---

## 📐 Refill Permission Logic

`binary_sensor.water_refill_allowed` = ON when ALL of:
- `input_boolean.water_tank_refill_enabled` is ON
- `sensor.inverter_battery_soc` ≥ `input_number.water_battery_soc_sufficient`
- `group.inverter_grid` is ON **OR** SOC is above extended threshold
- Not currently in safety abort state
- Within solar window (preferred) OR battery sufficiently high

---

## 🎯 Next Steps (Agreed Priority)

1. **Verify `from: "off"` triggers** are in place on all pump automations
2. **Add 10s stability window** to pump triggers
3. **Add pre-check** for `water_refill_allowed` at start of control automation
4. **Formalize state machine** — Idle/Running/Completed/Aborted as explicit sensor states
5. **Reduce notification flood** — one notification per real cycle start/end only

---

## 🔗 Dependencies

- **Power system:** `sensor.inverter_battery_soc`, `group.inverter_grid` → feeds `water_refill_allowed`
- **Alerts system:** `script.notify_water_event`
- **Context system:** `input_boolean.notifications_quiet_hours`

---

*Last updated: <!-- DATE -->*  
*Updated by: <!-- CHANGE SUMMARY -->*
