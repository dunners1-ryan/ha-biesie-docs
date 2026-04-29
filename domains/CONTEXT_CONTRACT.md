##########################################################
# HABiesie — CONTEXT CONTRACT
# Domain Audit: Global Context / Shared State
# Generated: 2026-04-16
# Source: packages/context/context_global.yaml
#         packages/context/context_night.yaml
#         packages/context/context_presence.yaml
#         packages/context/context_schedules.yaml
##########################################################

---

## Section 1: Domain Responsibility

The Context domain is the **shared state hub** of the system. It owns:
- Night mode detection (sun elevation → confidence → binary_sensor)
- Global home context (away / night / late_night / home)
- Trust model infrastructure: staff/contractor presence inputs, derived sensors, schedules
- Mode flags (guest, holiday, entertaining, bedtime)

It is an **infrastructure layer** domain — its outputs are consumed by almost
every other domain (security, lighting, presence, alerts) but it consumes very
little from others.

**Architectural violation:** `context_presence.yaml` owns trust-model booleans and
staff/gardener scheduling — this properly belongs in `presence/` but was placed here
historically. Do not move it until a dedicated refactor session has mapped all
consumers. See BUG-CTX01.

---

## Section 2: File Inventory

| File | Contents |
|------|----------|
| `context/context_global.yaml` | `sensor.home_context`, `binary_sensor.security_night_mode`, `binary_sensor.security_nobody_home` |
| `context/context_night.yaml` | Night confidence ladder, night binary sensors, `binary_sensor.night_confirmed` |
| `context/context_presence.yaml` | Trust model: all staff/contractor booleans, datetimes, derived sensors, maid/gardener schedules |
| `context/context_schedules.yaml` | `input_boolean.bedtime_mode`, `input_datetime.bedtime_time` — stub, should merge into lighting_helpers.yaml |

---

## Section 3: Entity Reference

### Night Detection (context_night.yaml)

```
binary_sensor.nighttime               — sun below horizon (simple)
binary_sensor.night_early             — sun elevation < 2° (early dusk)
binary_sensor.civil_night             — sun elevation < −6°
binary_sensor.nautical_night          — sun elevation < −12°
binary_sensor.astronomical_night      — sun elevation < −18°
binary_sensor.night_occupied          — civil_night AND any_room_occupied
binary_sensor.night_confirmed         — night_confidence >= 60 (canonical gate)
binary_sensor.quiet_arrival_mode      — night_confirmed AND bedrooms_occupied
sensor.night_confidence               — 0/40/60/80/100 based on sun elevation
```

**Canonical night entity for automations:** `binary_sensor.night_confirmed`
Never use `binary_sensor.nighttime` or `binary_sensor.civil_night` directly in
automation conditions — use `night_confirmed` which is already gated on confidence ≥ 60.

### Global Context (context_global.yaml)

```
sensor.home_context           — away / night / late_night / home
binary_sensor.security_night_mode    — alias of binary_sensor.night_confirmed
binary_sensor.security_nobody_home   — anyone_connected_home=off AND staff_on_site=off
```

Note: `sensor.home_context` depends on `sensor.security_mode` (from security/).
This is a layering inversion — context/ should not import from security/.
See BUG-CTX03.

### Trust Model (context_presence.yaml) — ARCHITECTURAL NOTE: belongs in presence/

```
# Input booleans (manual / auto-set by schedules)
input_boolean.guest_mode               — Guest / House Sitter Mode (manual)
input_boolean.low_trust_present        — ⚠️ LEGACY — never use in automations
input_boolean.maid_on_site             — auto-set by maid_schedule_start/end
input_boolean.gardener_on_site         — auto-set by gardener_schedule_start/end
input_boolean.staff_on_site            — ⚠️ LEGACY — never use in automations
input_boolean.contractor_on_site       — manual toggle
input_boolean.low_trust_enabled        — gates maid/gardener schedules
input_boolean.entertaining_mode        — manual
input_boolean.holiday_mode             — manual
input_boolean.staff_on_site_override   — when ON: forces staff_on_site OFF (clears low trust)
input_boolean.boundary_permissive_override  — manual boundary gate override

# Input datetimes (configures schedule windows)
input_datetime.maid_start
input_datetime.maid_end
input_datetime.gardener_start
input_datetime.gardener_end

# Derived binary sensors (USE THESE in automations — never the input_booleans above)
binary_sensor.staff_on_site       — maid_on_site OR gardener_on_site (with override)
binary_sensor.low_trust_present   — staff_on_site OR contractor_on_site (with override)
```

### Schedules / Bedtime (context_schedules.yaml + context_presence.yaml)

