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
| `packages/network/network_helpers.yaml` | All sensors, input_numbers, statistics sensors |
| `packages/alerts/alerts_network.yaml` | Groups, alert binary sensors, alert entity, context sensor |

There is no `*_automations.yaml` for network — notifications are driven by `alert.network_alert`
via the alerts pipeline. No direct notify calls.

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

*Last updated: 2026-04-16*
*Source: packages/network/network_helpers.yaml, packages/alerts/alerts_network.yaml*
