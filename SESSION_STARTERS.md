# HABiesie — Session Starter Prompts
> Updated 2026-04-29. Contracts are authoritative — always prefer CONTRACT over CONTEXT for real work.

---

## 🔄 Claude Chat — Context Refresh Commands

Claude chat (claude.ai "HA Biesie — Way of Work") fetches docs live from:
  https://github.com/dunners1-ryan/ha-biesie-docs

Commands to use at session start:
- `refresh context`          → fetches PROJECT_STATE.md
- `fetch coding standards`   → fetches CODING_STANDARDS.md
- `fetch [domain] contract`  → fetches domains/[DOMAIN]_CONTRACT.md
- `refresh all`              → fetches PROJECT_STATE + CODING_STANDARDS

No file uploads or pasting needed. GitHub sync is automatic on every `gitupdate.sh`.

---

## 🏁 Claude Code — Session Closing Ritual

Run this at the end of EVERY Claude Code session before disconnecting:

1. Update `/config/docs/PROJECT_STATE.md`
   - Append session summary to the bottom of the session log section
   - Format: `*2026-MM-DD session: [what changed, what was fixed, what was deferred]*`

2. Update `/config/docs/CODING_STANDARDS.md` if any new rules were learned

3. Run: `./gitupdate.sh "docs: update PROJECT_STATE [brief description]"`
   This auto-syncs to ha-biesie-docs — Claude chat will have current state next session.

4. Confirm: "docs synced and pushed to ha-biesie-docs ✅"

---

## 🚀 Universal Session Starter

```
I'm working on my Home Assistant config HABiesie — production smart home, Johannesburg SA.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Confirm you understand the loaded docs before proceeding.

Session rules:
- All code needs standard header comment block (see CODING_STANDARDS.md)
- All new automations go in correct package file, never automations.yaml
- All float/int conversions must have defaults: float(0)
- Never produce code without stating which file it goes in
- Always use binary_sensor.low_trust_present NOT input_boolean.low_trust_present
- All notifications through script.notify_*_event, never direct notify.*

Task: [DESCRIBE]
```

---

## 🛡️ Security Session

```
Working on Security package of HABiesie HA config.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load domains/SECURITY_CONTRACT.md  ← authoritative, use this not SECURITY_CONTEXT

Constraints (VERIFIED by audit):
- Camera naming LOCKED: cam01_street_driveway pattern
- Do NOT modify security_event_router automation
- security_cam* prefixed snapshot files are unreferenced orphans — safe to clean
- cam03_rear_perimeter does not exist — do not reference it
- Trust model fix = replace input_boolean.low_trust_present with binary_sensor version

Today's focus: [e.g. "Sprint 1 quick fixes" / "Fix duplicate snapshots"]
```

---

## ⚡ Power/Energy Session

```
Working on Power/Energy/Prepaid package of HABiesie HA config.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load domains/POWER_CONTRACT.md

Constraints (VERIFIED by audit):
- Core sensor names (inverter_*, prepaid_*) are LOCKED
- Dual inverter: Master=grid/losses/BMS, Slave=PV/load/battery
- Financial model is stable — do not change prepaid tracking logic
- Authoritative source: prepaid meter, not inverter totals
- sensor.inverter_1_battery is internal — use sensor.inverter_battery_soc publicly

Today's focus: [e.g. "Buy Score v2" / "Confidence layer"]
```

---

## 🚰 Water Session

```
Working on Water package of HABiesie HA config.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load domains/WATER_CONTRACT.md

Constraints (VERIFIED by audit):
- Read water lifecycle contract rules before any change
- Safety is INDEPENDENT and ABSOLUTE — never gate it
- Capture OBSERVES only — no control decisions
- Truth = physical depth sensor, not switch state
- sensor.water_state never produces "full" — that trigger is dead code
- Policy helpers (water_policy_helpers.yaml thresholds) are orphaned

Today's focus: [e.g. "Fix trigger integrity" / "State machine"]
```

---

## 🧭 Presence Session

