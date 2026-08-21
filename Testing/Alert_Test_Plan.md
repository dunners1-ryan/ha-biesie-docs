# HABiesie — Alert System Test Plan
> Created: 2026-04-14
> Update after each test run with results and any fixes applied.
> Run this after any alert pipeline changes to verify end-to-end behaviour.
>
> **⚠️ 2026-08-21: this plan has never actually been run** — the Test Run Log below has
> exactly one entry, from creation day, with no Pass/Fail results filled in anywhere in
> the document. Meanwhile the alert pipeline changed substantially in the 4+ months
> since (BUG-A08 through BUG-A19, BUG-N15 through BUG-N18 — STD_Alerts breaking and
> getting fixed twice, Cancel Alert buttons added to every domain, delivery moving from
> `alert:` entity notifiers to `route_*_alert` automations). Two concrete stale claims
> found and fixed in a docs-only pass this session (temperature bypass claimed still
> pending — fixed since 2026-06-19; a deprecated camera referenced in a known-issue row).
> Treat every other expected-result cell as **unverified against current delivery
> mechanics**, not confirmed passing — re-read ALERTS_CONTRACT.md's current Domain
> Pipeline Audit (Section 4) before trusting a specific check in this plan, especially
> Test 6 (Security) and Test 7 (Quiet Hours), which test mechanisms that have since
> changed. This plan is due a real run, not just a doc-drift correction.

---

## 🎯 Test Objectives

1. Verify each alert domain fires correctly and appears in global context
2. Verify dashboard alert widget shows correct severity and device details
3. Verify notifications deliver to correct channels (mobile + Telegram)
4. Verify quiet hours suppress info-level correctly
5. Verify alerts clear correctly when condition resolves
6. Verify no duplicate notifications

---

## 📋 Pre-Test Checklist

Before running any test:
- [ ] HA fully loaded — no startup errors in logs
- [ ] `sensor.global_alert_context` = normal (no active alerts)
- [ ] `sensor.total_active_alerts` = 0
- [ ] Notifications enabled: `input_boolean.notifications_enabled` = ON
- [ ] Quiet hours OFF: `input_boolean.notifications_quiet_hours` = OFF
- [ ] Note current time — alerts dashboard shows last-changed timestamps

---

## 🌡️ TEST 1 — Temperature Alerts

**Easiest test — use threshold manipulation**

### Method
```
Developer Tools → States → input_number.device_temp_high_trigger
Set to: 30
Wait 10–15 seconds
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.device_temp_alert_active` | ON | |
| `sensor.device_temp_alert_context` | warning or critical | |
| `sensor.global_alert_context` | warning or critical | |
| Alert widget shows device name | e.g. "Asus ROG Router" | |
| Mobile notification received | ✅ | |
| Telegram notification received | ✅ | |
| Alert clears after threshold raised | ✅ | |

### Restore
```
Set input_number.device_temp_high_trigger back to 55
```

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## 🔒 TEST 2 — Door/Gate Alerts

**Trigger: Open main gate or leave it open past escalation threshold**

### Method A — Physical test
```
Open main gate
Wait > input_number.door_warning_escalation_minutes (default 5 min)
```

### Method B — Threshold manipulation
```
Developer Tools → States → input_number.door_warning_escalation_minutes
Set to: 0
Open/close main gate sensor via Developer Tools if no physical access
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.door_alert_active` | ON | |
| `sensor.door_alert_context` | warning → critical after escalation | |
| Alert widget shows gate name + duration | ✅ | |
| Mobile notification | ✅ | |
| Escalation notification after repeat interval | ✅ | |
| Alert clears when gate closed | ✅ | |

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## 💧 TEST 3 — Water Alerts

**Two test paths — threshold and state override**

### Method A — Lower depth threshold
```
Developer Tools → States → input_number.water_depth_critical
Set to: 2.0 (above current tank level — forces critical state)
Wait 10–15 seconds
```

