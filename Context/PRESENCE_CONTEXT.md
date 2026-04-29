# HABiesie — Presence Domain Context
> **Living document.** Update after every change to the presence package.  
> Paste alongside `PROJECT_STATE.md` when working on presence.

---

## 🎯 Intended Design

Three-layer model: who is home → who is trusted → what does that mean for security:

```
Layer 1: Presence Engine ──→ home/away/arrival_detected
Layer 2: Trust Model     ──→ trusted/low_trust/unknown
Layer 3: Security Engine ──→ ignore/notify/alert/alarm
```

---

## 📁 Package Files

```
packages/presence/
  presence_helpers.yaml       # person helpers, mode flags, timestamps
  presence_templates.yaml     # AP location, confidence, room occupancy sensors
  presence_state.yaml         # room occupied aggregation, home/away state
  presence_automations.yaml   # arrival/departure resolver, trust scheduling
  presence_core.yaml          # device_tracker and zone config
```

---

## 🔴 Critical Entities (DO NOT RENAME)

### Per-Person AP Location
```
sensor.ryan_ap_location     # room name or 'Away'/'Disconnected'
sensor.vicky_ap_location
sensor.luke_ap_location
sensor.tayla_ap_location
```

### Per-Person Presence Debug
```
device_tracker.ryan_iphone_tracker
device_tracker.vicky_iphone_tracker
device_tracker.luke_iphone_tracker
device_tracker.tayla_iphone_tracker
binary_sensor.ryan_unknown_ap
binary_sensor.vicky_unknown_ap
binary_sensor.luke_unknown_ap
binary_sensor.tayla_unknown_ap
```

### Home Presence State
```
binary_sensor.anyone_connected_home   # ON if any family member is home
binary_sensor.any_room_occupied       # ON if any room has occupancy confidence
sensor.presence_active_rooms          # count of occupied rooms
```

### Room Occupancy (Confidence-based)
```
binary_sensor.garage_occupied         # final output (confidence filtered)
binary_sensor.living_areas_occupied
binary_sensor.bedrooms_occupied
binary_sensor.office_occupied
binary_sensor.bar_occupied

sensor.garage_presence_confidence     # 0-100
sensor.living_areas_presence_confidence
sensor.office_presence_confidence
sensor.bedrooms_presence_confidence
sensor.bar_presence_confidence

binary_sensor.garage_occupied_raw     # AP-only, unfiltered
binary_sensor.living_areas_occupied_raw
binary_sensor.office_occupied_raw
binary_sensor.bedrooms_occupied_raw
binary_sensor.bar_occupied_raw
binary_sensor.garage_motion_raw       # motion sensor raw
binary_sensor.living_areas_motion_raw
binary_sensor.office_motion_raw
binary_sensor.main_bedroom_motion_raw
binary_sensor.bar_motion_raw
```

### Trust Model
```
# Manual toggles (set by user or schedule)
input_boolean.maid_on_site
input_boolean.gardener_on_site
input_boolean.contractor_on_site
input_boolean.staff_on_site_override    # force no-staff mode
input_boolean.guest_mode
input_boolean.holiday_mode
input_boolean.entertaining_mode

# Derived sensors (USE THESE in security logic, not the booleans above)
binary_sensor.staff_on_site             # ON if maid OR gardener is scheduled
binary_sensor.low_trust_present         # ON if staff OR contractor on site
sensor.security_trust_mode              # trusted/low_trust/unknown
```

### Trust Schedules
```
input_datetime.maid_start
input_datetime.maid_end
input_datetime.gardener_start
input_datetime.gardener_end
```

### Arrival/Departure Engine
```
input_boolean.arrival_detected
input_datetime.last_arrival_time
input_datetime.last_departure_time
input_text.last_arrival_person
input_text.last_departure_person
sensor.presence_resolver_arrivals       # count today
sensor.presence_resolver_departures     # count today
input_datetime.last_boundary_event
input_text.presence_debug_last_event
input_boolean.property_entry_event      # gate + motion correlation flag
sensor.presence_arrival_marker          # 1 when arrival detected (for chart)
sensor.presence_departure_marker        # 1 when departure detected (for chart)
```

### Network Sanity
```
binary_sensor.unknown_unifi_ap_detected  # ON if unknown AP on network
sensor.unknown_unifi_ap_connections
sensor.unknown_unifi_ap_details
```

---

## ⚠️ Known Problems

### 1. Duplicate Trust Concepts
- `input_boolean.staff_on_site` (manual) vs `binary_sensor.staff_on_site` (derived)
- These overlap and create confusion about which is authoritative
- **Rule:** Always use the derived `binary_sensor.staff_on_site` in automations — never the boolean directly
- **Fix needed:** Rename or remove `input_boolean.staff_on_site` to eliminate ambiguity

### 2. Security Not Consuming Trust Model
- Classification logic had trust filtering commented out during testing
- Security system currently makes decisions without knowing who is home
- **Fix needed:** Re-enable trust conditions in `security_automations.yaml` classification

### 3. No Clean Override Model
- Can't easily say "ignore staff schedule for today"
- `staff_on_site_override` exists but isn't cleanly integrated
- **Fix needed:** Override should cleanly suppress derived sensor, not fight it

### 4. Arrival Detection Not Feeding Presence Engine
- Gate + camera correlation works for arrival notification
- But arrival confirmation doesn't update person-level presence confidence
- **Fix needed:** Successful arrival should boost `ryan_ap_location` confidence

---

## ✅ What Works Well

- AP-based room location (UniFi roaming triggers room changes)
- Confidence-based occupancy (eliminates false positives from single signal)
- Gate + WiFi arrival correlation
- Time-based staff schedules (maid/gardener)
- Presence chart (ApexCharts multi-series AP + gate + arrival/departure)
- Unknown AP detection (security alert when new device joins)

---

## 📐 Trust Decision Flow (Intended)

```
Motion Detected
      ↓
Is anyone_connected_home?
  No  → Escalate (potential intruder)
  Yes ↓
Is low_trust_present?
  Yes → Allow warning, suppress critical
  No  ↓
Is it a known family member AP?
  Yes → Suppress (family member movement)
  No  → Classify as visitor/intruder
```

**This flow is NOT fully implemented. Security currently skips trust check.**

---

## 🎯 Next Steps (Agreed Priority)

1. **Rename/remove `input_boolean.staff_on_site`** — eliminate duplicate concept
2. **Re-enable trust conditions** in security classification
3. **Override model** — single `input_boolean` that cleanly suspends schedules
4. **Wire arrival confirmation** to presence confidence boost
5. **Standardise derived sensor outputs** for security consumption

---

## 🔗 Dependencies

- **Security:** Consumes `binary_sensor.low_trust_present`, `binary_sensor.anyone_connected_home`, `sensor.security_trust_mode`
- **Lighting:** Consumes `binary_sensor.*_occupied` for presence-aware scenes
- **Alerts:** Presence state feeds door alert severity decisions
- **Context:** `binary_sensor.night_confirmed` affects presence interpretation

---

*Last updated: <!-- DATE -->*  
*Updated by: <!-- CHANGE SUMMARY -->*