```
input_boolean.bedtime_mode         — stub, from context_schedules.yaml
input_datetime.bedtime_time        — stub, from context_schedules.yaml
```
Maid/gardener schedule automations live in `context_presence.yaml`:
- `maid_schedule_start` — sets `input_boolean.maid_on_site` on at `maid_start` time (weekdays only, if low_trust_enabled)
- `maid_schedule_end` — clears `input_boolean.maid_on_site` at `maid_end` time
- `gardener_schedule_start` — same pattern for gardener
- `gardener_schedule_end` — same pattern for gardener

---

## Section 4: Critical Rules

### RULE 1 — Trust model — always use the derived sensor, never the input_boolean

```
✅ ALWAYS USE:   binary_sensor.staff_on_site
                 binary_sensor.low_trust_present
❌ NEVER USE:    input_boolean.staff_on_site      (never auto-set)
                 input_boolean.low_trust_present   (never auto-set)
```

The derived binary sensors incorporate the `staff_on_site_override` flag.
The input_booleans are exposed in the UI for visibility but are never set
by automations — they are effectively display-only legacy entities.

### RULE 2 — Night gating — always use night_confirmed

```
✅ ALWAYS USE:   binary_sensor.night_confirmed     (confidence >= 60)
❌ AVOID:        binary_sensor.nighttime            (sun.below_horizon — immediate)
                 binary_sensor.civil_night          (no confidence gating)
```

### RULE 3 — Never set bedtime_mode from automations

`input_boolean.bedtime_mode` is a UI-only helper. No automation logic should
gate on it — use `binary_sensor.night_confirmed` + occupancy sensors instead.

---

## Section 5: Published Outputs (Consumed by Other Domains)

| Entity | Consumed By |
|--------|-------------|
| `binary_sensor.night_confirmed` | security, lighting, alerts_doors |
| `binary_sensor.security_night_mode` | security (alias) |
| `binary_sensor.security_nobody_home` | lighting_security, dashboard |
| `binary_sensor.staff_on_site` | security, context_global |
| `binary_sensor.low_trust_present` | security, lighting_departure, alerts_doors |
| `input_boolean.guest_mode` | security |
| `input_boolean.holiday_mode` | security, lighting |
| `input_boolean.entertaining_mode` | security |
| `sensor.home_context` | dashboard, lighting (future) |
| `sensor.night_confidence` | informational/dashboard |

---

## Section 6: Known Bugs

### BUG-CTX01 [ARCH / LOW] — Trust model in wrong package
**Problem:** `context_presence.yaml` owns trust model booleans, datetimes, derived
sensors, and schedules. This belongs in `presence/` per the architecture.
The violation has existed since initial setup and all consumers reference these
entities correctly by entity_id, so there is no functional impact.
**Fix:** Create `packages/presence/presence_trust.yaml` with the trust model
infrastructure, delete `context_presence.yaml`. Update `presence_helpers.yaml`
or `presence_core.yaml` to absorb any remaining references.
**Risk:** Medium — file rename requires `ha core check` before reload.

### BUG-CTX02 [LOW] — `context_schedules.yaml` is an orphaned stub
**Problem:** `context_schedules.yaml` contains only `input_boolean.bedtime_mode`
and `input_datetime.bedtime_time`. These are lighting scheduling helpers that
have no consumers in any automation. The file adds a package directory entry
for no functional benefit.
**Fix:** Move `bedtime_mode` and `bedtime_time` into `packages/lighting/lighting_helpers.yaml`.
Delete `context_schedules.yaml`. Wire `bedtime_mode` and `bedtime_time` into the
kids bedtime automation or remove entirely if unused.

### BUG-CTX03 [ARCH / LOW] — `context_global.yaml` imports from security/
**Problem:** `sensor.home_context` reads `sensor.security_mode` from the security
domain. This creates a reversed dependency: the infrastructure layer (context/)
depends on the decision layer (security/). If security/ is broken, context/ breaks.
**Fix options:**
  1. Move `sensor.home_context` into `security/security_core.yaml` (where it logically belongs)
  2. Derive `sensor.home_context` from `binary_sensor.night_confirmed` + presence directly
**Recommend:** Option 2 — removes the circular dependency and makes context/ truly
self-contained.

---

## Section 7: Architecture Notes

- The context/ package has no automation triggers of its own beyond the maid/gardener
  schedules. It is a pure state-publishing package.
- `binary_sensor.night_confirmed` is used by at least 4 other packages. Never remove it.
- `binary_sensor.security_night_mode` is a thin alias over `night_confirmed` — it exists
  so that security automations can say "night mode" without importing from context/.
- `staff_on_site_override` is the only way to force-clear low trust during a working day
  (e.g. Ryan is home with the maid and wants full trust). Dashboard toggle.

---

*Last updated: 2026-04-16*
*Source: packages/context/*.yaml*