### Method B — Force water_state directly
```
Note: sensor.water_state is a template sensor — cannot force it directly.
Use Method A or lower input_number.water_depth_low to trigger low state first.
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.water_alert_active` | ON | |
| `sensor.water_alert_context` | warning or critical | |
| `sensor.global_alert_context` | reflects water | |
| Alert widget shows tank level + depth | ✅ | |
| Mobile notification via script.notify_water_event | ✅ | |
| Water alert appears in active alerts list | ✅ | |
| Alert clears when threshold restored | ✅ | |

### Safety note
```
Do NOT test by actually running the tank dry.
Always restore thresholds immediately after testing.
```

### Restore
```
Set input_number.water_depth_critical back to correct value
Check WATER_CONTRACT.md for correct threshold values
```

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## ⚡ TEST 4 — Power Alerts

### TEST 4A — Grid Offline

### Method
```
Developer Tools → States
Find: group.inverter_grid
Override state to: off
Wait > input_number.grid_off_delay_trigger (default 20s)
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.power_grid_offline_alert_active` | ON | |
| `sensor.power_alert_context` | critical | |
| Alert widget shows grid offline | ✅ | |
| Mobile + Telegram notification | ✅ | |
| Battery runtime visible in alert details | ✅ | |

### Restore
```
Remove override on group.inverter_grid (set back to on)
```

---

### TEST 4B — Battery Low

### Method
```
Developer Tools → States → input_number.inverter_battery_soc_warning_trigger
Set to: 100 (forces battery always below threshold)
Wait 10–15 seconds
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.power_battery_low_alert_active` | ON | |
| `sensor.power_alert_context` | warning or critical | |
| Battery appears in alert devices list | ✅ — verified BUG-A07 fix | |
| SOC value shows in device detail | ✅ | |

### Restore
```
Set input_number.inverter_battery_soc_warning_trigger back to 30
```

---

### TEST 4C — Prepaid Drift

### Method
```
Developer Tools → States → sensor.prepaid_drift_percentage
Note: template sensor, cannot force directly.
Alternative: set input_number.prepaid_alignment_offset to extreme value
to cause drift, then run script.prepaid_realign_offset to restore.
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.prepaid_meter_drift_alert_active` | ON | |
| Prepaid appears in power alert devices list | ✅ — verified BUG-A07 fix | |
| Drift percentage shows in value field | ✅ | |

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## 🌐 TEST 5 — Network Alerts

### Method A — Lower uptime trigger
```
Developer Tools → States → input_number.network_uptime_trigger
Set to: 1
Restart any monitored network device OR wait for next ping cycle
```

### Method B — Disconnect a monitored device briefly
```
Power cycle a switch or access point briefly
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.network_alert_active` | ON | |
| `sensor.network_alert_context` | warning or critical | |
| **Real-time update** — no 60s lag | ✅ — verified BUG-A05 fix | |
| `sensor.global_alert_context` updates immediately | ✅ | |
| Mobile notification | ✅ | |

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## 🛡️ TEST 6 — Security Alerts

### Method A — Lower threat score threshold
```
Trigger any rear camera motion (wave hand, walk past)
Verify sensor.security_threat_level reaches "warning"
```

### Method B — Force threat level
```
Developer Tools → Template → test template:
{{ states('sensor.security_threat_level') }}
Then manually trigger motion on a camera
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| `binary_sensor.security_alert_active` | ON during threat | |
| `sensor.security_alert_context` | warning or critical | |
| Alert widget shows classification + camera | ✅ | |
| `skip_first: true` means NO duplicate with security_automations notification | ✅ | |
| Escalation reminder after 5 min if threat persists | ✅ | |
| Alert clears after motion stops + delay_off (30s) | ✅ | |

### False Positive Status (Dogs)
```
Current debounce settings:
- Rear cameras: delay_on 2s, delay_off 45s
- Adjustment needed: increase delay_on if dogs still triggering

