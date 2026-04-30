##########################################################
# HABiesie — CONTEXT CONTRACT
# Domain Audit: Global Context / Shared State
# Generated: 2026-04-16
# Updated:   2026-04-30 — BUG-CTX01/02/03 fixed
# Source: packages/context/context_global.yaml
#         packages/context/context_night.yaml
##########################################################

---

## Section 1: Domain Responsibility

The Context domain is the **shared state hub** of the system. It owns:
- Night mode detection (sun elevation → confidence → binary_sensor)
- Global home context (away / night / late_night / home)
- Helper aliases for security and nobody-home state

It is an **infrastructure layer** domain — its outputs are consumed by almost
every other domain (security, lighting, presence, alerts) but it consumes very
little from others.

**Trust model was moved (2026-04-30):** `context_presence.yaml` content migrated to
`packages/presence/presence_trust.yaml` (BUG-CTX01 fix). Context/ no longer owns
trust model entities. See PRESENCE_CONTRACT.md for trust model reference.

---

## Section 2: File Inventory

| File | Contents |
|------|----------|
| `context/context_global.yaml` | `sensor.home_context`, `binary_sensor.security_night_mode`, `binary_sensor.security_nobody_home` |
| `context/context_night.yaml` | Night confidence ladder, night binary sensors, `binary_sensor.night_confirmed` |
| ~~`context/context_presence.yaml`~~ | **DELETED 2026-04-30** — content moved to `packages/presence/presence_trust.yaml` (BUG-CTX01) |
| ~~`context/context_schedules.yaml`~~ | **DELETED 2026-04-28** — `bedtime_mode` moved to `lighting_helpers.yaml` (BUG-CTX02) |

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

### Trust Model — moved to `packages/presence/presence_trust.yaml` (2026-04-30)

Trust model entities (input booleans, datetimes, derived binary sensors, maid/gardener
schedule automations) now live in `presence/presence_trust.yaml`.
See PRESENCE_CONTRACT.md for the full entity reference.

**Key rule unchanged:** always use `binary_sensor.staff_on_site` and
`binary_sensor.low_trust_present` in automations — never the input_booleans.

### Bedtime mode — moved to `packages/lighting/lighting_helpers.yaml` (2026-04-28)

```
input_boolean.bedtime_mode    — set by kids/full bedtime automations, cleared by morning
# input_datetime.bedtime_time — deleted (unused)
```

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

### ~~BUG-CTX01~~ ✅ FIXED 2026-04-30 — Trust model in wrong package
Created `packages/presence/presence_trust.yaml` with all trust model content.
Deleted `packages/context/context_presence.yaml`. Entity IDs unchanged — all consumers
reference by entity_id. ha core check passed. Reload: Helpers + Templates + Automations.

### ~~BUG-CTX02~~ ✅ FIXED 2026-04-28 — `context_schedules.yaml` was an orphaned stub
`input_boolean.bedtime_mode` moved to `packages/lighting/lighting_helpers.yaml`.
`input_datetime.bedtime_time` deleted (unused). `context_schedules.yaml` deleted.

### ~~BUG-CTX03~~ ✅ FIXED 2026-04-30 — `context_global.yaml` imported from security/
`sensor.home_context` now derives from `binary_sensor.security_nobody_home` (self-contained
in context_global.yaml) and `binary_sensor.night_confirmed` (context_night.yaml) directly.
No longer reads `sensor.security_mode` or `sensor.security_trust_mode` from security/.
`trust` attribute simplified: `low_trust` / `normal` via `binary_sensor.low_trust_present`.

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
