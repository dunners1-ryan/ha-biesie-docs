# HABiesie — Claude Code Deep Audit Prompts
> One prompt per domain. Each produces a single living contract document.
> Run these IN ORDER — Security first, then others, as security has most active issues.
> Each audit reads ALL relevant files including UI storage, helpers, and cross-domain deps.

---

## HOW TO USE THESE PROMPTS

1. Open VS Code connected via SSH Remote to `/config/`
2. Open Claude Code chat panel
3. Paste the relevant domain prompt below
4. Claude Code will read all files, produce the contract document
5. Save it to `/config/docs/domains/<DOMAIN>_CONTRACT.md`
6. That file replaces the simple context doc we created earlier — it's the full living contract

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PROMPT 1: SECURITY DOMAIN AUDIT
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```
You are performing a deep audit of the Security domain in my Home Assistant 
configuration at /config/. Your goal is to produce a single comprehensive 
living contract document that captures everything about how this domain 
currently works, what is broken, and what the intended design is.

## STEP 1 — READ CONTEXT FIRST
Read these files before anything else:
- /config/docs/PROJECT_STATE.md
- /config/docs/CODING_STANDARDS.md
- /config/docs/domains/SECURITY_CONTEXT.md

## STEP 2 — READ ALL SECURITY PACKAGE FILES
Read every file in /config/packages/security/ — list them first, then read each one.
For each file document:
- What entities it defines (automations, sensors, helpers, scripts)
- What external entities it consumes (states() calls to other packages)
- What it produces that other packages consume
- Any obvious errors, missing from: constraints, missing defaults

## STEP 3 — READ UI STORAGE (CRITICAL — these define runtime helpers)
Read /config/.storage/core.entity_registry and search for all entities with:
- entity_id containing: security, cam, snapshot, motion, intrusion, arrival
Read /config/.storage/lovelace.* files to find:
- Any input_boolean, input_text, input_datetime, input_number helpers 
  created via the UI (not in YAML) that the security system depends on
- Document ALL of these — they are invisible to package file audits

## STEP 4 — READ AUTOMATIONS.YAML (LEGACY)
Read /config/automations.yaml and find ALL automations related to:
- security, camera, snapshot, motion, arrival, gate, intrusion
Document each one:
- What it does
- What entities it reads/writes
- Whether it duplicates logic in the packages
- Whether it should be migrated to a package file

## STEP 5 — READ CURRENT LOG FOR ACTIVE ERRORS
Read /config/home-assistant.log (last 200 lines)
Find and document all errors/warnings related to:
- security package entities
- camera entities
- snapshot/www/ path issues
- template errors in security sensors

## STEP 6 — ANALYSE SNAPSHOT SYSTEM
Look at /config/www/ directory listing (ls -la /config/www/cam*.jpg | head -50)
Document:
- How many snapshot files exist
- Naming patterns (cam05_* vs security_cam05_* duplication)
- File sizes and ages
- Which automations are creating them and where (grep -r "snapshot" /config/packages/)
- Root cause of duplication

## STEP 7 — MAP CROSS-DOMAIN DEPENDENCIES
Document every place the security system touches other domains:
- What it reads FROM presence (binary_sensor.anyone_connected_home, trust model)
- What it reads FROM context (binary_sensor.night_confirmed, boundary_permissive_window)
- What it reads FROM alerts (what feeds the security alert context)
- What it reads FROM notifications (which scripts it calls)
- What it PROVIDES to lighting (motion sensors lighting responds to)

## STEP 8 — IDENTIFY ALL ACTIVE PROBLEMS
For each problem found, document:
- Symptom (what you observe)
- Root cause (why it happens)  
- Affected entities
- Proposed fix
- Risk level of fix (low/medium/high — would it break other things?)

Known problems to verify and expand:
1. Duplicate snapshots (cam05_* AND security_cam05_* for same event)
2. input_text.security_event_session overflow (255 char limit)
3. Trust model not wired into classification
4. Missing from: constraints on motion triggers
5. Camera feed instability from NVR/go2rtc

## STEP 9 — PRODUCE THE CONTRACT DOCUMENT

Write a single comprehensive markdown document covering:

### Section 1: System Overview
- What the security system is supposed to do
- The intended event flow (camera motion → classify → snapshot → notify)
- Hardware involved and its limitations

### Section 2: File Inventory
- Every file in packages/security/ with one-line purpose
- Every UI-created helper that this domain depends on (from Step 3)
- Every legacy automation in automations.yaml for this domain (from Step 4)

### Section 3: Entity Reference (Complete)
For EVERY entity this domain owns or critically depends on:
- Entity ID
- Type (sensor/binary_sensor/input_boolean/etc)
- Where defined (which file OR "UI helper" OR "automations.yaml")
- Current state/value if you can determine it
- What consumes it
- What produces it

### Section 4: Data Flow Map
ASCII diagram showing the complete flow:
Camera trigger → motion valid → snapshot capture → session update → 
classification → threat score → notification → dashboard

### Section 5: Cross-Domain Interface
Exact list of what this domain exposes as its "public API" to other domains
and what it requires from other domains as inputs.

### Section 6: Known Issues (Prioritised)
Each issue with: symptom, root cause, fix, risk, estimated effort

### Section 7: Active Log Errors
Direct copy of relevant log errors found in Step 5

### Section 8: Optimisation Recommendations
Things that work but could be improved — do NOT mix with broken things

### Section 9: Implementation Checklist
Ordered list of what to fix/build, ready to execute as tasks

### Section 10: Design Decisions (Locked)
Things that must not change and why — naming conventions, 
file locations, entity IDs that dashboards depend on

Save this document as: /config/docs/domains/SECURITY_CONTRACT.md
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PROMPT 2: POWER DOMAIN AUDIT
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```
You are performing a deep audit of the Power/Energy/Prepaid domain in my 
Home Assistant configuration at /config/. Your goal is to produce a single 
comprehensive living contract document.

