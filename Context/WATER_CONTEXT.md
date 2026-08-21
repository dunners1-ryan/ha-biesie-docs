# HABiesie — Water Domain Context
> **Living document.** Update after every change to the water package.  
> Before modifying ANY water file, read `packages/water/a_water_lifecycle_contract.yaml` first.  
> Paste alongside `PROJECT_STATE.md` when working on water.
>
> **⚠️ Superseded by `docs/domains/WATER_CONTRACT.md` — use the contract for real work.**
> This file is a quick-reference summary, dated 2026-04-13, essentially untouched since
> while the water domain absorbed months of fixes (most recently a full drift sweep
> earlier the same day as this correction, 2026-08-21). Package Files listed 4
> nonexistent files and misplaced one; the Safety Abort Logic table described dry-run
> protection as active (disabled) and a max-runtime helper that no longer exists
> (redesigned into a rate-based guard the same day); all 3 Known Problems were already
> fixed. All corrected 2026-08-21.

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

**⚠️ Corrected 2026-08-21 — this list named 4 files that don't exist
(`water_state.yaml`, `water_automations.yaml`, `water_core.yaml`, `water_debug.yaml`) and
misplaced `water_notifications.yaml` (it actually lives in `packages/notifications/`, not
`packages/water/`). Replaced with the live 18-file inventory in `packages/water/`.**

```
packages/water/  (18 files)
  a_water_lifecycle_contract.yaml  # SPECIFICATION DOCUMENT (locked) — read before anything else
  water_helpers.yaml               # all input helpers and thresholds
  water_templates.yaml             # depth validation, level %, water_state, spike guard
  water_tank_refill_control.yaml   # main refill control logic (Safety/Critical/Emergency/etc.)
  water_protection_automations.yaml # borehole no-rise/degraded-rate/spike-rejection guards
  water_safety.yaml                # hard stops (max depth, battery hard stop)
  water_refill_cycle.yaml          # cycle state + summary sensors
  water_refill_capture.yaml        # start/end capture automations
  water_state_extensions.yaml      # borehole_control_status, refilling (derived) mirror
  water_consume_cycle.yaml, water_fault_logging.yaml, water_health.yaml,
  water_maintenance_automations.yaml, water_reporting.yaml, water_scripts.yaml,
  water_sensor.yaml, water_test_helpers.yaml

packages/notifications/
  water_notifications.yaml         # water event notifications (NOT in packages/water/)
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

**⚠️ Corrected 2026-08-21 against WATER_CONTRACT.md Section 11 (Safety System Audit,
re-verified live same day):**

| Protection | Trigger | Action |
|---|---|---|
| Max depth stop | depth ≥ 1.95m (hardcoded, not `water_target_depth_full`) | Stop pump — debounced 1min |
| ~~Dry run protection~~ | — | **❌ DISABLED**, not "in place" — deliberately replaced by no-rise protection (binary_sensor + automation both commented out) |
| No-rise protection | pump running + depth rate < 0.01 m/h for 15 min | Stop pump, log fault, **auto-retries after cooldown** (2026-08-18) |
| Battery hard stop | SOC drops below `water_battery_soc_hard_stop` (40%) | Stop pump |
| ~~Max runtime cutoff~~ | — | **Redesigned 2026-08-21** — `water_refill_max_runtime_minutes` deleted outright, replaced by `water_borehole_degraded_rise_rate_protection` (rate-based: stops if depth rate stays between 0.01-0.05 m/h for 60 min, not a flat wall-clock cutoff) |

---

## ⚠️ Known Problems

**⚠️ All 3 items below corrected 2026-08-21 — all confirmed done in WATER_CONTRACT.md's
Section 9 Implementation Checklist (Sprint 2, "Fix Trigger Integrity"), re-verified
live during this session's own WATER_CONTRACT.md pass earlier the same day.**

### 1. ✅ DONE — False Cycle Detection
`from: "off"` guards confirmed present on the relevant pump triggers
(`water_borehole_no_rise_protection`, `water_reconcile_cycle_state`, fixed 2026-04-15)
plus a stability window (`water_refill_visibility_guard`, `for: "00:00:10"`). Not
"partially implemented" — done.
- ~~**Fix needed:** Add `from: "off"` constraint to all pump triggers + 10s stability window~~
- ~~**Status:** Partially implemented, needs verification~~

### 2. ✅ DONE — Control Loop Fighting Safety
WATER_CONTRACT.md's own Sprint 2 checklist: "E3 verified: `binary_sensor.water_refill_allowed`
checked before pump start on critical+low paths. Safety/limited-critical paths
intentionally bypass (emergency scenarios)" — by design, not an oversight.
- ~~**Fix needed:** Pre-check `binary_sensor.water_refill_allowed` before any pump start~~
- ~~**Status:** `binary_sensor.water_refill_allowed` exists but not consistently checked first~~

### 3. ✅ DONE (this specific cause) — Notification Spam
The control-loop-retry spam this item describes is resolved along with #2. **Note:** a
*different* notification-spam risk (Tuya spike-rejection notifications firing on every
glitch, not control-loop retries) was found and fixed separately today — see
WATER_CONTRACT.md Recommendation 5 (new cooldown gate, `water_protection_automations.yaml`).
- ~~**Fix:** Pre-check pattern above eliminates the artificial cycles~~
- ~~**Status:** Linked to issue #2~~

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
