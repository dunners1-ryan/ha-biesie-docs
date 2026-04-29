# HABiesie — Coding Standards & Conventions
> **This is a living contract.** Update when conventions change. Never write code that violates these rules.  
> All AI assistants working on this project must read and follow this document.

---

## 🔑 Core Principle

**Every piece of code produced for this project must be:**
1. Correctly placed in the right package file
2. Commented with a standard header block
3. Commented inline for all non-obvious logic
4. Defensive against `unknown` and `unavailable` states
5. Consistent with established naming conventions

---

## 📁 File Placement Rules

### Rule 1 — Always use the correct package file

| Content type | Goes in |
|---|---|
| `input_boolean`, `input_number`, `counter`, `timer`, `input_datetime`, `input_text` | `packages/<domain>/<domain>_helpers.yaml` |
| Template sensors (`sensor:` with `platform: template`) | `packages/<domain>/<domain>_templates.yaml` |
| State tracking binary sensors and aggregation sensors | `packages/<domain>/<domain>_state.yaml` |
| `automation:` blocks | `packages/<domain>/<domain>_automations.yaml` |
| Integration config (`camera:`, `notify:`, etc.) | `packages/<domain>/<domain>_core.yaml` |
| Alert definitions (`alert:`) | `packages/alerts/<subsystem>_alerts.yaml` |
| Notification scripts | `packages/notifications/<subsystem>_notifications.yaml` |

### Rule 2 — Never add to `automations.yaml`

This is the legacy UI-managed file. All new automations go in the appropriate package file.

### Rule 3 — Never create a new package directory without documenting it

If a new domain package is needed, add it to `PROJECT_STATE.md` and this file.

### Rule 4 — Always check the entity registry before creating any new entity

**Before creating any `input_boolean`, `utility_meter`, `sensor`, or helper in YAML:**

1. Search `.storage/core.entity_registry` for the intended `entity_id`
2. Check Developer Tools → States in the HA UI for similar entity names
3. If the entity exists as a UI-created entry (`platform: utility_meter`, `platform: input_boolean.corrupt`, config_entry_id present), DO NOT recreate it in YAML

HA will not error on duplicates — it silently appends `_2`, `_3` etc to the new entity, leaving both active and causing incorrect sensor reads.

**High-risk cases** (always check these before writing YAML):
- `utility_meter:` entities — frequently created via UI Energy dashboard
- `input_boolean:` helpers — often created via UI Helpers page
- Any sensor named `*_today`, `*_daily`, `*_monthly` — common utility meter targets

```bash
# Quick check before writing YAML:
grep -i "entity_id_to_check" /root/config/.storage/core.entity_registry
```

**Confirmed UI-managed entities that must NOT be recreated in YAML:**
- `sensor.solar_to_battery_energy_today` — utility_meter, created 2026-03-20
- `sensor.grid_to_battery_energy_today` — utility_meter, created 2026-03-20

---

## 🔄 Reload vs Restart Rules

### RELOAD ONLY — use Developer Tools → YAML

| What changed | Reload action |
|---|---|
| Template sensors / binary sensors | Reload Template Entities |
| Automations | Reload Automations |
| Scripts | Reload Scripts |
| Helpers (input_boolean, input_number, etc.) | Reload Helpers |
| Groups | Reload Groups |

### ⚠️ FULL RESTART REQUIRED

| What changed | Why |
|---|---|
| `alert:` entities | Cannot be reloaded — always requires restart |
| `configuration.yaml` | Core config, notify groups, recorder excludes |
| New integration or custom component | Integration init requires restart |
| New packages directory | `!include_dir_named` requires restart |
| `customize.yaml` | Requires restart |
| `.storage/lovelace` changes | Browser hard refresh only (`Cmd+Shift+R`) — not HA restart |

### Rules for alert changes
- **Batch all alert changes into one session** to minimise restarts
- Always note in commit message: `⚠️ requires HA restart`
- When a prompt touches `alert:` entities, state this explicitly at the top

### Standard closing block — use in all prompts
```
AFTER FIXES — RELOAD NOT RESTART (unless alerts changed)
1. Run: ha core check
2. Reload only what changed:
   - Template sensors changed  → Reload Template Entities
   - Automations changed       → Reload Automations
   - Scripts changed           → Reload Scripts
   - Helpers added/changed     → Reload Helpers
   - alert: entities changed   → ⚠️ Full HA restart required
   - configuration.yaml changed → ⚠️ Full HA restart required
   - .storage/lovelace changed → Browser hard refresh only (Cmd+Shift+R)
3. Verify in Developer Tools → States
4. Commit + update docs in same session
```

---

## 📝 YAML Comment Standards

### Automation Header Block (REQUIRED on every automation)

```yaml
automation:
  ##############################################################################
  # AUTOMATION: <Human readable name>
  # Package:    packages/<domain>/<filename>.yaml
  # Purpose:    <One sentence — what this automation does and why>
  # Trigger:    <What triggers it — entity, time, event>
  # Conditions: <Key conditions that gate execution>
  # Actions:    <What it does when it runs>
  # Notes:      <Known edge cases, dependencies, things to be careful about>
  # Last edit:  <Date> — <What changed>
  ##############################################################################
  - id: "unique_snake_case_id"
    alias: "Human Readable Name"
    ...
```