## STEP 1 — READ CONTEXT FIRST
Read these files before anything else:
- /config/docs/PROJECT_STATE.md
- /config/docs/CODING_STANDARDS.md
- /config/docs/domains/POWER_CONTEXT.md
- /config/docs/domains/POWER_DEPENDENCY_ANALYSIS.md  ← existing analysis to build on

## STEP 2 — READ ALL POWER PACKAGE FILES
Read every file in /config/packages/power/ — list them first, then read each one.
The existing dependency analysis already maps the structure — your job is to:
- Verify it is still accurate
- Add detail on actual entity states and values
- Find any errors or drift from the documented design

## STEP 3 — READ UI STORAGE
Read /config/.storage/core.entity_registry and find all UI-created helpers for:
- prepaid, inverter, battery, solar, grid, load, energy, power
Document ALL input_number, input_boolean, input_datetime created via UI —
these are the manual controls and thresholds the system depends on.

## STEP 4 — READ LEGACY AUTOMATIONS
Read /config/automations.yaml and find automations related to:
- power, inverter, battery, solar, grid, prepaid, load shedding, energy
Document each — does it duplicate package logic? Should it be migrated?

## STEP 5 — READ LOGS
Read /config/home-assistant.log (last 200 lines)
Find all errors related to power package sensors, inverter integration, 
solcast, template errors in power sensors.

## STEP 6 — VERIFY INVERTER INTEGRATION
Check /config/packages/power/power_core.yaml (or equivalent)
Verify the dual inverter setup is correctly mapped:
- Which solarman device_id maps to Master (grid/losses/BMS)
- Which maps to Slave (PV/load/battery)
- Are all aggregation sensors correct?
- Any missing or incorrectly mapped register readings?

## STEP 7 — VERIFY PREPAID MODEL
The existing analysis describes a 3-value alignment model.
Verify in the actual code:
- Is the alignment offset calculation correct?
- Is monthly spend calculation usage-based (not top-up based)?
- Are all template sensors properly defended against unavailable states?
- Is the buy score logic sound?

## STEP 8 — MAP CROSS-DOMAIN DEPENDENCIES
Document everything power provides to other domains:
- To water: battery SOC gate for refill permission
- To alerts: power_health, battery/grid alert contexts
- To lighting: energy saving mode signals
- To presence: orchestrator load decisions
- To security: (any power-based signals?)

## STEP 9 — PRODUCE THE CONTRACT DOCUMENT

Same structure as Security contract (Sections 1-10):