```
Working on Presence package of HABiesie HA config.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load domains/PRESENCE_CONTRACT.md

Constraints (VERIFIED by audit):
- Person naming LOCKED
- AP location sensors LOCKED
- Trust model lives in context/context_presence.yaml (wrong location but don't move yet)
- ALWAYS use binary_sensor.staff_on_site, binary_sensor.low_trust_present
- NEVER use input_boolean versions of these in automations
- presence_test_arrival automation is a test artifact — should be removed
- Motion sensors are currently stubs (hardcoded off) — confidence is AP-only

Today's focus: [e.g. "Fix trust chain" / "Wire arrival_detected"]
```

---

## 🔔 Alerts Session

```
Working on Alerts/Notifications package of HABiesie HA config.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load domains/ALERTS_CONTRACT.md and domains/NOTIFICATIONS_CONTRACT.md

Constraints (VERIFIED by audit):
- sensor.alert_device_entities is SINGLE SOURCE OF TRUTH
- All notifications MUST go through script.notify_*_event
- alerts_water.yaml and alerts_security.yaml are currently empty stubs
- alerts_presence.yaml is an empty stub
- Temperature and presence alerts currently bypass pipeline (known violations)
- Dual notification bug in alerts_device_power.yaml (resolve before migrating)

Today's focus: [e.g. "Implement alerts_water.yaml" / "Fix notification bypasses"]
```

---

## 🐛 Fix Session (Group A — Trust Model)

The highest-value fix in the system. Repairs security + lighting + door alerts in one pass.

```
Working on HABiesie trust model fix — Group A from SYSTEM_CONTRACT.md

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load SYSTEM_CONTRACT.md sections 4 and 7

Task: Implement Group A trust model fixes in order:

A1. packages/context/context_presence.yaml
    Restore input_datetime.low_trust_start and input_datetime.low_trust_end
    (they were commented out — entity registry shows them as orphaned)

A2. packages/security/security_core.yaml
    Fix binary_sensor.boundary_permissive_window to use low_trust_start/end

A3. packages/security/security_core.yaml + packages/security/security_logic.yaml
    Replace input_boolean.low_trust_present → binary_sensor.low_trust_present
    Replace input_boolean.staff_on_site → binary_sensor.staff_on_site

A4. packages/lighting/lighting_departure.yaml
    Same replacement as A3

Show me each change before applying. After all 4 done, I'll validate via HA YAML check.
```

---

## 🐛 Fix Session (Group D — Security Quick Wins)

No-risk fixes, each is 1-3 lines. Can batch in one HA reload.

```
Working on HABiesie security quick fixes — Group D from PROJECT_STATE.md

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load domains/SECURITY_CONTRACT.md Section 9 Sprint 1

Fix all Sprint 1 items:
D1. security_core.yaml:112-113 — two entity ID typos
D2. cameras_processing.yaml:131 — cam06 reads cam09
D3. security_helpers.yaml — cam10 history key name
D4. security_logic.yaml — add unique_id to security_correlation
D5. context_presence.yaml — low_trust_start/end (A1 from trust model)
D6. presence_boundary.yaml — remove presence_test_arrival automation

Show each change with exact line numbers before applying.
```

---

## 📊 Dashboard Session

```
Working on a dashboard in HABiesie HA config.

Say "refresh context" in Claude chat — it fetches live from https://github.com/dunners1-ryan/ha-biesie-docs
Then load relevant domain CONTRACT

Dashboard: [NAME / PATH]
Change: [WHAT YOU WANT]

Card toolkit: mushroom-template-card, auto-entities, card-mod, apexcharts-card,
plotly-graph, sunsynk-power-flow-card, power-flow-card-plus, expander-card,
mini-graph-card, config-template-card, navbar-card, logbook-card, history-graph,
button-card, fold-entity-row, multiple-entity-row, flex-table, decluttering-card,
timer-bar-card, Energy Flow Card Plus, bubble-card, ultra-card

Rules:
- Backend must be stable before UI
- Templates handle unavailable states gracefully
- Standard animation names: pulseCritical, pulseWarning
```

---

*Last updated: 2026-04-29 — Added GitHub sync workflow, context refresh commands, closing ritual*
