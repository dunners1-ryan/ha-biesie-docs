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
- LAN device reachability (UniFi gateway, APs, ASUS routers)
- WAN internet reachability (DNS ping to OpenDNS, Google)
- WAN quality metrics: latency, jitter, packet loss (approximated)
- ISP speed utilisation (download/upload as % of subscribed max)
- Derived NOC-style status: healthy / degraded / critical / offline

**Topology correction (2026-07-13):** the actual internet-facing WAN router is the
**ASUS ZenWiFi XD6 (192.168.1.3)**, not the ASUS ROG router. The ASUS ROG router
(GT-AX11000, 192.168.1.1) exists solely to provide a dual-LAG (link-aggregated) bonded
LAN connection for the Synology NAS's throughput (see `network_nas.yaml` / NAS bond
notes in Section 10) — it does not route WAN traffic. `sensor.unifi_gateway_*` (UniFi
Dream Machine) sits downstream of the ZenWiFi doing LAN routing/AP control for the
UniFi-managed devices. Both dashboards and `PROJECT_STATE.md` previously mislabeled the
ASUS ROG router as "WAN Router" — corrected across both in this session. See Section 11.

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
group.network_devices            all:true  — UDM + ASUS ROG + ZenWiFi + 5 APs (offline if ANY down)
group.wan_services               all:false — OpenDNS + Google DNS (offline if ANY unreachable)
group.network_device_uptimes     all:false — UDM + ASUS ROG + ZenWiFi uptime sensors
group.network_device_restart_times          — AP + router uptime (dashboard only)
```

#### Members in group.network_devices

| Entity | Source | Notes |
|--------|--------|-------|
| `binary_sensor.unifi_gateway_ping` | ping (UI) | UDM gateway |
| `binary_sensor.asus_rog_router_ping` | ping (UI) | ASUS ROG router — NAS dual-LAG only, not WAN |
| `binary_sensor.zenwifi_xd6_connected` | template → `sensor.zenwifi_xd6_cpu_usage` availability | **ASUS ZenWiFi XD6 — the actual WAN router** (192.168.1.3). Added 2026-07-13 (was IMP-NET04); no discrete connected/disconnected state exists for this device, so availability of a numeric asuswrt sensor is used as the online/offline proxy, matching the dashboard's established pattern. |
| `binary_sensor.ap_bar_connected` | template → `sensor.ap_bar_state` | UniFi controller state |
| `binary_sensor.ap_passage_connected` | template → `sensor.ap_passage_state` | UniFi controller state |
| `binary_sensor.ap_office_connected` | template → `sensor.ap_office_state` | UniFi controller state |
| `binary_sensor.ap_lounge_connected` | template → `sensor.ap_lounge_state` | UniFi controller state |
| `binary_sensor.ap_garage_connected` | template → `sensor.ap_garage_state` | UniFi controller state |

`sensor.zenwifi_xd6_uptime` was also added to `group.network_device_uptimes` (restart
detection) and `group.network_device_restart_times` (dashboard display).

APs migrated from `binary_sensor.ap_*_ping` (ping platform, UI-configured) to UniFi state wrappers on 2026-04-21.
Old ping entries can be removed from Settings → Devices & Services → Ping.

#### Router devices (asuswrt integration)

```
sensor.zenwifi_xd6_*        ASUS ZenWiFi XD6, 192.168.1.3 — the actual WAN router
                             cpu_usage, memory_usage, devices_connected, uptime, last_boot,
                             download/upload (speed). No ping/state/button entity exposed —
                             binary_sensor.zenwifi_xd6_connected (network_helpers.yaml)
                             treats cpu_usage unavailable as the offline proxy, and IS in
                             group.network_devices as of 2026-07-13 (was IMP-NET04).
sensor.asus_rog_router_*     ASUS ROG GT-AX11000, 192.168.1.1 — NAS dual-LAG bond only,
                             not a WAN router. Same asuswrt sensor set, plus
                             binary_sensor.asus_rog_router_ping (in group.network_devices,
                             monitoring the LAG link itself, not WAN).