### Section 1: System Overview
### Section 2: File Inventory (packages + UI helpers + legacy automations)
### Section 3: Entity Reference (Complete — all ~80+ entities)
### Section 4: Data Flow Map (hardware → aggregation → strategy → decisions)
### Section 5: Cross-Domain Interface
### Section 6: Known Issues (Prioritised)
### Section 7: Active Log Errors
### Section 8: Optimisation Recommendations
### Section 9: Implementation Checklist
### Section 10: Design Decisions (Locked)

Special sections for Power:
### Section 11: Measurement Model
  - Physical meter vs inverter tracking
  - Drift pattern and reconciliation procedure
  - When to realign and how

### Section 12: Inverter Register Map
  - Master inverter: which registers, what they mean
  - Slave inverter: which registers, what they mean
  - Known firmware quirks

Save as: /config/docs/domains/POWER_CONTRACT.md
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PROMPT 3: WATER DOMAIN AUDIT
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```
You are performing a deep audit of the Water domain in my Home Assistant 
configuration at /config/. CRITICAL: Read the lifecycle contract file 
before anything else — it defines hard rules that cannot be violated.

## STEP 1 — READ CONTEXT FIRST
- /config/docs/PROJECT_STATE.md
- /config/docs/CODING_STANDARDS.md
- /config/docs/domains/WATER_CONTEXT.md
- /config/packages/water/a_water_lifecycle_contract.yaml  ← READ THIS FIRST

## STEP 2 — READ ALL WATER PACKAGE FILES
Read every file in /config/packages/water/
For each file verify against the lifecycle contract:
- Does the capture logic correctly only observe, not control?
- Is safety logic truly independent?
- Do all pump triggers have from: "off" constraints?
- Do all pump triggers have stability windows (for: 00:00:10)?
- Is binary_sensor.water_refill_allowed checked BEFORE pump starts?

## STEP 3 — READ UI STORAGE
Find all UI-created helpers for: water, borehole, pump, tank, refill, depth
These thresholds and control flags are critical to system behaviour.

## STEP 4 — READ LEGACY AUTOMATIONS
Find water-related automations in /config/automations.yaml
Specifically check for:
- Any pump control automations that bypass the permission gate
- Any notification triggers that could cause spam
- Any lifecycle capture automations that could double-log

## STEP 5 — READ LOGS
Find errors related to: water, borehole, pump, tank depth, tuya sensor
Specifically look for:
- Trigger spam (repeated START/END cycles)
- Sensor unavailable transitions causing false triggers
- Safety abort events

## STEP 6 — VERIFY TUYA SENSOR RELIABILITY
Check how sensor.water_tank_depth_validated is defined
Verify the spike rejection logic:
- What counts as a spike?
- Is the validation window correct?
- Are there any templates that use the raw depth instead of validated?

## STEP 7 — VERIFY SAFETY SYSTEM COMPLETENESS
Verify all 5 safety protections are present and correctly implemented:
1. Max depth stop (depth ≥ target_full)
2. Dry run protection (pump ON + power below threshold)
3. No-rise protection (running but depth not increasing)
4. Battery hard stop (SOC drops below hard_stop threshold)
5. Max runtime cutoff (pump on > max_runtime_minutes)

For each: is it truly independent? Can any other logic bypass it?

## STEP 8 — MAP CROSS-DOMAIN DEPENDENCIES
- FROM power: battery SOC gate, grid state gate
- FROM alerts: what triggers water alerts
- FROM notifications: which script handles water events
- FROM context: quiet hours affecting water notifications

## STEP 9 — PRODUCE CONTRACT DOCUMENT
Same 10 sections as Security + Water-specific sections:

### Section 11: Lifecycle Contract Verification
  - Each contract rule, verified against actual code: PASS/FAIL
  - Any contract violations found

### Section 12: Safety System Audit
  - Each protection: verified present, correctly independent, cannot be bypassed

### Section 13: Sensor Reliability Assessment
  - Tuya sensor behaviour characterised
  - Spike rejection effectiveness
  - Recommendation on sensor trustworthiness

Save as: /config/docs/domains/WATER_CONTRACT.md
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PROMPT 4: PRESENCE DOMAIN AUDIT
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```
You are performing a deep audit of the Presence domain in my Home Assistant 
configuration at /config/.