### Template Sensor Header Block (REQUIRED)

```yaml
##############################################################################
# TEMPLATE SENSOR: sensor.<entity_id>
# Package:         packages/<domain>/<domain>_templates.yaml
# Purpose:         <What this sensor computes and why it exists>
# Source data:     <What entities it reads from>
# Output:          <States it can produce>
# Availability:    <When it returns unavailable/unknown and why>
# Notes:           <Dependencies, quirks, update frequency>
##############################################################################
- name: "sensor_name"
  state: >
    ...
```

### Helper Header Block (REQUIRED on groups of helpers)

```yaml
##############################################################################
# HELPERS: <Subsystem name>
# Package:  packages/<domain>/<domain>_helpers.yaml
# Purpose:  <What these helpers support>
##############################################################################
```

### Inline Comment Rules

```yaml
# ✅ DO: Comment the WHY, not the WHAT
- condition: template
  value_template: >
    # Only allow refill if battery has enough charge to run the pump
    # without risking shutdown during a load shedding event
    {{ states('sensor.inverter_battery_soc') | float(0) >= 
       states('input_number.water_battery_soc_sufficient') | float(30) }}

# ❌ DON'T: Comment the obvious
- condition: template
  value_template: >
    # Check if battery SOC is greater than threshold
    {{ states('sensor.inverter_battery_soc') | float(0) >= 30 }}
```

---

## 🛡️ Template Safety Rules

### Rule 1 — Always use float(0) / int(0) defaults

```yaml
# ✅ CORRECT
{% set soc = states('sensor.inverter_battery_soc') | float(0) %}

# ❌ WRONG — crashes when sensor is unavailable
{% set soc = states('sensor.inverter_battery_soc') | float %}
```

### Rule 2 — Always guard availability before logic

```yaml
# ✅ CORRECT
{% if states('sensor.water_tank_depth_validated') not in ['unknown', 'unavailable'] %}
  {% set depth = states('sensor.water_tank_depth_validated') | float(0) %}
  ...
{% else %}
  unavailable
{% endif %}
```

### Rule 3 — Trigger integrity — always specify `from:` where it matters

```yaml
# ✅ CORRECT — won't fire on unavailable → on transitions
trigger:
  - platform: state
    entity_id: switch.borehole_pump
    from: "off"
    to: "on"

# ❌ WRONG — fires on unavailable → on (causes false cycles)
trigger:
  - platform: state
    entity_id: switch.borehole_pump
    to: "on"
```

### Rule 4 — Add stability windows for flapping sensors

```yaml
trigger:
  - platform: state
    entity_id: switch.borehole_pump
    from: "off"
    to: "on"
    for: "00:00:10"   # prevents reconnect glitches from triggering
```

### Rule 5 — Never use input_text to store structured data

`input_text` has a 255 character limit. For structured data use:
- `input_text` only for single short string values (last event description, etc.)
- Multiple discrete `input_text` entities for separate fields
- `pyscript` or `AppDaemon` for complex structured state
- Proper HA entities (counters, input_numbers, etc.) for numeric state

### Rule 6 — Never use Jinja2 block tags to conditionally emit YAML keys

HA's YAML parser processes `{% %}` tags **before** evaluating templates. A `{%` that appears at the structural YAML level (i.e. where a key or list item would appear) is seen as an illegal `%` token and causes HA to enter recovery mode.

```yaml
# ❌ WRONG — YAML parser sees {% as an illegal % token at key level
action: notify.send_message
data:
  message: "Hello"
  data:
    {% if sev == 'critical' %}
    inline_keyboard:
      - "Ack:/ack_alert"
    {% endif %}

# ✅ CORRECT — use choose: with separate branches; each branch has its own static key set
- choose:
    - conditions:
        - condition: template
          value_template: "{{ sev == 'critical' }}"
      sequence:
        - action: notify.send_message
          data:
            message: "Hello"
            data:
              inline_keyboard:
                - "Ack:/ack_alert"
  default:
    - action: notify.send_message
      data:
        message: "Hello"
        data:
          disable_notification: "{{ sev == 'information' }}"
```

Jinja2 `{{ }}` expressions are valid anywhere inside a YAML **scalar value** (strings, block scalars with `>` or `|`). The rule applies only to `{% if %}`, `{% for %}`, `{% set %}` etc. used to *conditionally emit YAML structure* (keys, list items, mappings).

---

## 🏷️ Naming Conventions

### Entity Naming Pattern

```
<domain>.<subsystem>_<description>_<qualifier>

# Examples:
sensor.water_tank_depth_validated
binary_sensor.water_refill_allowed
input_boolean.water_refill_cycle_active
switch.borehole_pump
sensor.prepaid_units_left_safe
```

### Automation ID Pattern

```
<subsystem>_<verb>_<object>_<qualifier>

# Examples:
id: "water_capture_refill_start"
id: "water_capture_refill_end"
id: "security_trigger_snapshot_capture"
id: "presence_resolver_arrival_detected"
id: "power_alert_battery_soc_low"
```

