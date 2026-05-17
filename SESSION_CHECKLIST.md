# Session Checklist

> Every session that ships code changes must satisfy this checklist before
> it's considered complete. Reference: CODING_STANDARDS Rule 7.
>
> If ANY item remains unchecked at session close — report **"SESSION INCOMPLETE"**
> and list what's owed. Do not silently close.

---

## At the start of a session

- [ ] Read `PROJECT_STATE.md` (or seed all doc URLs from `FETCH_URLS.md`)
- [ ] Read the contract(s) for the domain(s) being touched
- [ ] If touching hardware (cameras, sensors, alarm zones, network kit),
      grep the live config for ground truth BEFORE designing anything —
      do not trust contract descriptions of hardware state alone

## Before code changes

- [ ] Pre-flight gathered: exact entity names, zone compositions, field
      interfaces of any scripts being called
- [ ] User confirmed pre-flight findings — STOP-and-report happened
- [ ] Any `# VERIFY:` items in the prompt resolved against live state

## During code changes

- [ ] Edits are minimal and reversible (template + automation reloads, not
      core restarts) wherever possible
- [ ] Locked entity names from `PROJECT_STATE.md` are NOT changed casually
- [ ] New entities follow naming conventions (snake_case, descriptive,
      domain prefix where appropriate)
- [ ] No hardcoded thresholds — use `input_number` helpers

## After code changes, before commit

- [ ] `ha core check` passes
- [ ] Required reload(s) done (Template Entities / Automations / Helpers)
- [ ] Functional checks run and reported — actual behaviour verified, not
      just "no errors in log"
- [ ] Edge cases flagged by the user are spot-checked

## At commit time — BOTH REPOS

### `/config/` (private — habiesie config)

- [ ] `./gitupdate.sh "<message including sprint/session ID>"`
- [ ] Message references the change scope (e.g. `security S3: ...`)

### `ha-biesie-docs/` (public — contracts and PROJECT_STATE)

These update in the SAME session as the code commit:

- [ ] `PROJECT_STATE.md` — session log entry (top of log)
- [ ] `PROJECT_STATE.md` — Locked Entity Names (if entities added/renamed/removed)
- [ ] `PROJECT_STATE.md` — Hardware Summary (if hardware changed)
- [ ] Relevant domain contract — entity inventory for new/changed entities
- [ ] Relevant domain contract — bug list for opened/closed bugs
- [ ] `git commit -m "docs: <change> (YYYY-MM-DD)"` + `git push`

## Session close report

Report back to the user:

- [ ] What was changed (code-side)
- [ ] What doc updates were made
- [ ] Functional check results
- [ ] Anything DEFERRED — stated explicitly as "this remains open"
- [ ] If checklist has unchecked items: **"SESSION INCOMPLETE — owed: [list]"**

---

## Common drift patterns to avoid

These patterns have caused doc drift previously:

1. **"It's a small change, I'll skip the doc update"** — small changes compound. Update anyway.
2. **"The contract is wrong but the code is right"** — fix the contract immediately. Future sessions read the contract.
3. **"I'll batch the doc updates later"** — later doesn't happen. Update now.
4. **"The bug was N/A so I don't need to record it"** — record the N/A finding. Future audits will hit the same false positive.
5. **"I renamed an entity but the old name still works as an alias"** — propagate the rename through docs anyway.
6. **"I purged orphan entities — no functional change"** — purges ARE entity changes. Record them.
7. **"The pre-flight showed the code was already correct, nothing to do"** — the verification itself is a finding. Record it.

---

## What "complete docs update" looks like

After a session like 2026-05-17 S2 (three-zone classifier):

**`PROJECT_STATE.md` session log:**
```
*2026-05-17 (security S2): Three-zone inside-house classifier built.
New sensors: security_inside_garage_motion / _main_motion / _bedrooms_motion.
Arming gates: inside_garage_armed / _main_armed / _bedrooms_armed.
Classifier sensor.security_event_classification rewritten — 9 output states.
New presence sensors: family_arriving / family_departing / all_family_home.
BUG-S14/S15/S16/S17 closed. BUG-P13 closed.*
```

**`PROJECT_STATE.md` Locked Entity Names:** 12 new entities added.

**`SECURITY_CONTRACT.md` Section 11:** Status: Design → Implemented.

**`SECURITY_CONTRACT.md` bug list:** BUG-S14/15/16/17 → CLOSED 2026-05-17.

That is a complete update. If a session ships code touching 12 entities and
the docs only mention 4 — that session is INCOMPLETE.

---

*Created: 2026-05-17*
*Reference: CODING_STANDARDS Rule 7*
