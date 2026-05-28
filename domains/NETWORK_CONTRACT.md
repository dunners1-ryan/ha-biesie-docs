##########################################################
# HABiesie — NETWORK CONTRACT
# Domain Audit: Network / WAN Monitoring
# Generated: 2026-04-16
# Updated: 2026-04-21
# Source: packages/network/network_helpers.yaml, packages/alerts/alerts_network.yaml
##########################################################

---

## Section 1: Domain Responsibility

The Network domain monitors the health of the home network infrastructure (LAN) and
internet connection (WAN). It feeds the alert pipeline for connectivity failures and
quality degradation.

**Scope:**
- LAN device reachability (UniFi gateway, APs, ASUS router)
- WAN internet reachability (DNS ping to OpenDNS, Google)
- WAN quality metrics: latency, jitter, packet loss (approximated)
- ISP speed utilisation (download/upload as % of subscribed max)
- Derived NOC-style status: healthy / degraded / critical / offline

**Not in scope:**
- Media server availability → `alerts_media.yaml`
- Temperature of network devices → `alerts_temperature.yaml`
- Presence detection via UniFi AP association → `presence/`

---

## Section 2: File Inventory

| File | Contents |
|------|----------|
| `packages/network/network_helpers.yaml` | WAN/LAN sensors, input_numbers, statistics sensors |
| `packages/network/network_ups.yaml` | EcoFlow River Pro UPS — helpers, templates, automations (added 2026-05-27) |
| `packages/network/network_nas.yaml` | Synology NAS graceful shutdown protection + WoL restore (added 2026-05-28) |
| `packages/alerts/alerts_network.yaml` | Groups, alert binary sensors, alert entity, context sensor |

There is no `*_automations.yaml` for LAN/WAN — notifications are driven by `alert.network_alert`
via the alerts pipeline. No direct notify calls.
UPS and NAS automations call `script.notify_power_event` directly.

---

## Section 3: Entity Reference

### Published Sensors (consumed by dashboard / alerts pipeline)

```
sensor.udm_download_mbps          Mbit/s  — download speed (converted from bytes/s)
sensor.udm_upload_mbps            Mbit/s  — upload speed (converted from bytes/s)
sensor.udm_download_percent       %       — download as % of ISP max
sensor.udm_upload_percent         %       — upload as % of ISP max
sensor.unifi_cpu_5m_max           %       — UniFi gateway CPU (5-min stat) ← BUG-NET01
sensor.unifi_memory_5m_max        %       — UniFi gateway memory         ← BUG-NET02
sensor.wan_cloudflare_packet_loss %       — approx loss (Cloudflare)     ← BUG-NET03
sensor.wan_google_packet_loss     %       — approx loss (Google)          ← BUG-NET03
sensor.wan_microsoft_packet_loss  %       — approx loss (Microsoft)       ← BUG-NET03
sensor.wan_max_5min_latency       ms      — max of CF/GG/MS 5-min peaks
sensor.wan_latency_jitter         ms      — max−min spread across three targets
sensor.wan_health_score           %       — composite: latency + loss + jitter
sensor.wan_noc_status             text    — offline / critical / degraded / healthy
```

### Helpers (input_numbers)

```
input_number.isp_download_max     Mbit/s  — UI-configurable ISP download cap (default 500)
input_number.isp_upload_max       Mbit/s  — UI-configurable ISP upload cap (default 200)
```

### Helpers (input_booleans — per alerts_network.yaml)

```
input_boolean.network_device_down_notify   — toggle LAN device down alerts
input_boolean.wan_down_notify              — toggle WAN offline alerts
input_boolean.wan_degraded_notify          — toggle WAN quality alerts
input_boolean.device_restart_notify        — toggle router/AP restart alerts
```

### Groups

```
group.network_devices            all:true  — UDM + ASUS + 5 APs (offline if ANY down)
group.wan_services               all:false — OpenDNS + Google DNS (offline if ANY unreachable)
group.network_device_uptimes     all:false — UDM + ASUS uptime sensors
group.network_device_restart_times          — AP + router uptime (dashboard only)
```

#### AP members in group.network_devices

| Entity | Source | Notes |
|--------|--------|-------|
| `binary_sensor.unifi_gateway_ping` | ping (UI) | UDM gateway |
| `binary_sensor.asus_rog_router_ping` | ping (UI) | ASUS WAN router |
| `binary_sensor.ap_bar_connected` | template → `sensor.ap_bar_state` | UniFi controller state |
| `binary_sensor.ap_passage_connected` | template → `sensor.ap_passage_state` | UniFi controller state |
| `binary_sensor.ap_office_connected` | template → `sensor.ap_office_state` | UniFi controller state |
| `binary_sensor.ap_lounge_connected` | template → `sensor.ap_lounge_state` | UniFi controller state |
| `binary_sensor.ap_garage_connected` | template → `sensor.ap_garage_state` | UniFi controller state |