### Script ID Pattern

```
notify_<subsystem>_event
<subsystem>_manual_<action>

# Examples:
script.notify_water_event
script.notify_power_event
script.water_manual_borehole_run_5min
```

### Input Helper Naming

```
input_boolean.<subsystem>_<description>
input_number.<subsystem>_<parameter>_<unit_or_type>
input_datetime.<subsystem>_<event>_<qualifier>
input_text.<subsystem>_<data_type>

# Examples:
input_boolean.water_tank_refill_enabled
input_number.water_battery_soc_sufficient
input_datetime.water_refill_solar_start
input_text.borehole_last_fault
```

---

## 🔔 Notification Standards

### Always use the central notify scripts

```yaml
# ✅ CORRECT
service: script.notify_water_event
data:
  severity: "warning"
  title: "Water System"
  message: "Tank level below 30%"

# ❌ WRONG — bypass routing and quiet hours logic
service: notify.mobile_app_ryan_iphone
data:
  message: "Tank level below 30%"
```

### Severity levels

| Level | When to use |
|---|---|
| `info` | Informational, low urgency, suppressed in quiet hours |
| `warning` | Action recommended, always delivered |
| `critical` | Immediate action required, always delivered, escalated |

---

## 🎨 Dashboard Card Standards

### Template card defensive pattern

```yaml
# ✅ CORRECT — handles unavailable gracefully
primary: >
  {% set v = states('sensor.water_tank_level') | float(0) %}
  {% if v < 15 %} CRITICAL
  {% elif v < 30 %} LOW
  {% else %} Normal
  {% endif %}

# ❌ WRONG — will show 'None' or crash if unavailable
primary: >
  {{ states('sensor.water_tank_level') }}%
```

### CSS animation pattern (standard pulse)

```yaml
# Use consistent animation names across cards
@keyframes pulseCritical {
  0%   { box-shadow: 0 0 0 0 rgba(255,0,0,0.6); }
  70%  { box-shadow: 0 0 0 14px rgba(255,0,0,0); }
  100% { box-shadow: 0 0 0 0 rgba(255,0,0,0); }
}
@keyframes pulseWarning {
  0%   { box-shadow: 0 0 0 0 rgba(255,140,0,0.6); }
  70%  { box-shadow: 0 0 0 10px rgba(255,140,0,0); }
  100% { box-shadow: 0 0 0 0 rgba(255,140,0,0); }
}
```

---

## 🐍 Pyscript Rules

Pyscript restricts its execution environment in ways that differ from standard Python. All three of the following patterns **fail at runtime** and must never be used:

| Pattern | Error |
|---|---|
| `hass.data.get("entity_registry")` | `NameError: name 'hass.data' is not defined` — `hass` proxy blocks `.data` |
| `from homeassistant.helpers.entity_registry import ...` | `ModuleNotFoundError: import from homeassistant.helpers.entity_registry not allowed` |
| `task.executor(pyscript_fn)` | `TypeError: pyscript functions can't be called from task.executor` — only real Python functions work |

### The only working patterns in pyscript

```python
# ✅ Enumerate entity IDs by domain
entity_ids = state.names(domain="sensor")

# ✅ Read entity state
value = state.get("sensor.inverter_battery_soc")

# ✅ Call a HA service
service.call("group", "set", object_id="my_group", entities=entity_ids)

# ✅ Log
log.warning("something went wrong")
log.info("sync complete")
```

### What you cannot do from pyscript

- Access entity labels — entity registry is inaccessible; label-based filtering requires a YAML group definition instead
- Import from `homeassistant.*` — blocked by pyscript's ALLOWED_IMPORTS
- Access any attribute of `hass` directly — the `hass` proxy is restricted to `hass` itself as a symbol; no sub-attributes are exposed

### Pyscript service call patterns

```python
# Both forms are valid for calling services:
service.call("group", "set", object_id="real_power_loads", entities=real)

# Shorthand dot notation (equivalent):
group.set(object_id="real_power_loads", entities=real)
```

---

## ✅ Pre-Commit Checklist

Before saving and applying any change:

- [ ] File is in the correct package directory
- [ ] Automation/sensor has a complete header comment block
- [ ] All `float()` and `int()` calls have a default value
- [ ] Triggers that should be edge-triggered have `from:` specified
- [ ] No new use of `input_text` for structured/multi-field data
- [ ] `notify.*` calls go through central script, not direct service
- [ ] New diagnostic sensors are excluded from recorder in `configuration.yaml`
- [ ] Automation ID follows naming convention
- [ ] YAML validated via Developer Tools before reload
- [ ] If `alert:` entities changed — HA restart required, noted in commit message
- [ ] New entity IDs checked against `.storage/core.entity_registry` — no UI duplicates
- [ ] No `{% if %}` / `{% for %}` blocks used to conditionally emit YAML keys — use `choose:` branches instead

---

*Last updated: 2026-04-29*  
*Updated by: Added Rule 6 (Jinja2 block tags cannot emit YAML keys — causes illegal % token / recovery mode) + pre-commit checklist entry. Added pyscript rules section 2026-04-22.*
