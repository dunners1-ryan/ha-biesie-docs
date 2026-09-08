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
| Any `.py` file inside `custom_components/*` edited (even an existing one) | Full restart — Python modules are already imported in memory; a config-entry reload re-runs setup using the STALE code, it does not re-import the file. Confirmed 2026-09-08 patching `hikvision_next`. |
| New packages directory | `!include_dir_named` requires restart |
| `customize.yaml` | Requires restart |
| `.storage/lovelace` changes — **edited as a raw file** (`Write`/`Edit` on the JSON directly) | **Full HA restart** — confirmed 2026-07-03 and again 2026-07-06 that a browser hard refresh (`Cmd+Shift+R`) is NOT reliably sufficient; the frontend can hold a stale in-memory copy of the dashboard config regardless of browser cache. Restart to guarantee it takes effect. Also avoid opening that dashboard's UI editor before restarting — an autosave from the stale in-memory copy would silently revert a direct `.storage` edit. **Does NOT apply** to a change pushed via the `lovelace/config/save` WebSocket command (see below) — that path goes through the same mechanism the UI editor itself uses and takes effect immediately, no restart, confirmed 2026-09-07. |

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
   - .storage/lovelace changed → ⚠️ Full HA restart required (hard refresh alone is not
     reliable) — UNLESS pushed via the lovelace/config/save WebSocket call instead of a
     raw file edit, which needs neither (see "Deploying a dashboard edit without a
     restart" above)
3. Verify in Developer Tools → States
4. Commit + update docs in same session
```

---

## 🩺 Live API / Recorder Query Rules

**2026-09-06 — logged, then corrected same session.** While diagnosing why the
boundary street light didn't come on, a `curl` to `/api/logbook/<date>` with no
`entity=` filter over a ~2-day window was issued against the live Supervisor API —
initially assumed (from an ambiguous `ha core logs` line, "Thread DbWorker_2 is
still running at shutdown", right before a Core restart) to have caused that
restart. **That attribution was wrong.** PROJECT_STATE.md's open TODO already
documented three unexplained Core restarts that same evening (~20:46 and ~20:54
SAST, no update involved, candidate cause a `hikvision_next` entity-setup
`AttributeError` / watchdog force-kill) from earlier, unrelated work — the restart
observed here was the same pre-existing, still-unresolved event, not something this
query triggered. See PROJECT_STATE.md for the actual open investigation.

The unscoped-query practice below is still worth avoiding on its own merits (an
unfiltered multi-day logbook build is real unnecessary load on a shared recorder),
just not because it was confirmed to have crashed anything here:

- **Never call `/api/logbook/<start>` without an `entity=` filter**, and never over
  more than a few hours — it builds the *entire house* logbook for that window.
- **Prefer `/api/history/period/<start>?filter_entity_id=...&minimal_response`**
  for entity investigation — scope to the specific entities needed, and pass an
  explicit `end_time` (the default window is short and silently truncates a query
  meant to reach "now").
- If a broad query is genuinely needed, page it (a few hours at a time) rather than
  requesting a multi-day span in one call.
- **Before blaming your own diagnostic query for a restart/outage you observe,
  check PROJECT_STATE.md's open TODO first** — this box has had recurring
  unexplained restarts independent of any session's actions; a coincidental restart
  during your own investigation is more likely to be that than something you did.

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

### Rule 5b — Never put `initial:` on a legacy-YAML helper that holds live/mutable state

Confirmed live, independently, in two domains the same day (2026-09-06 —
Water Cooler and Gas Bottles, see `docs/domains/UTILITIES_CONTRACT.md`
Sections 3 and 8c-bis/8c-ter/8's Session Log for both real incidents and
their fixes): a legacy-YAML `input_number`/`input_boolean`/`input_select`/
`input_datetime` (defined via `packages/`, not created through the UI) with
an `initial:` key resets to that exact value on **every HA Core restart**,
unconditionally — not just first-ever creation, and not just an occasional
restore-state gap. Proven both times with a real restart (not just a config
check): set a deliberately different test value, restart, watch it revert.

This silently destroys any automation-refined state — an EMA average, a
"which physical thing is connected" select, a stock count, a status flag —
every single restart, with no error and no `unknown` state to notice
anything is wrong. Both real incidents went undetected for hours precisely
because the reverted value looked plausible (a real-shaped number/date), not
obviously broken.

```yaml
# ❌ WRONG — silently resets to 3.9 on every restart, discarding every real
# EMA refinement an automation has made since
input_number:
  watercooler_avg_days_per_bottle:
    initial: 3.9

# ✅ CORRECT — no initial: on anything an automation writes to or that
# represents live state the user changes over time. The entity keeps
# whatever it currently holds; nothing to silently fall back to.
input_number:
  watercooler_avg_days_per_bottle:
    min: 0.5
    max: 30
    step: 0.1
    unit_of_measurement: "d"
```

**Test before deciding an entity is safe to keep `initial:` on** — don't
guess from the name. The rule of thumb that held in both real fixes:

- **Remove `initial:`** from anything an automation calls `set_value`/
  `turn_on`/`turn_off`/`select_option`/`set_datetime` on, OR anything a
  human changes via the dashboard to reflect an evolving real-world fact
  (which physical bottle is connected, current stock, a status select).
  Losing this on a restart is a silent, hard-to-notice data-corruption bug.
- **`initial:` is still fine** on genuine settings a human tunes rarely
  and where reverting to the shipped default is low-stakes — thresholds,
  rate/price references, reminder times/intervals — and on a toggle whose
  correct idle value already equals its `initial` (e.g. a per-transaction
  "include X" toggle that's supposed to sit at `false` between uses
  anyway, so reverting to `false` on restart isn't actually wrong).
- When genuinely unsure, prove it live rather than assume: set the entity
  to a value deliberately different from its would-be `initial:`, force a
  real restart (`curl -X POST .../core/restart`, then poll until the API
  responds `200` again — a fast `200` on the first check usually means the
  restart never actually happened, not that it was quick), and check
  whether the test value survived.

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
            inline_keyboard:        # Telegram extras at TOP level of data — never nested under data.data
              - "Ack:/ack_alert"
  default:
    - action: notify.send_message
      data:
        message: "Hello"
        disable_notification: "{{ sev == 'information' }}"   # TOP level of data
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

### Rule — `notify.send_message` never uses `data.data`

`notify.send_message` (HA 2024.8+) has a strict schema: it rejects a `data` key nested inside `data`. This error reads as `extra keys not allowed @ data['data']` and breaks script execution unless `continue_on_error: true` is set (in which case it fails silently).

Telegram-specific extras (`inline_keyboard`, `disable_notification`) go **directly inside `data:`**, at the same level as `message:`.

```yaml
# ❌ WRONG — schema rejects data.data; breaks silently or fatally
- action: notify.send_message
  target:
    entity_id: notify.telegram_bot_5527
  data:
    message: "Alert"
    data:                          # ← rejected
      inline_keyboard:
        - "Acknowledge:/ack"
      disable_notification: true

# ✅ CORRECT — Telegram extras at top level of data
- action: notify.send_message
  target:
    entity_id: notify.telegram_bot_5527
  data:
    message: "Alert"
    inline_keyboard:               # ← same level as message
      - "Acknowledge:/ack"
    disable_notification: true     # ← same level as message
```

> **Note:** The old `notify.STD_*` group services still accept `data.data` for companion app push extras (`push:`, `channel:`, `ttl:`, `priority:`). Only `notify.send_message` rejects it.

### Severity levels

| Level | When to use |
|---|---|
| `info` | Informational, low urgency, suppressed in quiet hours |
| `warning` | Action recommended, always delivered |
| `critical` | Immediate action required, always delivered, escalated |

---

## 🎨 Dashboard Card Standards

### Deploying a dashboard edit without a restart

No REST endpoint exists for lovelace config — only the WebSocket API does. Editing
`.storage/lovelace.<dashboard_id>` directly needs a full restart (see the table above).
To avoid that: connect to `ws://supervisor/core/websocket` (or `wss://<host>:8123/api/
websocket` from outside the supervisor network), authenticate with `{"type": "auth",
"access_token": "<token>"}`, then:
1. `{"id": 1, "type": "lovelace/config", "url_path": "<dashboard-url-path>"}` — read the
   live config first. Diff it against the `.storage` file on disk before editing anything;
   if they differ, another session/the UI editor changed it since you last looked.
2. Edit the returned config object in Python/similar (JSON in, JSON out — same shape as
   the `.storage` file's `data.config`).
3. Validate any new/changed Jinja template (card content, `card_mod` style) via
   `POST /api/template` (`{"template": "..."}`) BEFORE pushing — confirms it renders
   against live state and catches syntax errors without touching the dashboard.
4. `{"id": 2, "type": "lovelace/config/save", "url_path": "<dashboard-url-path>",
   "config": <edited config>}` — takes effect immediately, no restart, no stale
   frontend cache (unlike the raw-file path).
5. Read the config back (`lovelace/config` again) AND re-read the `.storage` file from
   disk — confirm both match what you intended, not just that the save call succeeded.

No `websockets`-equivalent Python package is preinstalled — `pip install` hits an
externally-managed-environment error; use a throwaway venv (`python3 -m venv`, install
into it, delete it after) rather than `--break-system-packages` against the system
Python. Get the dashboard's `url_path` from `.storage/lovelace_dashboards` (`data.items[]
.url_path`), not its storage-key id (`dashboard_operations` the file vs. `dashboard-
operations` the url_path — they differ by a hyphen/underscore and are NOT the same
string). First used 2026-09-07 for the Boundary Lighting Watchdog card
(LIGHTING_CONTRACT.md / PROJECT_STATE.md same-day entry).

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

### Rule — `custom:auto-entities` `filter.template` output MUST be bare entity IDs, one per line

The vendored `www/community/lovelace-auto-entities/auto-entities.js` in this repo does
**not** YAML-parse a `filter.template` result. Its renderer does:
```js
e._template = "string" == typeof t ? t.split(/[\s,]+/) : t;
```
Any template with more than one Jinja statement (any `{% %}` block) always renders to a
plain string, so it always takes the `.split(/[\s,]+/)` path — the result is naively
tokenized on whitespace/commas, **not** parsed as YAML. A mapping/dict row style like
`- entity: sensor.x\n  secondary_info: last-changed` gets shredded into garbage tokens
(`-`, `entity:`, `sensor.x`, `secondary_info:`, `last-changed`, …), which breaks the card
with a silent, persistent "Configuration error" — confirmed live 2026-07-10, survived
multiple full HA restarts because the card's internal state doesn't self-heal, only a
fresh render with a corrected template fixes it.

```yaml
# ✅ CORRECT — bare entity id, one per line (whitespace-split-safe)
filter:
  template: >
    {% for e in states.binary_sensor | selectattr('state','eq','on') | list %}
      {{ e.entity_id }}
    {% endfor %}

# ❌ WRONG — dict/mapping rows get shredded by the whitespace splitter, not YAML-parsed
filter:
  template: >
    {% for e in states.binary_sensor | selectattr('state','eq','on') | list %}
    - entity: {{ e.entity_id }}
      secondary_info: last-changed
    {% endfor %}
```

Do not add a `{% if ... | count == 0 %}[]{% endif %}` "empty list" fallback either — an
emitted literal `[]` string just becomes one token `"[]"` (a nonexistent entity, silently
ignored), not an actual empty array. Emitting nothing when there's nothing to show is
correct and already proven safe (see the Critical/Warning/Info severity cards in
`alerts_summary.yaml`'s dashboard, which have used this exact bare-token pattern
successfully since before this rule was written).

If per-row extras (`secondary_info`, per-row `card_mod`, etc.) are genuinely needed, they
require the `filter.include`/`filter.exclude` static-attribute-matching form instead — not
`filter.template` in this repo's vendored card version.

---

## 🏷️ Label-Based Dynamic Device Groups (added 2026-08-21)

For an open-ended device fleet (batteries, or anything else where "which devices" grows
over time) — do **not** hand-list entity IDs in YAML. Every existing example of that
pattern (`alerts_batteries.yaml`'s two hardcoded Honor tablets) requires a YAML edit +
restart per device added. Instead:

1. **Create a label** (`.storage/core.label_registry` or Settings → Labels) — e.g.
   `battery_monitor`. Onboarding a device = applying the label to its entity (or its
   device — both resolve, see point 2). No YAML edit.
2. **Read it with `label_entities('label_id')`** in a template sensor — this is a stock
   HA Jinja global (2024.4+) and, importantly, it resolves entities that carry the label
   directly *and* entities whose **device** carries it — same resolution the vendored
   `custom:auto-entities` card's own `filter.include: [{label: ...}]` uses (confirmed
   against `www/community/lovelace-auto-entities/auto-entities.js`), so template sensors
   and dashboard card filters agree on membership without extra plumbing.
3. **One roster sensor is the source of truth.** Compute the full per-device list (name,
   category, severity, whatever) once, in one template sensor's attribute (a list of
   dicts). Every consumer — alert context sensor, dashboard markdown cards — reads that
   attribute instead of re-deriving the list, so there's exactly one place that knows how
   to categorise/score a device. See `sensor.device_battery_fleet` in
   `alerts_device_batteries.yaml`.
4. **Dashboard rendering: one markdown card per section, not one card per device.** A
   markdown card's `content:` can loop a Jinja list (`{% for d in devs %}`) to render an
   arbitrary number of rows — HA's template render-tracking subscribes to whatever
   entities/labels get touched during evaluation, so it stays reactive even though the
   entity list isn't known at YAML-authoring time. This is unrelated to (and unaffected
   by) the `custom:auto-entities` `filter.template` limitation above — that's a quirk of
   the vendored card's own JS, not of markdown cards or HA's own templating.
5. **Time-series estimates without a recorder query:** HA templates have no stock
   "value N days ago" lookup. For a slow-moving metric (battery drain, etc.) build a
   small **self-referencing trigger-based template sensor** instead: `trigger: time_pattern`
   (once/day is enough) reading `this.attributes.get('log', {})` (its own previous value —
   template sensors restore state+attributes across restarts, no recorder dependency),
   appending today's reading per entity, capped to a rolling window. A second sensor
   fits a rate from that log against the live current value. See
   `sensor.device_battery_history_log` in `alerts_device_batteries.yaml` for the full
   pattern, including the entity-not-a-mapping pitfall below.

**Pitfall — `rejectattr`/`selectattr` on plain lists, not dicts:** `rejectattr('0', 'eq', x)`
does **not** mean "index 0" — Jinja's `attribute` resolution tries `getattr` then
`obj['0']` (string key), which raises `TypeError` against a plain `[date, value]` list.
Filtering list-of-lists by position needs an explicit `{% for %}` + `{% if p[0] == x %}`
loop, not `select`/`rejectattr`.

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

## 📝 Session Documentation Rules

### Rule 7 — Documentation MUST update with code in the SAME session

This rule is **blocking, not advisory.** A session that ships code changes without corresponding doc updates is **INCOMPLETE.** Never silently leave drift between live config and contracts.

Three doc-drift reconciliations happened in 2026-05 alone (trust chain, zone arming, camera fleet). Each cost a full session. The root cause: code shipped faster than docs were updated. This rule is the engineering defense.

#### What counts as a "code change" (any of these requires a doc update)

- Adding, renaming, or removing an entity (sensor, binary_sensor, input_boolean, automation, script, etc.)
- Changing the source-data list of a template sensor or zone aggregation
- Changing trigger / condition / action on an automation
- Closing or opening a bug
- Deprecating, removing, or replacing hardware (cameras, sensors, alarm zones)
- Any change that affects observable behaviour from the dashboard or another package

#### Required doc updates per change type

| If you changed... | Update this |
|-------------------|-------------|
| Security domain entity | SECURITY_CONTRACT.md |
| Presence domain entity | PRESENCE_CONTRACT.md |
| Lighting | LIGHTING_CONTRACT.md |
| Power | POWER_CONTRACT.md |
| Notifications | NOTIFICATIONS_CONTRACT.md |
| Water | WATER_CONTRACT.md |
| Context (trust, night, schedules) | CONTEXT_CONTRACT.md |
| Closed a bug | The contract that listed it — mark CLOSED with date |
| Opened a bug | The relevant contract — add BUG-XXX entry |
| Renamed/added/removed an entity | PROJECT_STATE.md "Locked Entity Names" |
| Removed hardware | PROJECT_STATE.md + relevant contract |
| Any functional change | PROJECT_STATE.md session log entry |

#### Session-close checklist

See `SESSION_CHECKLIST.md` (repo root) for the full version. Required before any session is marked complete:

```
[ ] ha core check passes
[ ] Required reloads done and functional checks run
[ ] ./gitupdate.sh committed in /config/
[ ] PROJECT_STATE.md session log entry added
[ ] PROJECT_STATE.md Locked Entity Names updated (if entities changed)
[ ] Relevant domain contract(s) updated (entity tables + bug list)
[ ] Both repos committed with paired commit messages
```

#### When a session ends with checklist items unsatisfied

Flag it explicitly: **"SESSION INCOMPLETE — the following doc updates are still owed: [list]."** Do not silently move on.

If time pressure prevents finishing: commit with `WIP: docs incomplete` in the message AND add a resume note at the top of `PROJECT_STATE.md` so the next session resumes there before doing anything new.

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
- [ ] `notify.send_message` calls have NO nested `data.data` — Telegram extras (`inline_keyboard`, `disable_notification`) are at top level of `data:`
- [ ] No `initial:` on a new helper that an automation writes to or that represents live/mutable state (Rule 5b) — settings/thresholds/references are still fine to keep it

---

*Last updated: 2026-09-06*  
*Updated by: Added Rule 5b (`initial:` on a legacy-YAML input_* helper resets it on every restart, not just first creation — confirmed live in two independent domain fixes the same day, Water Cooler and Gas Bottles) + matching pre-commit checklist entry.*
*Last updated: 2026-08-21*  
*Updated by: Added "Label-Based Dynamic Device Groups" section (label_entities(), one-roster-sensor-as-source-of-truth, markdown-card-per-section rendering, self-referencing trigger template for time-series estimates without a recorder query, and the rejectattr-on-plain-lists pitfall) — new pattern introduced by `alerts_device_batteries.yaml` / the Batteries dashboard view (2026-08-21).*
*Last updated: 2026-04-29*  
*Updated by: Added Rule 6 (Jinja2 block tags cannot emit YAML keys) + pyscript rules (2026-04-22). Fixed Rule 6 example (was showing broken data.data pattern). Added notify.send_message data.data rule + pre-commit checklist entry (2026-04-29).*