APs migrated from `binary_sensor.ap_*_ping` (ping platform, UI-configured) to UniFi state wrappers on 2026-04-21.
Old ping entries can be removed from Settings → Devices & Services → Ping.

### Alert Pipeline (alerts_network.yaml)

```
binary_sensor.network_device_down_alert_active   — LAN device(s) offline > 250s
binary_sensor.wan_down_alert_active              — both WAN DNS targets down > 250s
binary_sensor.wan_degraded_alert_active          — health score <= 30 for 5 min
binary_sensor.device_restart_alert_active        — uptime drop detected > 250s
binary_sensor.network_alert_active              — master OR across above four
sensor.network_alert_context                    — severity: warning/critical + devices list
alert.network_alert                             — STD_Alerts, repeat 60 min, skip_first:true
```

---

## Section 4: WAN Health Score Formula

```
score = 100
if latency_max > 80ms:  score -= (latency_max - 80) / 2
score -= (avg_packet_loss_3_targets * 2)
if jitter > 10ms:       score -= (jitter - 10) * 1.5
score = clamp(0, 100)
```

NOC status mapping:
- `offline` → `binary_sensor.wan_down_alert_active` = on
- `critical` → health_score ≤ 30
- `degraded` → `binary_sensor.wan_degraded_alert_active` = on
- `healthy` → default

---

## Section 5: Integration Dependencies

| Integration | Provides |
|-------------|----------|
| `unifi` (built-in) | `sensor.unifi_dream_machine_*`, `sensor.unifi_gateway_*` latency/CPU/memory; `sensor.ap_*_state/uptime/clients` |
| `ping` (built-in) | `binary_sensor.unifi_gateway_ping`, `binary_sensor.asus_rog_router_ping`, WAN DNS targets — **APs no longer use ping** |
| Statistics platform | 5-min max/sum/count statistics for WAN sensors |

---

## Section 6: Known Bugs

### ~~BUG-NET04~~ [FIXED 2026-04-21] — Alert fires but nothing surfaces in summary when APs are unavailable
**File:** `packages/alerts/alerts_network.yaml`
**Problem:** `sensor.network_device_down_alert_severity` only counted `state == 'off'` members
of `group.network_devices`. When new template binary sensors (`binary_sensor.ap_*_connected`)
are briefly `unavailable` (e.g. after config reload or UniFi integration restart), the group
goes `off` — firing the binary alert — but severity computed `down = 0` and returned `none`.
`sensor.network_alert_context` stayed `normal`, so the summary showed nothing despite the
alert being active. Same gap in the `devices` attribute list.
**Fix applied:** `selectattr('state','in',['off','unavailable'])` in both the severity sensor
and the context devices list. Unavailable devices now show as "Unavailable" in the summary.

### BUG-NET01 [MEDIUM] — `sensor.unifi_cpu_5m_max` availability self-reference
**File:** `packages/network/network_helpers.yaml`
**Problem:** The availability template reads `has_value('sensor.unifi_cpu')` but
the entity is `sensor.unifi_gateway_cpu_utilization`. There is no `sensor.unifi_cpu` —
this availability check always returns false, rendering the sensor unavailable.
**Fix:** Change availability to `{{ has_value('sensor.unifi_gateway_cpu_utilization') }}`
or remove the availability block entirely (state already reads the correct entity).

### BUG-NET02 [MEDIUM] — `sensor.unifi_memory_5m_max` circular availability
**File:** `packages/network/network_helpers.yaml`
**Problem:** The availability template reads `has_value('sensor.unifi_memory_5m_max')`
(itself!) — circular reference, always unavailable until manually cleared.
**Fix:** Change to `{{ has_value('sensor.unifi_gateway_memory_utilization') }}` or
remove the availability block.

### BUG-NET03 [HIGH] — WAN packet loss formula is mathematically wrong
**File:** `packages/network/network_helpers.yaml`
**Problem:** Packet loss is computed as `(count - sum) / count` where `count` =
number of state changes in 5 min and `sum` = sum of latency ms values. This produces
a nonsensical result (e.g. 5 pings × 20ms latency → sum=100, count=5, loss=−19).
The formula was likely copied from a binary pass/fail sensor pattern but applied to
a continuous latency sensor.
**Real impact:** `sensor.wan_health_score` and `sensor.wan_noc_status` work
despite this because the latency score alone is a reasonable quality proxy.
Packet loss component contributes noise rather than signal.
**Fix options:**
  1. Remove packet loss component from health score (simplest)
  2. Replace with dedicated binary ping sensors and use count/available approach
  3. Use `state_characteristic: count` vs `state_characteristic: count` on a
     binary availability sensor (requires separate binary ping entities)