```

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

### ~~BUG-NET05~~ [MEDIUM] — ✅ Fixed 2026-07-09 — `*_down_list` sensors missed the
BUG-NET04 fix; alert-active sensors had no restart guard
**File:** `packages/alerts/alerts_network.yaml`
**Problem 1:** BUG-NET04's 2026-04-21 fix added `unavailable` handling to the severity
sensor and the `devices` context attribute, but missed `network_devices_down_list` and
`wan_services_down_list` — the sensors whose value is interpolated directly into the
notify message text. Left filtering on literal `state == 'off'` only, so a member reading
`unavailable` during a post-restart integration reconnect wasn't listed, and if the list
sensor itself hadn't recomputed yet when the router read it, the notification showed the
literal word "unknown" instead of a device name.
**Problem 2:** `network_device_down_alert_active` and `wan_down_alert_active` had no
restart guard at all — `group.network_devices`/`group.wan_services` are `all: true`, so a
member reading `unavailable` while UniFi/ping integrations reconnect after an HA restart
reads as group `off`, same as a real outage. If reconnect took longer than the 250s
anti-flap window, this fired a false CRITICAL "device down" alert on every restart.
**Fix applied:** `selectattr('state','in',['off','unavailable'])` extended to the two
down-list sensors (appends " (unavailable)" per affected device name). Added
`is_state('input_boolean.system_startup','off')` to both alert-active sensors — reuses the
existing 2-minute post-restart guard (see INFRA_CONTRACT.md), same pattern as the
2026-07-09 prepaid fix in POWER_CONTRACT.md Issue 18.

### ~~BUG-NET01~~ [MEDIUM] — ✅ Fixed 2026-06-19
`sensor.unifi_cpu_5m_max` availability now reads `has_value('sensor.unifi_gateway_cpu_utilization')`.
The self-referencing `has_value('sensor.unifi_cpu')` has been corrected.

### ~~BUG-NET02~~ [MEDIUM] — ✅ Fixed 2026-06-19
`sensor.unifi_memory_5m_max` availability now reads `has_value('sensor.unifi_gateway_memory_utilization')`.
The circular `has_value('sensor.unifi_memory_5m_max')` has been corrected.

### ~~BUG-NET06~~ [MEDIUM] — ✅ FIXED 2026-07-17
**File:** `packages/alerts/alerts_network.yaml`
**Problem:** `sensor.network_device_down_alert_severity` is a trigger-based template sensor
(`trigger: state` on `binary_sensor.network_device_down_alert_active` and
`group.network_devices` only — no time-based re-evaluation). Live-observed stuck at `critical`
since 2026-07-09T21:14:33 despite `group.network_devices` = `on` and
`binary_sensor.network_device_down_alert_active` = `off` at time of discovery. Most likely cause:
a startup-time evaluation (an AP briefly `unavailable` during an HA restart, pushing the down-count
to ≥2) that never re-fired because `group.network_devices` hasn't changed state since. This feeds
`sensor.network_alert_context` = `critical` with an empty `devices` attribute list — a false
CRITICAL reading with no actual cause, which would show a misleading banner on any dashboard that
reads the severity/context sensors directly instead of the underlying `alert.network_alert` state.
**Interim workaround (dashboard only, not a code fix, still in place):** the `network-control`
dashboard (Section 11) keys its WAN status banner off `sensor.wan_noc_status` /
`sensor.wan_health_score` and its "Active Network Alerts" card off the `alert.network_alert`
domain state instead of the stale severity/context sensors.
**Root-cause fix applied 2026-07-17:** added a `time_pattern: minutes: "/5"` trigger to
`sensor.network_device_down_alert_severity` (same class of fix as BUG-NET04/05) so it
periodically re-evaluates even when its watched entities don't change state. Live-verified the
bug reproducing in real time immediately before the fix: a `template.reload` (itself unrelated —
triggered while investigating a "why won't this clear on reload/restart" report) caused all 5 APs
to blip `unavailable` during the UniFi reconnect, freezing the severity/context sensors at
`critical` again exactly as this entry predicted, while `alert.network_alert` and
`binary_sensor.network_device_down_alert_active` correctly stayed `idle`/`off` throughout —
confirming the diagnosis before applying the fix. Post-fix reload confirmed the new trigger
self-corrects the stale value within one 5-min cycle without needing another reload/restart.
Applied via `template.reload` (no restart needed — no `alert:`/`group:` structure changed).

### ~~BUG-NET07~~ [MEDIUM] — ✅ Fixed 2026-07-13 — WAN Health Score / Packet Loss graphs showed "No statistics found"
**File:** `packages/network/network_helpers.yaml`
**Problem:** `sensor.wan_health_score`, `sensor.wan_cloudflare_packet_loss`,
`sensor.wan_google_packet_loss`, and `sensor.wan_microsoft_packet_loss` are template
sensors with no `state_class` set. HA's recorder only creates long-term statistics (the
data `statistics-graph` cards query) for sensors with a `state_class`; without one, only
raw short-term state history is kept. Confirmed live: the "WAN Health Score (7d)" and "WAN
Packet Loss (7d)" `statistics-graph` cards on `network-debug` showed "No statistics found"
while the adjacent "Gateway CPU Temperature (7d)" card (a native UniFi integration sensor,
which sets its own `state_class`) worked fine — same card type, same section, different
sensor origin.
**Fix applied:** added `state_class: measurement` to all four sensors. Statistics begin
accumulating from the next recorder cycle after reload; HA also runs a one-time missing-
statistics backfill from existing state history on the next recorder start, so some of the
existing 7-day history may repopulate automatically without needing to wait a further 7
days. Same class of fix affects `sensor.wan_health_score`'s use in `network-control`'s
"WAN Health Score (48h)" graph — same underlying sensor, same fix.
**Reload:** Template sensor change — **Reload Template Entities**, no restart required.

### ~~BUG-NET03~~ [HIGH] — ✅ Fixed 2026-06-19
WAN packet loss formula rewritten using ping pass/fail binary count approach.
`wan_*_packet_loss` sensors now use `state_characteristic: count` (successful pings) and
`state_characteristic: sum` (total pings) on binary ping sensors, producing a valid
`(total - successful) / total` loss percentage. `wan_health_score` packet loss
component now contributes real signal.

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
| ~~BUG-NET05~~ | ~~Medium~~ | ~~Down-list sensors missed BUG-NET04 fix; no restart guard on alert-active sensors~~ — **FIXED 2026-07-09** |
| ~~BUG-NET01~~ | ~~Medium~~ | ~~Fix `sensor.unifi_cpu_5m_max` availability~~ — **FIXED 2026-06-19** |
| ~~BUG-NET02~~ | ~~Medium~~ | ~~Fix `sensor.unifi_memory_5m_max` self-referencing availability~~ — **FIXED 2026-06-19** |
| ~~BUG-NET03~~ | ~~High~~ | ~~Fix WAN packet loss formula (currently meaningless)~~ — **FIXED 2026-06-19** |
| ~~BUG-NET06~~ | ~~Medium~~ | ~~`network_device_down_alert_severity` has no periodic re-evaluation trigger — can stick at a stale `critical` indefinitely.~~ — **FIXED 2026-07-17**, see Section 6. |
| IMP-NET01 | Low | Add `sensor.network_alert_context` to `sensor.alert_device_entities` aggregator (verify wired — B3 done 2026-04-14) |
| IMP-NET02 | Low | Add ISP name/plan to a descriptive input_text for context on dashboard |
| IMP-NET03 | Low | Several UniFi diagnostic entities are disabled by the integration by default (`sensor.ap_bar_clients`, `sensor.ap_lounge_clients`, `sensor.ap_passage_clients`, `sensor.usw_ultra_poe_clients`, all per-port PoE switch/link-speed sensors) — inconsistent with `ap_garage`/`ap_office` which have clients enabled. Enable in the UniFi integration entity list if per-AP client counts / per-port PoE control become needed; the network-control LAN table degrades gracefully to "—" for these today. |
| ~~IMP-NET04~~ | ~~Medium~~ | ~~ZenWiFi (actual WAN router) not in any alert group~~ — **FIXED 2026-07-13.** Added `binary_sensor.zenwifi_xd6_connected` (template, keyed off `sensor.zenwifi_xd6_cpu_usage` availability) to `group.network_devices`; `sensor.zenwifi_xd6_uptime` added to `group.network_device_uptimes` and `group.network_device_restart_times`. Applied via Reload Template Entities + Reload Groups (no restart needed), verified live via API. |

---

## Section 11: Dashboards (added 2026-07-10)

Two views on `dashboard_operations` (`.storage/lovelace.dashboard_operations`), same
`sections`/`max_columns: 3` pattern used by power-control/security-control/water-control.
Linked to each other via a breadcrumb chip-card at the top of each view's first section, and
from the Home page's "Network Health" card (chips: Network Control / Network Debug).

### `network-control` (path: `network-control`) — at-a-glance + primary controls
- **WAN section:** NOC-status banner (pulses on degraded/critical/offline), download/upload +
  peak-5m tiles, packet-loss/jitter tiles, per-provider (CF/Google/MS) latency tiles, a
  consolidated markdown status line for the Gateway/ISP plan/WAN services group, two 24–48h
  trend graphs (WAN health score, per-provider latency), and the existing "Active Network
  Alerts" `auto-entities` card.
- **LAN section:** one consolidated markdown table (Gateway/Switch/5×AP/ZenWiFi — status,
  clients, CPU, memory) instead of per-device entity cards, a 24h LAN client-count trend graph,
  a 7-button restart grid (Gateway/Switch/5×AP, each with a tap confirmation), and a
  conditional "Unknown AP Detected" tile (only renders when
  `binary_sensor.unknown_unifi_ap_detected` is on).
- **NAS & UPS Protection section:** carried over from the pre-existing control view (NAS
  shutdown-imminent tile, UPS reserve tile, auto-shutdown controls, manual shutdown/WoL
  buttons) — this section was already in good shape and needed no redesign.

### `network-debug` (path: `network-debug`) — deep-dive, reorganised from the old version
Replaces the old `network-debug` view, which had duplicated WAN latency tiles across two
sections, duplicated AP Garage blocks, and ~10 CPU/memory gauge cards with no way to see trends.
Five sections: **WAN** (quality metrics, live 5-min + 7-day latency/loss graphs, ISP throughput
gauges+graph), **LAN Core** (Gateway/Switch/ASUS ROG router — badges + compact entities + current
gauges + the two graphs that actually have recorder history), **LAN Wireless** (5-day AP
connectivity history-graph, one compact badge-only block per AP + a shared restart grid instead
of 5× duplicated gauge cards, ZenWiFi mesh node, 7-day client-count graph), and two sections that
were previously **not on any dashboard**: **NAS** (Synology Guardians — drive/volume status,
CPU/memory/throughput trend graphs, reboot/shutdown/WoL buttons) and **UPS** (EcoFlow River Pro —
reuses the pre-built `sensor.ups_status_card` / `sensor.ups_load_markdown` display strings,
battery-level and AC in/out power trend graphs, key settings).

**Design constraint that shaped both views:** `configuration.yaml`'s recorder `exclude.entity_globs`
(`sensor.ap_*_cpu_*`, `sensor.asus_*_cpu_*`, `sensor.asus_*_memory_*`, `sensor.*_ping_*`,
`sensor.*_uptime`, `sensor.*_last_boot`) means AP/ASUS CPU, memory, and ping-RTT sensors have
**zero recorder history** — confirmed empirically via `/api/history/period` before building
(0 data points over 2 days vs. tens of thousands for non-excluded sensors). Those are shown as
gauges/badges (current value only) rather than history graphs on both views; this is intentional,
not an oversight — don't "fix" it by adding trend graphs for those entities unless the recorder
excludes are deliberately changed first (which would grow the DB — see CODING_STANDARDS.md).

⚠️ `.storage/lovelace` edits require a full HA restart (not a browser refresh) — see
CODING_STANDARDS.md and the OPEN TODO in PROJECT_STATE.md.

### Post-restart polish pass (same day, 2026-07-10)
After the first restart, visual review turned up one real defect and confirmed two non-issues:

- **Fixed — oversized restart buttons on `network-debug`.** The 5 per-AP restart buttons were
  each a standalone `button`-type card nested inside its own `vertical-stack`, so they got full
  section-grid sizing instead of sharing a row — rendered as five huge cards. Consolidated into
  one shared `columns: 4` grid (`lan_wireless_restart_grid`, same pattern as the working
  `network-control` restart grid) placed once at the top of the LAN Wireless section, covering
  Gateway/Switch/5×AP. `ap_block()` no longer takes a `restart_entity` param.
- **Confirmed — no ASUS ROG router restart control exists.** Checked the device registry: the
  GT-AX11000 (192.168.1.1) is monitored purely via a `ping` binary sensor plus a stats-only
  source (no `asuswrt` integration configured) — **zero** button/switch entities exist for that
  device in HA. This isn't a dashboard gap; there is nothing to wire up. If restart control is
  wanted later, the `asuswrt` integration would need to be added for that device first.
- **Non-issue — "Failed to fetch dynamically imported module" + garbled entity rows.** Both
  symptoms appeared identically across unrelated cards on both dashboards right after the
  restart — a stale frontend JS-chunk cache in the browser tab (chunk hashes changed with the
  HA version bump), not a card config defect. Resolves with a hard refresh; no YAML change
  needed or made for this.
- **Added** a "Total Known Clients" line under the `network-control` LAN table, summing the
  entities that do have a live client-count sensor (Gateway + AP Garage + AP Office + ZenWiFi),
  with an explicit note that Bar/Lounge/Passage/Switch counts are excluded (disabled sensors,
  see IMP-NET03).

⚠️ This polish pass also touched `.storage/lovelace.dashboard_operations` — same restart
requirement applies again before it's live.

### Topology correction pass (2026-07-13)

User flagged that both dashboards (and `PROJECT_STATE.md`) had the WAN router wrong: the
**ASUS ZenWiFi XD6 (192.168.1.3)** is the actual internet-facing WAN router; the **ASUS ROG
router (192.168.1.1)** exists only to provide a dual-LAG bonded LAN connection for the
Synology NAS and does not route WAN traffic. Previously the ZenWiFi appeared only as
"ZenWiFi Mesh" in the LAN wireless table/section, and the ASUS ROG router didn't appear on
`network-control` at all (only as "ASUS ROG Router (Secondary LAN)" on `network-debug`).

- **`network-control`:** Added a "WAN Router (ASUS ZenWiFi XD6 · 192.168.1.3)" status line
  at the top of the WAN markdown block (above the UniFi Gateway line, now labelled
  "downstream LAN routing" instead of "WAN Edge"). LAN table: ZenWiFi row renamed
  "ZenWiFi (WAN Router)" and moved to the top; added a new "ASUS ROG (NAS Dual-LAG)" row
  directly below it (previously absent from this view). LAN Client Count graph updated to
  match (both series renamed/added).
- **`network-debug`:** Added a new "WAN Router — ASUS ZenWiFi XD6" block (clients/CPU/mem
  badges + uptime/last-boot/devices-connected entities) in the WAN section. Relabelled the
  LAN Core heading from "ASUS ROG Router (Secondary LAN)" to "ASUS ROG Router (NAS Dual-LAG
  only — not WAN)". Relabelled the Wireless section's ZenWiFi block heading to note it's
  also the WAN router (kept in place since it does still provide mesh wifi coverage; full
  detail lives in the new WAN-section block to avoid re-introducing the pre-2026-07-10
  duplication this rebuild removed).
- **Not done (flagged, not fixed):** `group.network_devices` still doesn't monitor the
  ZenWiFi — see IMP-NET04, Section 8.
- Both `.storage/lovelace.dashboard_operations` and `.storage/lovelace.operations_debug`
  backed up before editing (`*.bak.20260713_185021`). **⚠️ Requires a full HA restart**
  before either dashboard's changes take effect (see CODING_STANDARDS.md).

---

*Last updated: 2026-07-13 (same day, follow-up) — IMP-NET04 closed: ZenWiFi XD6 now
monitored by the alert pipeline via a new `binary_sensor.zenwifi_xd6_connected` template
sensor in `group.network_devices`, plus `sensor.zenwifi_xd6_uptime` added to both uptime/
restart-tracking groups. Applied live via Reload Template Entities + Reload Groups, no
restart required; verified via API (`group.network_devices` state and `entity_id`
attribute both confirmed to include the new member).*
*Last updated: 2026-07-17 — BUG-NET06 closed: added `time_pattern: minutes: "/5"` trigger to
`sensor.network_device_down_alert_severity` so it self-corrects a stale `critical` reading instead
of needing a state-change event. Reproduced the bug live via a `template.reload` immediately
before applying the fix (confirmed severity/context freezing while `alert.network_alert` stayed
correctly `idle`), then confirmed the fix self-heals within one 5-min cycle post-reload.*
*Last updated: 2026-07-13 — Topology correction: ZenWiFi XD6 (192.168.1.3) identified as the
actual WAN router; ASUS ROG router (192.168.1.1) corrected to its real role (NAS dual-LAG
only). Both dashboards and PROJECT_STATE.md updated; IMP-NET04 added (ZenWiFi not in any
alert group).*
*Last updated: 2026-07-10 — Section 11 (dashboards) added; BUG-NET06 (stale alert severity sensor)
discovered and documented, IMP-NET03 (disabled UniFi diagnostic entities) added.*
*Last updated: 2026-06-19 — BUG-NET01/02/03 closed (CPU/memory availability and packet loss formula all corrected; verified in code)*
*Last updated: 2026-05-28 — network_nas.yaml added; NAS graceful shutdown + WoL restore pipeline documented (Section 10)*
*Source: packages/network/network_helpers.yaml, packages/alerts/alerts_network.yaml, packages/network/network_ups.yaml, packages/network/network_nas.yaml, .storage/lovelace.dashboard_operations*