To tune: increase delay_on for cam09/cam10/cam11 to 5s
This means motion must persist 5s before valid trigger
Dogs typically trigger 1-3s bursts
```

### Notes / Result
```
Date tested:
Result:
Dog false positive rate:
Adjustment made:
```

---

## 🔔 TEST 7 — Quiet Hours Suppression

### Method
```
1. Enable quiet hours: input_boolean.notifications_quiet_hours = ON
2. Trigger an INFO-level alert (lower temp threshold to 30°C)
3. Verify NO mobile notification received
4. Verify NO Telegram message received
5. Trigger a WARNING-level alert (lower battery trigger to 100%)
6. Verify warning notification IS received despite quiet hours
7. Restore all thresholds and disable quiet hours
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| Info alert fires → no notification | ✅ suppressed | |
| Warning alert fires → notification received | ✅ bypasses quiet hours | |
| Critical alert fires → notification received | ✅ always delivered | |
| Alert still visible in dashboard during quiet hours | ✅ — only notifications suppressed | |

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## 📊 TEST 8 — Global Context Aggregation

**Verify global_alert_context reflects highest severity across all domains**

### Method
```
1. Trigger info-level alert only → global = info
2. Add warning-level alert → global = warning  
3. Add critical-level alert → global = critical
4. Resolve critical → global drops back to warning
5. Resolve warning → global drops back to info
6. Resolve all → global = normal
```

### Expected Results
| Check | Expected | Pass/Fail |
|---|---|---|
| Global context = highest active severity | ✅ | |
| Dashboard header card updates in real-time | ✅ | |
| Alert counts (critical/warning/info) correct | ✅ | |
| Acknowledged alerts show in acknowledged section | ✅ | |
| Acknowledging doesn't clear the underlying alert | ✅ | |

### Notes / Result
```
Date tested:
Result:
Issues found:
```

---

## 🐛 Known Issues / Accepted Behaviours

| Issue | Status | Notes |
|---|---|---|
| Dogs triggering rear cameras | 🟡 Ongoing (re-verify — not checked in this pass) | delay_on tuning in progress — hardware fix (NVR zones) preferred. SECURITY_CONTRACT.md's active bug catalog is the current source for camera false-positive status, not this row. |
| ~~cam01/cam04/cam07 no snapshots~~ | ✅ Moot, doc-drift correction 2026-08-21 | `cam01` was deprecated 2026-05-08 (replaced by ipcam01+ipcam02) — no longer part of the active camera fleet at all, so a snapshot gap referencing it can't recur. cam04/cam07 status not re-checked this pass. |
| Water alert fires at 10% tank level | ✅ By design (threshold recalibrated since — see below) | Threshold is `input_number.water_depth_critical`, now 0.25m default (≈12.8% of the 1.95m max depth, per WATER_CONTRACT.md's 2026-07-17 dashboard recalibration) — the "10%" figure here is a rough approximation, not exact, but the by-design status still holds. |
| Security alert skip_first = true | ✅ By design (delivery mechanism has changed since, see note below) | Avoids duplicate with security_automations immediate notify. **Note 2026-08-21:** `alert.security_alert`'s own `notifiers:` was removed 2026-07-10 (BUG-A10) — repeats now deliver via `automation.security_alert_repeat_reminder`, and a Cancel Alert button was added 2026-08-18 (BUG-A19). This test's specific expected-results checklist (Test 6) predates both changes and should be re-read against ALERTS_CONTRACT.md's current Security Domain section before re-running. |
| ~~Temperature alerts bypass notify script~~ | ✅ FIXED — doc-drift correction 2026-08-21 | BUG-A03 fixed 2026-06-19 (confirmed again live during today's ALERTS_CONTRACT.md sweep) — temperature routing automations call `script.notify_system_event`, not `notify.STD_*`. No longer "pending" or "on the Group C fix list" (that list is itself fully done, see PROJECT_STATE.md Group C). |

---

## 📅 Test Run Log

| Date | Tests Run | Pass | Fail | Notes |
|---|---|---|---|---|
| 2026-04-14 | Pipeline implementation | — | — | BUG-A01/02/04/05/07 fixed |
| | | | | |

---

*Last updated: 2026-04-14*
*Update after each test session with results*