---

## Section 7: Architecture Notes

- The network domain feeds only the alert pipeline. No other domain consumes
  network sensors directly.
- `sensor.wan_noc_status` is a clean abstraction for dashboards — prefer this
  over reading raw health scores on cards.
- ISP max values (`input_number.isp_download_max`, `isp_upload_max`) must be
  updated manually when the ISP plan changes (not auto-discovered).
- Alert anti-flap delay is 250s consistently across all LAN binary sensors.
  WAN degraded uses a `for: minutes: 5` on the numeric_state trigger instead.

---

## Section 9: UPS Monitoring — EcoFlow River Pro (added 2026-05-27)

**Device:** EcoFlow River Pro (RIVER_PRO), via Ecoflow-Cloud integration, area: Network.
All entities prefixed `river_pro_ups_*`.

### Key source entities (already enabled)
```
sensor.river_pro_ups_main_battery_level   %    SOC — drives runtime calculation
sensor.river_pro_ups_ac_in_power          W    AC input (0 = on battery)
sensor.river_pro_ups_ac_out_power         W    AC output (network equipment load)
sensor.river_pro_ups_total_out_power      W    all outputs combined
sensor.river_pro_ups_dc_out_power         W    DC out
sensor.river_pro_ups_type_c_out_power     W    USB-C out
sensor.river_pro_ups_usb_1_out_power      W    USB 1 out
sensor.river_pro_ups_usb_2_out_power      W    USB 2 out
sensor.river_pro_ups_usb_3_out_power      W    USB 3 out
sensor.river_pro_ups_remaining_time       min  EcoFlow native estimate (0 when on grid)
sensor.river_pro_ups_status                    status string (standby/charging/discharging)
switch.river_pro_ups_ac_always_on              auto-restores AC on UPS restart — MUST stay ON
switch.river_pro_ups_ac_enabled                enable/disable AC output
```

### Derived sensors (network_ups.yaml)
```
binary_sensor.ups_on_battery         on = on battery (ac_in < 5W, ac_out > 0)
sensor.ups_runtime_seconds           s    calculated reserve (always, not just on battery)
sensor.ups_runtime_friendly          text "X hours, Y minutes" human readable
sensor.ups_runtime_eta               text HH:MM depletion ETA (-- when on grid)
sensor.ups_status_card               text markdown status string for dashboard card
sensor.ups_runtime_severity          ok | warning | critical | unknown (battery only)
sensor.ups_load_percent              %    ac_out / rated_output_watts
sensor.ups_load_status               ok | warning | critical
sensor.ups_load_markdown             text load breakdown string for dashboard card
sensor.ups_accessories_power         W    sum of USB1/2/3 + TypeC + DC out
sensor.ups_visibility_score          %    accessories / total × 100 (mirrors load_visibility_score pattern)
```

### Configuration helpers (network_ups.yaml)
```
input_number.ups_rated_capacity_wh        Wh   720 default (River Pro rated capacity)
input_number.ups_rated_output_watts       W    600 default (River Pro max AC output)
input_number.ups_load_warning_percent     %    65 default
input_number.ups_load_critical_percent    %    85 default
input_number.ups_battery_warning_minutes  min  60 default
input_number.ups_battery_critical_minutes min  20 default
input_boolean.ups_alerts_notify                enable/disable UPS alert pipeline
```

### Automations (network_ups.yaml)
| ID | Purpose |
|----|---------|
| `ups_enforce_ac_always_on` | Re-enables switch.river_pro_ups_ac_always_on if ever OFF |
| `ups_notify_on_battery` | Warning on grid→battery transition (10s stability) |
| `ups_notify_grid_restored` | Info on battery→grid restore |
| `ups_battery_runtime_warning` | Warning when runtime_severity → warning |
| `ups_battery_runtime_critical` | Critical when runtime_severity → critical |
| `ups_load_warning` | Warning/critical when load_status → warning/critical |

All UPS alerts route through `script.notify_power_event` (same pipeline as inverter).

### Runtime calculation
```
usable_wh = (battery_level / 100) × rated_capacity_wh
runtime_s  = (usable_wh / ac_out_power) × 3600
```
Deliberately flat-rate and pessimistic vs EcoFlow native (which uses battery curve + temperature).
Both shown in `sensor.ups_status_card` when on battery for comparison.