## STEP 1 — READ CONTEXT FIRST
- /config/docs/PROJECT_STATE.md
- /config/docs/CODING_STANDARDS.md
- /config/docs/domains/PRESENCE_CONTEXT.md

## STEP 2 — READ ALL PRESENCE PACKAGE FILES
Read every file in /config/packages/presence/
Document the full confidence scoring model:
- How is room occupancy confidence calculated?
- What inputs feed each room's confidence score?
- What threshold converts confidence to binary occupied state?
- How does AP location feed into room confidence?

## STEP 3 — READ UI STORAGE
Find UI-created helpers for: presence, arrival, departure, trust, maid, 
gardener, contractor, staff, guest, holiday
These are critical — the trust model is heavily UI-helper based.

## STEP 4 — READ LEGACY AUTOMATIONS
Find presence-related automations in automations.yaml:
- Arrival/departure detection
- Trust model scheduling (maid/gardener times)
- Any WiFi/AP-based triggers

## STEP 5 — READ LOGS
Find errors related to: presence, device_tracker, unifi, AP location sensors

## STEP 6 — AUDIT TRUST MODEL DUPLICATION
This is the core known problem. Find and document:
- Every place input_boolean.staff_on_site is referenced
- Every place binary_sensor.staff_on_site is referenced
- Every place binary_sensor.low_trust_present is referenced
- Do they conflict anywhere? Which automations use which?
- Is the security system using EITHER of these currently?

## STEP 7 — AUDIT ARRIVAL DETECTION
Verify the gate + camera + WiFi correlation:
- What is the exact trigger combination for arrival_detected?
- How quickly does it fire after gate opens?
- Does it correctly suppress during departure?
- Is there a cooldown?

## STEP 8 — MAP CROSS-DOMAIN DEPENDENCIES
- What presence provides to security (trust, home/away)
- What presence provides to lighting (room occupancy)
- What presence provides to alerts (unknown AP detection)
- What presence receives from security (camera motion for arrival)

## STEP 9 — PRODUCE CONTRACT DOCUMENT
Same 10 sections + presence-specific:

### Section 11: Trust Model Design
  - Intended single source of truth
  - Current fragmented state
  - Migration path to clean model

### Section 12: Confidence Scoring Model
  - Per-room confidence formula
  - Input weights
  - Threshold calibration

Save as: /config/docs/domains/PRESENCE_CONTRACT.md
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PROMPT 5: ALERTS & NOTIFICATIONS AUDIT
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```
You are performing a deep audit of the Alerts and Notifications domain 
in my Home Assistant configuration at /config/.

## STEP 1 — READ CONTEXT FIRST
- /config/docs/PROJECT_STATE.md
- /config/docs/CODING_STANDARDS.md
- /config/docs/domains/ALERTS_CONTEXT.md

## STEP 2 — READ ALL ALERT AND NOTIFICATION PACKAGE FILES
Read every file in /config/packages/alerts/
Read every file in /config/packages/notifications/

## STEP 3 — AUDIT THE ALERT PIPELINE
The pipeline should be:
alert entity → alert_device_entities → context sensor → global summary

Verify for EACH domain (security/power/water/network/temp/doors):
- Is there a properly defined alert entity?
- Does alert_device_entities correctly include it with severity metadata?
- Does the domain context sensor derive from alert_device_entities?
- Does global_alert_context correctly aggregate all domain contexts?

## STEP 4 — AUDIT NOTIFICATION SCRIPTS
For each script (notify_power_event, notify_water_event, etc.):
- Does it correctly check quiet hours?
- Does it route by severity?
- Does it format output as human-readable (not raw dict)?
- Are all variables defended against undefined/unavailable?

## STEP 5 — FIND ALL DIRECT NOTIFY CALLS (CRITICAL)
Search the ENTIRE config for any direct notify.* service calls 
that bypass the central scripts:
grep -r "service: notify\." /config/packages/ 
grep -r "notify\." /config/automations.yaml
Document every bypass found — these are the architectural violations.

## STEP 6 — AUDIT QUIET HOURS
- When do quiet hours apply?
- Is the logic consistent across all scripts?
- Are critical alerts truly always delivered?
- Is there any domain that ignores quiet hours incorrectly?

## STEP 7 — READ LOGS
Find template errors in alert/notification sensors
Find any notification failures or delivery errors

## STEP 8 — MAP CROSS-DOMAIN DEPENDENCIES
Document what each domain feeds INTO the alert system:
- Security → what alert entities/severity sensors
- Power → what alert entities/severity sensors
- Water → what alert entities/severity sensors
- Network → what alert entities/severity sensors
- Temperature → what device groups feed temp alerts

## STEP 9 — PRODUCE CONTRACT DOCUMENT
Same 10 sections + alerts-specific:

### Section 11: Pipeline Audit Results
  Per domain: PASS/FAIL for each pipeline stage

### Section 12: Direct Notify Violations
  Complete list of bypasses found with migration priority

### Section 13: Quiet Hours Coverage Map
  Which alerts respect quiet hours, which don't, which correctly bypass

Save as: /config/docs/domains/ALERTS_CONTRACT.md
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PROMPT 6: CROSS-DOMAIN SYSTEM AUDIT
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Run this AFTER all domain audits are complete.

```
You are performing a cross-domain system audit of my Home Assistant 
configuration at /config/. All domain contract documents have been 
produced. Your goal is to produce a master system contract that shows 
how everything works together.