### Load breakdown
AC out = network equipment (unknown by device — no sub-metering yet).
USB/DC = accessories/charging. Total = `total_out_power`.
Add smart plugs to individual devices to build up "known" breakdown.

### Disabled entities — none needed for this implementation
Disabled diagnostics (cell voltages, temperatures, slave battery data) can be enabled
later for battery health monitoring. Not required for runtime/load tracking.

---

## Section 10: NAS Protection — Synology Graceful Shutdown (added 2026-05-28)

**Device:** Synology NAS "Guardians" at 192.168.1.6 (bonded LAN 1+2, Bond 1).
**Shutdown entity:** `button.guardians_shutdown` (Synology DSM integration)
**WoL entity:** `button.wol_synology_nas_00_11_32_ad_af_a5` (WoL integration, MAC 00:11:32:ad:af:a5 — LAN 1 physical NIC)
**Network note:** NAS on 192.168.1.x, HA Pi on 10.10.1.x. WoL broadcast_address set to 192.168.1.255 (cross-subnet broadcast via routing).
**WoL reliability note:** WoL is per physical NIC (LAN 1 / LAN 2), not the bond interface. Magic packet targets LAN 1 NIC MAC directly — works while bond is down (NAS off).

### Shutdown timing (all adjustable via helpers)
```
Warning at 15 min remaining → 5 min grace → NAS shutdown at ~10 min remaining
  → 4 min wait → HA Pi (hassio.host_shutdown) at ~6 min remaining.
NAS needs ~2–3 min to complete. Pi needs ~30 sec.
UniFi APs + Hikvision NVR tolerate hard power loss — no graceful shutdown needed.
```

### Cancel path
Toggle `input_boolean.ups_nas_auto_shutdown_enabled` OFF before grace period expires.
Automation re-arms this flag automatically when grid restores.

### Entities (network_nas.yaml)
```
# Helpers
input_boolean.ups_nas_auto_shutdown_enabled    — master enable/disable (cancel gate)
input_boolean.ups_nas_was_shutdown             — flag: set when graceful shutdown fires; gates WoL on HA restart
input_number.ups_nas_shutdown_warn_minutes     — runtime threshold for Stage 1 warning (default 15 min)
input_number.ups_nas_shutdown_grace_minutes    — grace period between warning and forced shutdown (default 5 min)
input_number.ups_pi_shutdown_wait_minutes      — wait after NAS shutdown before Pi shutdown (default 4 min)

# Derived
binary_sensor.ups_nas_shutdown_imminent        — ON when on battery AND runtime < warn threshold
```

### Automations (network_nas.yaml)
| ID | Purpose |
|----|---------|
| `ups_nas_graceful_shutdown` | 3-stage: warn → wait grace → NAS shutdown → wait → Pi shutdown (mode: single) |
| `ups_nas_rearm_on_grid_restore` | Re-enables auto_shutdown flag when grid restores (if user cancelled) |
| `ups_nas_wol_on_grid_restore` | HA startup trigger: if was_shutdown flag ON → 3 min delay → WoL → confirm online |

### DSM configuration (manual — not HA-managed)
DSM → Control Panel → Hardware & Power → General:
- ✅ Enable WOL on LAN 1 (confirmed enabled 2026-05-28)
- ✅ Enable WOL on LAN 2 (confirmed enabled 2026-05-28)
- ✅ Restart automatically when power supply issue is fixed (safety net for hard power loss)

---

## Section 8: Improvement Backlog

| ID | Priority | Description |
|----|----------|-------------|
| ~~BUG-NET04~~ | ~~High~~ | ~~Alert fires but summary empty when APs unavailable~~ — **FIXED 2026-04-21** |
| BUG-NET01 | Medium | Fix `sensor.unifi_cpu_5m_max` availability |
| BUG-NET02 | Medium | Fix `sensor.unifi_memory_5m_max` self-referencing availability |
| BUG-NET03 | High | Fix WAN packet loss formula (currently meaningless) |
| IMP-NET01 | Low | Add `sensor.network_alert_context` to `sensor.alert_device_entities` aggregator (verify wired — B3 done 2026-04-14) |
| IMP-NET02 | Low | Add ISP name/plan to a descriptive input_text for context on dashboard |

---

*Last updated: 2026-05-28 — network_nas.yaml added; NAS graceful shutdown + WoL restore pipeline documented (Section 10)*
*Source: packages/network/network_helpers.yaml, packages/alerts/alerts_network.yaml, packages/network/network_ups.yaml, packages/network/network_nas.yaml*