## STEP 1 — READ ALL CONTRACT DOCUMENTS
- /config/docs/PROJECT_STATE.md
- /config/docs/domains/SECURITY_CONTRACT.md
- /config/docs/domains/POWER_CONTRACT.md
- /config/docs/domains/WATER_CONTRACT.md
- /config/docs/domains/PRESENCE_CONTRACT.md
- /config/docs/domains/ALERTS_CONTRACT.md

## STEP 2 — BUILD CROSS-DOMAIN DEPENDENCY MAP
From the contract documents, build a complete map of every entity that 
crosses domain boundaries. Format as a matrix:

| Entity | Owned By | Consumed By |
|--------|----------|-------------|
| binary_sensor.anyone_connected_home | presence | security, lighting, alerts |
| ...

## STEP 3 — FIND INTERFACE VIOLATIONS
An interface violation is when Domain A reads an internal implementation 
detail of Domain B instead of consuming its published output.
Find and document all violations.

## STEP 4 — FIND MISSING INTERFACES
Are there places where Domain A should be consuming Domain B's output 
but currently isn't?
Known gap: security not consuming presence trust model.
Find all such gaps.

## STEP 5 — IDENTIFY SYSTEM-WIDE RISKS
What single entity failure would break the most things?
What shared helper is most fragile?
What integration going offline causes the most cascade failures?

## STEP 6 — PRODUCE SYSTEM CONTRACT DOCUMENT

### Section 1: System Architecture Overview
### Section 2: Cross-Domain Dependency Matrix (complete)
### Section 3: Domain Public Interfaces (what each domain exposes)
### Section 4: Interface Violations (what to fix)
### Section 5: Missing Interfaces (what to wire up)
### Section 6: System Risk Assessment
### Section 7: Recommended Execution Order for Fixes
  (which fixes unlock other fixes, which are safe to do independently)

Save as: /config/docs/SYSTEM_CONTRACT.md
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## SUPPLEMENTARY: ChatGPT RECOVERY PROMPTS
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Use these in ChatGPT to extract remaining design thinking from old sessions:

### For Security history:
```
Summarise everything we decided about the HABiesie security system.
Include: the event session data model problem and proposed solution,
the duplicate trigger root cause and fix approach, the trust model 
integration plan, and any specific code we wrote or reviewed.
Format as a technical design document I can hand to another AI.
```

### For Presence history:
```
Summarise everything we decided about the HABiesie presence system.
Include: the confidence scoring model design, the trust model 
consolidation plan (staff_on_site duplication fix), the arrival 
detection correlation logic, and any code written.
Format as a technical design document.
```

### For any domain:
```
Summarise all HABiesie [DOMAIN] decisions, designs, code written, 
problems identified and solutions agreed. Include entity names, 
file locations, and the reasoning behind each decision.
I need enough detail to continue this work in a new AI tool
without losing context. Format as a technical handover document.
```
