# HABiesie — Master Project State
> **Paste at the start of every new Claude session.**
> Last full audit: 2026-04-13 (Claude Code deep audit — all domains verified against live files)
> Update after every meaningful change. Commit: `./gitupdate.sh "docs: update PROJECT_STATE"`

## ⚠️ OPEN TODO

- [x] **2026-09-03 (evening) — Debug dashboard energy cards: fixed cumulative-
      looking Monthly Production / Energy Use Over Time / Appliance Energy by
      Day, an orphaned-entity Grid Import bug, and a zoom-button snap-back +
      y-axis over-scale bug on 4 `custom:plotly-graph` cards. Dashboard-only
      session — no package YAML changed; all edits in `.storage/lovelace.
      operations_debug` (gitignored, so not git-committed itself, only
      documented here).** User: "seems top [Monthly Production] one is
      adding delta per month for cumulative and bottom one looks more
      right?" then flagged the same pattern on Energy Use Over Time and
      Appliance Energy by Day, then asked for 60d/90d/14d range buttons,
      then reported the buttons reverting and a mis-scaled y-axis.
      **Root cause 1 — `statistic:"sum"` misuse:** the broken cards pointed
      `statistic:"sum"` at sensors that reset (`_today` daily counters, or
      lifetime `total_increasing` appliance integrals) with `period:"month"`/
      `"day"` — this card's `"sum"` statistic returns the raw cumulative
      long-term-statistics column, not a period delta, so values only ever
      grew. Fixed: repointed at sensors that themselves reset on the target
      cadence (`solar_production_monthly`/`house_load_energy_monthly`/etc.
      utility meters for month; the 8 appliance plugs' UI-created `_energy_
      day` siblings, found in `core.entity_registry`, for day) with
      `statistic:"state"` instead — deleted the broken duplicate "Monthly
      Production" card outright rather than patch it, since a correct
      sibling card already existed right next to it.
      **Root cause 2 — orphaned entity:** the "Energy Balance" card's Grid
      Import series read `sensor.inverter_today_energy_import`, which
      POWER_CONTRACT Issue 7 documents as deleted 2026-08-21 (dead code, its
      "zero consumers" repo grep was scoped to `packages/` and missed this
      dashboard card, since `.storage/lovelace*` is gitignored). Repointed
      to `sensor.grid_energy_import_today`. POWER_CONTRACT.md updated: Issue
      7 addendum, its two other stale same-fact references (Known
      Aggregation Workaround section + Entity Reference table) corrected,
      new dashboard "Gotchas" subsection added.
      **Root cause 3 — zoom buttons reverting + y-axis stuck too high:**
      root-caused by reading the vendored card's own minified source
      (`www/community/lovelace-plotly-graph-card`), not guesswork — its
      layout-merge function applies `config.layout` twice, last, so any
      static `layout.xaxis.range` in card config permanently overrides
      live pan/zoom on every render (`uirevision` doesn't help — it only
      guards against an *omitted* value, not one deliberately re-supplied
      every time). Fixed by removing `xaxis.range` from all 3 affected
      cards' `layout` entirely; trade-off is the default view width is
      now whatever `hours_to_show` is (90d) rather than a fixed 7d — no
      config-level way found yet to get both. y-axis-stuck-too-high on the
      same cards was `defaults.yaxes.fixedrange:true` locking the axis to
      the *full* fetched dataset's max at first render instead of the
      visible window; fixed with an explicit `layout.yaxis.fixedrange:
      false` override.
      **Also added:** 60d/90d buttons to Energy Flow Daily + Daily Totals;
      14d to Energy Use Over Time + Appliance Energy by Day (was 90d/30d/7d
      only on both); bumped `hours_to_show` on Energy Use Over Time from
      31d to 2160 (90d) so its pre-existing 90d button actually has data.
      **Verification:** each round applied via direct `.storage` JSON edit
      (validated + backed up locally before/after every change) then `ha
      core restart` (plain restart, not stop→edit→start — worked fine,
      confirmed via `ha core logs` showing no lovelace/plotly errors and
      re-reading the file post-restart to confirm edits survived, each
      round). Final round's fix (zoom persistence + y-axis + 14d buttons)
      was pending the user's live click-through confirmation as of this
      entry — visual/interactive behavior wasn't independently re-verified
      beyond source-code analysis and absence-of-error-in-logs.
- [x] **2026-09-03 (later) — Vacuum: dashboard multiline fix + disk-backed
      restart-survival for the tracker averages. Real pyscript dead end hit
      and documented; landed on `shell_command` + script instead, proven
      live with a full corrupt→restore round-trip. A SECOND, still-
      unexplained reset event happened mid-session, independent of the
      first — ruled out `input_number.reload`, `input_boolean.reload`, AND
      the new restore automation itself as causes.** User: "make sure the
      markup cards are multiline as cutting off some details above the
      buttons; fix the restart survive as well."
      **Multiline fix**: all 7 `custom:mushroom-template-card` tiles on the
      Operations → Vacuum view (status tile, Today, Lifetime, Clean Water,
      Dirty Water, Detergent, Manual Clean) were missing `multiline_
      secondary: true` — mushroom cards truncate multi-line secondary text
      to one line with an ellipsis by default. Fixed by walking the live
      dashboard config and adding it to every tile whose secondary content
      has a newline or exceeds ~40 chars — all 7 qualified. Verified saved.
      **Restart-survival — the pyscript route, tried and abandoned (kept
      as a documented dead end, not silently dropped)**: built `pyscript/
      vacuum_tracker_persist.py` to persist the 10 fields that matter
      (6 EMA averages, 3 `_logged_once` guards, the detergent counter) to a
      JSON file on save + restore on `@time_trigger("startup")`. Hit three
      real, reproducible pyscript bugs in sequence, each confirmed live via
      `system_log/list` tracebacks, not guessed: (1) `dict.items()` on a
      local var got mis-parsed as a state entity lookup
      (`state.get('data.items')`); (2) a variable whose only assignment is
      inside a `try:` block isn't visible after it even when the syntax
      itself is fine; (3) fatal — **`open()` is not a defined builtin in
      this pyscript sandbox at all** (`NameError: name 'open' is not
      defined`), confirmed after fixing (1) and (2) got far enough to reach
      the actual file call. No existing pyscript file in this repo does
      file I/O — now clear why: it's not possible here. Deleted the file
      rather than leave a permanently-erroring `@time_trigger("startup")`
      hook in place.
      **What actually worked — this repo's own proven pattern**:
      `packages/integrations/vacuum_tracker_save.sh` (mirrors `gitupdate.
      sh`'s shell-script style) + `shell_command.vacuum_tracker_save`
      (Jinja-templated `states(...)` calls as plain positional args — all
      10 values are space-free tokens, floats or on/off strings, so no
      quoting risk at all) + a `command_line: sensor:` that reads the
      written JSON back via `json_attributes` + a `homeassistant: event:
      start` automation that waits for the sensor to populate then force-
      applies each value via `input_number.set_value` / `choose:`-gated
      `input_boolean.turn_on/off`. State file
      (`packages/integrations/vacuum_tracker_state.json`) deliberately
      lives inside `packages/integrations/`, not a temp/hidden path — same
      precedent as `watercooler_invoice_history.json` — so it rides the
      normal daily `gitupdate.sh` backup and gets real git history.
      Deliberately does NOT persist the 4 `last_*_time` / 3 `area_at_last_*`
      snapshot fields — those self-correct on the very next real press
      regardless of a restart, only the averages take multiple presses to
      re-converge, not worth the extra complexity.
      **Proven live, not just deployed**: set `avg_days_per_water_refill`
      to a distinctive 2.71, saved it via the real `shell_command` service,
      force-updated the `command_line` sensor and confirmed it read 2.71
      back, then "corrupted" the live value to 9.99, called `automation.
      trigger` on the restore automation directly (bypasses the real
      startup trigger, runs the identical action sequence) — value came
      back to 2.71, confirmed via a background wait-loop, not assumed.
      Cleaned up the test value afterward and re-saved the file with the
      genuine current state.
      **Found along the way, not fixed — a second mystery reset**: while
      testing, all three `_logged_once` flags + the detergent counter
      flipped back to their defaults again, at 2026-09-03T14:52:05 UTC —
      independent of the original 2026-09-02T20:07:48 incident. Checked
      `automation.vacuum_tracker_restore_on_startup`'s own `last_triggered`
      first (`None` — hadn't fired, ruled out immediately) and directly
      tested `input_boolean.reload` the same way `input_number.reload` was
      proven safe (set a boolean non-default, reload, value survived) —
      also ruled out. **Root cause of either reset event is still not
      identified.** The disk-backed persistence above doesn't depend on
      knowing why — it just needs the file to be current at save time and
      read on the next real HA restart, which is now proven to work.

- [x] **2026-09-03 — Vacuum: real incident found + fixed — restart/reload
      wiped the water/dirty-water/manual-clean trackers' learned averages
      back to day-one seeds, discarding real learning. New "pre-emptive log"
      capability built so this class of problem (an early/non-representative
      top-up skewing the average) can't recur even when the guard flags are
      healthy.** User: "Just did a pre-emptive refil of water and empty of
      dirty water - both were about half full - please update records so
      doesn't mess with timing needed to fill." Investigated before touching
      anything, since the buttons had already been pressed (13:55:50/:55
      SAST) by the time the message arrived. **Root cause traced precisely**
      via entity `last_changed` timestamps (not guessed): `input_boolean.
      vacuum_water_refill_logged_once` / `_dirty_empty_logged_once` /
      `_manual_clean_logged_once` all show `last_changed` matching either
      today's real press or an earlier 2026-09-02T20:07:48 UTC event — and
      `input_number.vacuum_avg_days_per_water_refill` /
      `_avg_area_per_water_refill` / `_avg_days_per_dirty_empty` /
      `_avg_area_per_dirty_empty` / `_avg_area_per_manual_clean` (plus, as a
      negative control, `vacuum_lifespan_warning_threshold` — a value
      nothing ever writes to) ALL share that exact same 20:07:48 timestamp.
      Conclusion: some other session's helper reload/restart around that
      time reset all three `_logged_once` guards to their YAML `initial:
      false`, wiping the averages back to their seeds (190/1.0 m²/d water,
      95/0.5 m²/d dirty, 300 m² manual-clean) — a genuinely learned value
      (115 m²/refill, verified live earlier this week) is gone. **Net effect
      on today's specific event: harmless by coincidence, not by design** —
      because the guards were freshly reset, today's real press was treated
      as "the very first press ever," which already skips the EMA update
      by design (no interval to compute against a placeholder timestamp).
      The snapshot half (last_refill/last_empty = now, area reset to
      current total) was already correct regardless. **So no data
      correction was actually needed for today's event** — but the
      underlying gap (this happens on every future restart, silently) is
      real and now has an actual fix, not just a lucky non-issue.
      **Built**: `input_boolean.vacuum_log_preemptive` (off by default) —
      flip on before a non-representative early top-up/clean, press the
      relevant log button(s), the snapshot still updates but the EMA step
      is skipped; auto-resets 5 min after being switched on (a dedicated
      timeout automation, NOT a per-button auto-consume — deliberately
      avoided a real race condition caught during design: today's actual
      event needed BOTH the refill AND dirty-empty buttons pressed under
      one pre-emptive session, and a per-automation "turn myself off after
      I fire" approach would let the first automation silently disarm
      protection for the second). Extended to all three trackers (water,
      dirty-water, manual-clean) for consistency, not just the two named.
      Added to the Operations → Vacuum dashboard next to the log buttons.
      Config valid, all helpers/automations reloaded live and confirmed
      (`automation.vacuum_pre_emptive_toggle_auto_reset` = on). **Standing
      gap not fixed, only documented** — WHY the restart/reload didn't
      preserve these YAML-defined helpers' values (input_number.reload is
      separately documented elsewhere in this file as preserving current
      values) is unresolved; recommend spot-checking these six values
      after any future HA restart rather than assuming they survived.

- [x] **2026-09-02 (gas, /update-docs sweep) — Fixed `sensor.gas_alert_
      context`'s own identical miss on the `duration` attribute** (see the
      entry directly below — the two Gas Bottles sessions today independently
      made the exact same mistake Water Cooler's session made, same day,
      same repo). Found via this session's own `/update-docs` step 3a sweep
      reading the concurrent Water Cooler session's diff, which had already
      flagged it: "The Gas Bottles subsystem (Section 8) independently made
      the identical miss ... same day." **This is the real cause of the
      `alert_device_entities`/`problem_alert_devices` "duration" attribute
      error this session's OWN earlier entries (two of them, both above)
      wrongly called "pre-existing, unrelated"** — corrected in place rather
      than silently rewritten, see UTILITIES_CONTRACT.md's Session Log for
      the full fix + the two correction notes. Confirmed the errors actually
      stopped in the live log at the exact moment the fix + reload landed.
      Also fixed two genuine, unrelated-to-gas doc gaps found during the
      same sweep: `UTILITIES_CONTRACT.md` had **zero row** in this file's
      Document Index table despite existing since 2026-08-31 (same "missing
      entirely" pattern GARDEN_CONTRACT.md and SMART_CLEANING_CONTRACT.md
      each had before being caught); and `CLAUDE.md`'s `utilities/` package
      table row still said "(4 files)" and described Gas Bottles as a
      hypothetical future addition rather than the shipped subsystem it now
      is — both fixed.

- [x] **2026-09-02 (water, final round) — Fixed a real recurring log error
      (`sensor.watercooler_alert_context` missing the `duration` attribute
      ALERTS_CONTRACT.md Section 3's own canonical pipeline already required
      — `{domain, devices, duration}` — not a doc gap, a build miss; the Gas
      Bottles session independently made the identical miss on
      `gas_alert_context` the same day, confirmed via the aggregator's own
      "no duration attr" error in the live log). Added the same `duration:`
      calc `deebot_alert_context` uses (minutes since `binary_sensor.
      watercooler_low_stock` last turned on); reloaded, confirmed the error
      stopped recurring. **Open gap, not fixed here** (shared file, not
      touched without asking): neither `alert.watercooler_alert` nor `alert.
      gas_alert` are in `alerts_summary.yaml`'s explicit aggregator trigger
      list — both only get picked up by its 1-minute poll fallback, not
      instantly on state change, unlike every other domain's alert.
      **Chart rebuild** (user: "still squashed, no y-axis, no bar values,
      heading gone — look at gas dashboard"): compared directly against
      `gas-control`'s already-working `Monthly Cost`/`Monthly Refills`
      cards and adopted that exact pattern — native `{"type": "heading"}`
      card (not `markdown`+`card_mod`, which doesn't act as a real section
      header inside a `sections`-layout view, the actual bug behind "heading
      gone"), value labels always shown per bar (was capped to n≤8, dropped
      the cap), y-axis rounded to clean numbers (R1000/R0, not raw
      R990.15-ish), non-rotated horizontal month labels, slot-floor width
      math so a 3-bar view isn't taller than it is wide (the real cause of
      "squashed" — `preserveAspectRatio="none"` from an earlier fix was
      independently scaling width/height and visibly distorting the rotated
      labels; removed). Own bug caught mid-build: first patch script's
      `break` exited before ever reaching the chart cards later in the same
      list, so a content update silently didn't apply — caught by re-
      checking via `/api/template` before restarting, not assumed. Verified
      both final templates render correctly via `/api/template` before each
      restart. `input_select.watercooler_chart_range` (3/6/12 months/
      Lifetime) confirmed already working live in the browser mid-session
      (found set to "1 Year" — the user had already used it). Two restarts
      this round, both verified clean after (dashboard structure, `sensor.
      watercooler_invoice_history` = 21 records, no errors).

- [x] **2026-09-02 (water session, after Gas Bottles work) — Investigated a
      reported "watering full M/W/F/S, smaller afternoon, half other days"
      pattern; predictive fill was a red herring, found and fixed a real
      false-positive risk instead.** User asked whether `input_boolean.
      water_predictive_fill_enabled` (Branch 4.7) needed tuning. Checked
      live entities + recorder DB directly (Supervisor API + sqlite3 against
      `home-assistant_v2.db`), not just docs. Findings: (1) Branch 4.7 never
      fired in the prior 10 days — it gates on `sensor.energy_orchestrator_
      state == 'conserve'`, which was never true in that window (only
      `normal`/`surplus` + a few instant restart-glitch `critical` blips);
      correctly idle given healthy solar/SOC, not broken. (2) The demand-day
      → depth-target system (`sensor.water_effective_fill_target` reading
      `sensor.water_target_depth_tomorrow`) is intentional day-ahead
      pre-positioning, working as designed — explains why Friday's fill
      looks biggest (pre-filling for Saturday's Pool/1.6m day). (3) Initial
      hypothesis that `binary_sensor.water_refill_allowed` was flapping
      21-56×/day and causing `water_borehole_mid_run_shutdown` to fragment
      each day's fill into multiple sessions was **wrong** — that count was
      raw `state='off'` rows including duplicate re-published states, not
      real transitions. Deduplicated: the gate is stable all day; only real
      daytime blip in 10 days was a 3-minute dip on Aug 31. The multiple
      daily pump sessions are legitimate target-reached → consumption-
      drained → refill-again behavior. Self-corrected this to the user
      before implementing anything, rather than proceeding on the wrong
      diagnosis. **Fix applied anyway** (cheap insurance, not a fix for the
      reported pattern): debounced the `permission_revoked` trigger in
      `water_borehole_mid_run_shutdown` (`water_tank_refill_control.yaml`)
      with `for: "00:01:00"`, mirroring the existing Issue 9 max-depth-stop
      debounce — protects against the rare sub-minute Tuya/recorder blip
      class without weakening real revocations (SOC drop, grid loss,
      orchestrator critical all stay off well past a minute). `ha core
      check` clean (verified via Supervisor API `config/core/check_config`).
      No code bug found in predictive-fill or demand-target logic — both
      confirmed working as designed, nothing else changed. Docs:
      WATER_CONTRACT.md mid-run-shutdown description + new Session Log
      entry (see that file, directly above its "Last updated: 2026-08-21"
      footer).

- [x] **2026-09-02 (absolute last) — Gas Bottles: recalibrated burn-rate +
      price estimates from real purchase data, fixed a real dashboard
      chart-rendering bug.** User: "use value from purchases as base" —
      `gas_avg_days_per_bottle_stove` 75d→136.67d, `_heater` 21d→16d,
      `gas_owned_refill_price_reference` R330→R304.83, `gas_swap_exchange_
      price_reference` R309→R338.90, all computed from the real 6-
      transaction backfill (gap-averages for burn-rate, cost-averages for
      price refs — full math in UTILITIES_CONTRACT.md Section 8's Session
      Log). Set both live (API) and in `gas_helpers.yaml`'s `initial:`
      values. Separately, user reported the Trends section's two charts
      were "squashed," unreadable x-axis, no y-axis, no bar values — real
      bug: `preserveAspectRatio="none"` was stretching a viewBox sized for
      2-4 real months across a much wider card, only not visible on Water
      Cooler's own equivalent chart because that one already has ~20 real
      months. Fixed: `xMinYMid meet` aspect handling + a 4-slot minimum
      width floor, added y-axis 0/mid/max labels, added a value label on
      every bar (kept `<title>` hover as a bonus, not a replacement). Also
      added `multiline_secondary: true` to the 4 cost-tile mushroom cards
      and a Home-dashboard-style section heading ("Cost & Usage Trends")
      that section never had. Full restart done, confirmed live.
      **Correction, found during the same day's `/update-docs` sweep: the
      "pre-existing unrelated `alert_device_entities` duration bug" called
      out here was wrong** — see the dedicated entry below, it was actually
      `sensor.gas_alert_context` missing a required attribute.

- [x] **2026-09-02 (later still) — New domain: Gas Bottles tracker, second
      subsystem in `packages/utilities/` alongside Water Cooler, exactly as
      that domain's contract header anticipated.** 2× 9kg LPG cylinders
      (Owned/refillable, Swap/exchange) feeding the gas stove (year-round,
      main use) and a movable winter-only heater — manual single-hose
      connection per appliance (no auto-changeover regulator), confirmed
      live 2026-09-02: Swap bottle on stove, Owned bottle on heater
      (near-empty from recent use). Departs from Water Cooler's model in one
      key way: consumption tracked **per appliance** (stove/heater), not per
      bottle-pool, since each has a very different burn rate and either
      bottle identity can be assigned to either appliance
      (`input_select.gas_stove_bottle_identity`/`gas_heater_bottle_
      identity`). New files: `gas_helpers.yaml`, `gas_core.yaml`,
      `gas_automations.yaml`, `gas_gauge_history.json`. Full lifecycle:
      optional Place Order → Log Refill/Exchange Done (`input_boolean.
      gas_do_refill`/`gas_do_swap`, either or both in one transaction, cost
      fields, both entry points feed the same completion automation) →
      `sensor.gas_transaction_log` (self-referencing trigger-based template
      sensor, event-triggered off a custom `gas_transaction_logged` event
      rather than the button's own state change to avoid racing that same
      automation's reset of its own toggle helpers — see UTILITIES_CONTRACT.
      md Section 8e) → monthly/annual actual cost sensors. Alert pipeline is
      an exact structural copy of Water Cooler's (`route_gas_alert` /
      cancel-from-notification / snooze-reset / `alert.gas_alert`). New
      Operations dashboard view (`gas-control`) mirrors `watercooler-
      control`'s section layout (Stock & Usage / Order / Log Done / Cost
      tiles + range-filterable inline-SVG trend charts + full-detail
      expander / Settings), plus a Home-dashboard summary tile next to Water
      Cooler's. New skill `.claude/commands/log-gas-reading.md` (mirrors
      `/log-water-invoice`) logs photographed stove-gauge readings to
      `gas_gauge_history.json` on a reminder cycle — deliberately NOT fed
      into the burn-rate EMAs, trend/calibration reference only.
      **Two real config-shape bugs caught live while building this** (not
      hypothetical — HA rejected/silently-broke the config until fixed): a
      trigger-based template sensor must be its own list item under
      `template:`, not nested inside a plain `sensor:` list nor a second
      top-level `template:` key in the same file (duplicate-key warning,
      one block silently dropped); and Jinja dict `.update()` is blocked by
      HA's template sandbox (`SecurityError`) — the dashboard's monthly
      aggregation charts use the same `dict(list_of_tuples)` fold
      `sensor.device_battery_history_log` already established, not a
      mutated dict. `ha core check` clean; live states spot-checked via the
      Supervisor API (`sensor.gas_stove_days_remaining` = 56.2d,
      `sensor.gas_heater_days_remaining` = 7.2d — genuinely low, not a
      fresh-seed artifact, since the heater bottle really is near-empty).
      **Deliberately deviated from an early plan step**: did NOT press
      `input_button.gas_confirm_completed` to "seed a baseline" the way the
      vacuum's manual-clean tracker did — unlike that case, nothing was
      actually refilled/exchanged today, so pressing it would have falsely
      marked both bottles freshly serviced and corrupted the estimate the
      backdated seed connect-times already got right.
      **Restarted successfully** — `alert.gas_alert` confirmed live (state
      `idle`) post-restart via Supervisor API; `.storage/lovelace` dashboard
      edits active. **Real spend history backfilled same session**: a
      Discovery credit card statement screenshot came through (after the
      gauge photo itself still didn't), 6 real transactions May 2025 → Jul
      2026 fired into `sensor.gas_transaction_log` via its
      `gas_transaction_logged` event (dated historically, not `now()`) —
      R2,057 real 2026 spend now showing instead of R0. Categorization
      inferred from supplier name (Gas Affair = combo refill+exchange,
      BigF = exchange-only) and flagged to the user as not certain — see
      UTILITIES_CONTRACT.md Section 8h's Session Log entry for the full
      reasoning. **Real bug caught by the backfill**: `sensor.gas_
      transaction_log`'s own transaction-count `state` read the pre-update
      `this` snapshot instead of independently recomputing, lagging the
      true count by one — exact pitfall `sensor.device_battery_fleet`
      already documents avoiding in this same repo, missed here anyway;
      fixed, cost sensors were never affected (verified R2,057 by hand
      against the 4 real 2026 rows). **Gauge photo itself still
      unreceived** — three upload attempts now, all failed client-side (not
      a code issue) — `/log-gas-reading`'s reading-format section stays
      flagged open pending the first successful one. Docs: UTILITIES_
      CONTRACT.md Section 8 added + this backfill's own Session Log entry,
      this entry, Locked Entity Names below.

- [x] **2026-09-02 (later still) — Smart Cleaning: closed Group V's V1 doc gap
      — vacuum's "31-entity inventory" was never actually written anywhere,
      just pointed at circularly between two files.** User picked this as
      the priority ("v1 for vacuum, others still open"). `vacuum.yaml`'s
      header said "see PROJECT_STATE.md Known Integration Issues" (that
      section never had the list, only the map-bug row); SMART_CLEANING_
      CONTRACT.md Section 4 said "see vacuum.yaml's header comment" (never
      had it either) — neither side actually satisfied the other. Pulled
      the real registry live from `.storage/core.entity_registry`
      (`platform: ecovacs`, 31 entities confirmed) and wrote the full table
      into SMART_CLEANING_CONTRACT.md Section 4; both files' pointers
      corrected to point there instead of at each other. **Also caught a
      real inaccuracy while verifying against live data**: the contract's
      stated domain breakdown said 10 `sensor` entities; the true count is
      13 (missing `area_cleaned`, `total_area_cleaned`, `total_cleaning_
      duration`) — corrected. Also newly noted: `switch.deebot_t80s_biesie_
      advanced_mode` is `disabled_by: integration` (not previously flagged;
      nothing here currently needs it, just documented as genuinely off).
      Checked CLAUDE.md's package table row and this file's Document Index
      row for SMART_CLEANING_CONTRACT.md — both already correct from V14,
      no change needed. Other Group V items (V5 night-run risk, V10 room
      segments, V15 smart plug) deliberately left open per user instruction
      — not touched this pass. Not yet committed — user separately has a
      Gas Bottles subsystem in progress in another session, so held off
      running `gitupdate.sh` broadly to avoid touching those untracked
      files; only this session's doc/YAML changes are ready to commit
      scoped, whenever that's wanted.

- [x] **2026-09-02 (evening) — Tooling: `gitupdate.sh` + `/update-docs` skill
      changed to default to session-scoped commits, not `git add .`.** User
      caught a real problem live: an `/update-docs` pass ran `gitupdate.sh`
      with just a message (no file args), which — because `gitupdate.sh` has
      always done a blanket `git add .` — staged and committed an unrelated,
      unfinished concurrent-session change (`packages/alerts/alerts_doors.
      yaml`, 134 lines, + `packages/utilities/watercooler_invoice_history.
      json`) under a commit message describing neither. User: "update script
      to default to existing session files only unless user requested for
      broader" — then clarified they meant the `/update-docs` **skill**
      (`.claude/commands/update-docs.md`), not just the shell script, so
      both got fixed together. **`gitupdate.sh`**: now accepts optional file
      arguments after the message — `./gitupdate.sh "msg" file1 file2 ...`
      stages only those (`git add -- "$@"`); no-args falls back to the
      original `git add .` unchanged, so the daily 05:00 backup automation
      (`packages/backup/github.yaml`, which calls it with only `{{ reason
      }}`) is completely unaffected — still a full snapshot, as intended.
      **`.claude/commands/update-docs.md`**: Step 6 now defaults to building
      the file list from the session's own edit history and passing it
      explicitly; broad `git add .` only on explicit user request. Step 7
      checklist and "What NOT to do" updated to match.
      **`docs/SESSION_CHECKLIST.md`** (the authoritative source this skill
      is derived from) had the exact same now-stale guidance — caught via
      this session's own `/update-docs` step 3a sweep (`grep -rln
      "gitupdate\.sh"` across docs/, checked each hit for staleness) rather
      than missed the way the audit's own real examples describe — fixed to
      match. **Proven with a real test, not synthetic**: ran
      `./gitupdate.sh "TEST..." docs/PROJECT_STATE.md` — it picked up 30
      real lines another session had added (a Water Cooler entry) while
      leaving `alerts_doors.yaml`, both watercooler files, `ALERTS_CONTRACT.
      md`, `UTILITIES_CONTRACT.md`, and two brand-new untracked files
      (`gas_core.yaml`, `gas_helpers.yaml`) from other sessions completely
      untouched — confirmed via `git status` immediately after.

- [x] **2026-09-02 (later still) — Security: found and fixed a real dashboard
      gap — `input_boolean.security_visitor_alerts_suppressed` (BUG-S77,
      2026-08-31) was built as a manual dashboard mute but never actually
      added to any dashboard.** User asked where the "visitor override
      boolean" was on the security dashboard and said it belonged on the
      Camera System Control side. Checked every dashboard's saved JSON
      (`dashboard_operations`, `_system`, `_overview`, `_testing`,
      `operations_debug`) for `security_visitor_alerts_suppressed` and
      `visitor_alert_snoozed` — both came back **absent from all of them**,
      confirming this wasn't a placement question but a genuine miss from
      the original BUG-S77 session (its own helper comment even says
      "dashboard entry exists for visibility" for `visitor_alert_snoozed`,
      which was never true). Added `security_visitor_alerts_suppressed` to
      the Operations → Security dashboard's existing "Camera System Control"
      entities card (`.storage/lovelace.dashboard_operations`, direct JSON
      edit, backed up first). Did NOT add `visitor_alert_snoozed` — user only
      asked about the override boolean (a persistent manual mute); that one
      is the auto-managed per-cycle "Cancel Alert" snooze, explicitly "not
      meant to be toggled by hand" — flagging its absence rather than adding
      it unasked. Full core restart required (`.storage/lovelace` isn't
      reload-safe) — same restart also picked up the earlier laundry-door
      dashboard addition from this session, which had been sitting in
      `.storage` since before this restart. `ha core check` valid; confirmed
      live post-restart both by API state read
      (`input_boolean.security_visitor_alerts_suppressed` = `off`, present)
      and by re-reading the saved dashboard JSON. Docs: SECURITY_CONTRACT.md
      entity table row updated to note the dashboard-gap fix.

- [x] **2026-09-02 (later) — Water Cooler: Jun 2025 gap closed (now genuinely
      zero gaps, Dec 2024 → Aug 2026, 21 records, R15,589.78 lifetime).
      Fixed the charts (user reported "Monthly Cost broken" — was) and
      reworked the Trends section per user request.** Root cause of the
      broken charts: `markdown` cards run content through DOMPurify, which
      strips raw `<svg>` — confirmed via `/api/template` that the Jinja
      itself was always rendering correct SVG, the display layer was eating
      it. Fixed by switching both chart cards to `custom:html-template-card`
      (raw `innerHTML`, no sanitization) — the exact same card already used
      live elsewhere in this dashboard (the load-shedding schedule
      visualization), not a new unknown. **Then**: added `input_select.
      watercooler_chart_range` (3 Months/6 Months/1 Year/Lifetime, default
      3 Months) — both chart templates slice `monthly_breakdown` by it,
      confirmed via `/api/template` before restarting. Wrapped the selector
      + both charts + the old detail table in a `custom:expander-card`
      (collapsed by default) so the Trends section takes zero space until
      tapped. **Cost Tracking** converted from a 7-row `entities` list to a
      2×2 `grid` of `custom:mushroom-template-card` tiles — note this used
      the actual "Home dashboard style" precedent (mushroom-template-card +
      card_mod, confirmed by reading the vacuum view's Today/Lifetime
      tiles), not the literal "markdown" the user asked for, since that
      literal reading would have contradicted the cited "same as home
      dashboard" — flagged to the user rather than silently substituted.
      Rate Constants (input_numbers, still editable) deliberately left as
      an `entities` card, not touched — only Cost Tracking was named.
      One restart, verified clean after (dashboard structure, `sensor.
      watercooler_invoice_history` = 21, `input_select.watercooler_chart_
      range` live, no errors). Still not visually confirmed in a browser
      this session.

- [x] **2026-09-02 (later) — Alerts: new mute covering BOTH the laundry door
      AND the laundry security gate, `input_boolean.laundry_door_alert_notify`
      (alerts_doors.yaml), same pattern as `input_boolean.camera_alert_notify`
      — user asked for "similar to inside camera for security" but
      auto-resetting the next day; corrected mid-session to fold the gate
      into the same boolean rather than laundry door alone.** Split both
      `binary_sensor.laundry_door_sensor` and `binary_sensor.
      laundry_security_gate_sensor` out of the shared Tier 2 severity loop
      (and the `devices` attribute loop) into their own gated block, same
      thresholds/logic, mirroring the 2026-08-23 garage-door split-out —
      kitchen door and garage security gate are unaffected. New automation
      `laundry_door_alert_midnight_reset` (same pattern as
      `dogs_inside_midnight_clear` in security_automations.yaml) forces the
      boolean back to `on` at 00:00:00 so a forgotten mute can't silently
      leave either unmonitored past the day it was set. Left unmuted
      (deliberately) from `automation.house_secured_check` — that
      bedtime/everyone-left physical-security sweep is a different kind of
      check than the nuisance-alert pipeline this mutes, and muting it there
      too would create a real safety gap. Added to the Operations → Security
      dashboard's existing "Door Control" entities card, right below
      `door_alerts_notify` (`.storage/lovelace.dashboard_operations`, direct
      JSON edit — no card of that name existed for per-door mutes yet, this
      is the first one added there). `ha core check` valid throughout;
      `input_boolean`/`template`/`automation` reloaded live via the Supervisor
      API for the YAML logic change (no `alert:` entities touched); the
      dashboard edit got a **full core restart** per CODING_STANDARDS
      (`.storage/lovelace` isn't reload-safe) — confirmed live after restart
      by re-reading the saved dashboard JSON. Verified live: toggled the
      boolean off, confirmed `sensor.door_alert_context` recomputes with the
      laundry door excluded from `devices` (no errors), then restored it to
      `on`. Docs: ALERTS_CONTRACT.md changelog + Doors Domain table row + File
      Inventory line count (1282 → 1408).

- [x] **2026-09-02 — Water Cooler: full invoice history backfilled (20 real
      months, Dec 2024 → Aug 2026) + two SVG trend charts added to the
      dashboard.** User progressively sent every Aquazania invoice/statement
      they had via `/log-water-invoice` across several messages this
      session; `watercooler_invoice_history.json` now holds 20 records,
      lifetime total R14,861.83, 2026-YTD R6,246.03 (`sensor.watercooler_
      year_actual_cost`) — every record's `water+deposit+rental(+other)`
      cross-checked by hand against each statement's own "Total to settle"
      and matched exactly except where a credit carried over between months
      (expected, documented in the file's own `_comment`). **Schema gained a
      4th bucket, `other_total`** (0 everywhere except Aug 2025's R460
      "Maintenance Cleaning of Cooler" invoice — a real 4th document type
      that doesn't fit water/deposit/rental, added rather than mis-bucketed
      or silently dropped). One duplicate correctly caught and skipped (Apr
      2025 was re-sent). Remaining real gap: **Jun 2025 only** — everything
      else from Dec 2024 onward is now populated.

      **Dashboard**: added two `markdown` cards ("Monthly Cost", "Monthly
      Bottles") to the watercooler-control view's Trends section, each a
      hand-built inline-SVG bar chart driven live by Jinja reading `sensor.
      watercooler_invoice_history`'s `monthly_breakdown` attribute — chosen
      over `custom:plotly-graph` after checking the installed bundle
      (`www/community/lovelace-plotly-graph-card`, confirmed via the actual
      working `prepaid-control` config) only binds to entity/statistics
      history, not arbitrary attribute arrays, so it couldn't plot
      file-backed data. The original "Monthly Trend" markdown table was kept
      as a numeric fallback directly below the charts, not replaced — **not
      yet visually confirmed in a browser** (no screenshot access this
      session); if the SVG doesn't render, the table still shows every real
      number and is the thing to check first.

      Two live restarts this session (JSON/YAML-only edits earlier didn't
      need one; both dashboard edits did, per CODING_STANDARDS). One mid-
      session collision, not a bug: a concurrent session's own dashboard
      save briefly clobbered the just-added `watercooler-control` view
      (stale read-modify-write on the same `.storage` file) — re-applied
      cleanly, confirmed by content hash comparison against pre-edit
      backups, not a rollback of either session's real work.

- [x] **2026-09-02 (later same day) — Vacuum dashboard: 6 metric cards
      (Clean Water, Dirty Water, Detergent, Manual Clean, Today, Lifetime)
      converted from plain markdown to `custom:mushroom-template-card`
      with `card_mod` styling** — user: "where are the nice markdown card
      card mods... similar to what have on home dashboard" + flagged
      Today/Lifetime as wasting space, suggested mushroom-template-card.
      Matched the exact pattern already used on the Home dashboard's room-
      power tiles (rgba background wash via card_mod, conditional on the
      same severity threshold already driving icon_color) and the vacuum
      view's own main status tile (pulse keyframe while in a state that
      needs attention — reused here for "Overdue"/critical instead of
      "cleaning"). Water/Dirty Water/Manual Clean: red/orange/green on the
      days-remaining estimate, pulses when Overdue. Detergent: same 3-tier
      color on %, pulses under 15%. **Today**: new well/bad coloring not
      present before — red (+pulse) if `binary_sensor.deebot_alert_active`
      is currently on, green if area was cleaned today with no alert, grey
      if nothing's run yet today. **Lifetime**: deliberately left neutral
      (blue-grey, no thresholds) — a lifetime total isn't "good or bad",
      just informational, unlike the other five. Replaced 2 horizontal-
      stacks-of-markdown with 2-column `type: grid` layouts (Today+Lifetime
      side by side; the 4 consumable tiles 2×2) — meaningfully more compact
      than the stacked full-width markdown blocks it replaced. All 4
      templates test-rendered via `/api/template` against live data before
      pushing (water correctly showed red/Overdue — genuinely true, matches
      the real un-logged state). Verified live via a fresh `lovelace/config`
      fetch after saving. Docs-only note here — no YAML/entity changes,
      pure Lovelace edit, so no SMART_CLEANING_CONTRACT.md change needed.

- [x] **2026-09-02 — Vacuum: new manual machine-clean tracker (Group V,
      added to Section 3e of SMART_CLEANING_CONTRACT.md) — user asked for
      "same as water refill" for taking the roller out and clearing
      stuck hair/debris.** Same manual-log-plus-EMA pattern as the water/
      dirty-water trackers (3c): `input_button.vacuum_log_manual_clean`,
      `input_datetime.vacuum_last_manual_clean_time`, `input_number.
      vacuum_area_at_last_manual_clean`, EMA pair `vacuum_avg_area_per_
      manual_clean` / `vacuum_avg_days_per_manual_clean` (seeded 300 m² /
      7.0 d — pure guesses, genuinely zero prior data unlike water/dirty-
      water which had one real day to seed from), guarded by
      `input_boolean.vacuum_manual_clean_logged_once` so the first press
      doesn't compute a bogus interval. **Explicitly distinct from the V11
      lifespan sensors** — lifespan tracks wear toward replacement, this
      tracks periodic decluttering of a roller that isn't worn out, just
      tangled; don't conflate the two or reuse the reset-lifespan buttons.
      New `sensor.vacuum_manual_clean_estimate` (days until next needed).
      Config valid, all helpers/template/automation reloaded live and
      confirmed. **User said "can start with logged today"** — pressed
      `input_button.vacuum_log_manual_clean` via the API immediately after
      building it (2026-09-02 ~17:18 SAST) so today is the real, exact
      first baseline rather than the placeholder midnight default. Added a
      3rd row (colour-threshold markdown card, same red/orange/green
      pattern as V13, plus a log button) to the existing Water & Detergent
      card group on the Operations → Vacuum dashboard — fetched the live
      config fresh before editing, did not touch any other card. Docs:
      SMART_CLEANING_CONTRACT.md Section 3e (new) + Section 4 entity tables
      (3 new rows: Helpers, Template Sensors, Automations) + Section 5
      Pipeline Audit + File Inventory line count re-verified (1028, was
      892 as of the 08-31 sweep — grew ~136 lines this session).

- [ ] **2026-08-31 (evening) — New domain: Water Cooler tracker, first
      subsystem of a new `utilities/` package (user's steer: a home for
      recurring consumable-delivery utilities, distinct from `packages/
      water/`'s tank/plumbing system — a future one, e.g. gas bottles, gets
      its own files in the same package). Built from a real Aquazania
      invoice the user attached (account J077431, Jul 2026): stock estimate,
      order→delivery→invoice lifecycle, cost tracking, Operations dashboard
      view + Home summary card. Full detail in `docs/domains/
      UTILITIES_CONTRACT.md`.** Stock model mirrors the vacuum's water/
      detergent estimator exactly (no sensor exists — manual-log button +
      EMA elapsed-time estimate): `input_button.watercooler_log_bottle_
      changed` decrements spare stock, EMA-refines `avg_days_per_bottle`
      (seed 3.9d = 8 bottles/~31 days from the invoice), and `sensor.
      watercooler_current_bottle_fraction_remaining` estimates the mounted
      bottle's level purely from elapsed time — seeded to reproduce the
      user's stated "~1 full spare + ~1/4 bottle mounted" at deploy.
      Order/delivery is a 3-step lifecycle (Place Order → morning-of
      Delivery Reminder → Confirm Delivery), `sensor.watercooler_next_
      delivery_options` shows upcoming Mon/Wed/Fri dates as **guidance only,
      no hard cutoff** (user's explicit answer — "generally 2 work days,
      tighter toward the weekend"). **Real design pivot mid-session, worth
      flagging**: the approved plan for cost history was a pyscript service
      writing a JSON file — live-tested against this HA build (2026.8.3, not
      assumed) and found dead on both fronts: pyscript's `BUILTIN_EXCLUDE`
      explicitly bans `open()` (no file I/O possible from pyscript at all
      here), and `recorder.import_statistics` isn't registered as a service
      in this build either (checked via the Supervisor API services list).
      Landed on something better anyway: the user's actual workflow is
      "paste the invoice into a Claude Code session," so `watercooler_
      invoice_history.json` is maintained directly by Claude (Bash/Edit, no
      HA runtime code, dedup-checked on `reference` before every append),
      surfaced into HA via a `command_line:` sensor (`sensor.watercooler_
      invoice_history`) — a well-worn core-HA pattern, zero custom code, one
      less service to maintain than the original plan. Seeded with the real
      Jul 2026 invoice (8 delivered/7 returned, R936.79 total) so the
      tracker launches with one real data point, not an empty graph. Cost
      constants are all **ex-VAT**, confirmed against the invoice's own line
      math (15% VAT verified two ways) per user's explicit ask to check
      before trusting them; monthly rental is flagged as a reference-only
      estimate since Aquazania bills a variable usage-based factor, not a
      fixed formula — the real figure only ever comes from the invoice log.
      Dashboard: new Operations → "Water Cooler" view (`watercooler-
      control`, mirrors `prepaid-control`'s layout) + a Home-dashboard
      summary card next to the existing Vacuum card, same tap→navigate
      pattern. `ha core check` clean (one `alert:` schema warning caught and
      fixed — missing `repeat`/`notifiers` block, copied from `vacuum.
      yaml`'s `deebot_alert`). **Not yet done this session**: HA restart
      (needed to activate the new `alert:` entity and pick up both `.storage/
      lovelace` dashboard edits reliably — batched per CODING_STANDARDS, not
      yet triggered), and live button-press verification in the dashboard.
      User flagged more invoices (last year's) coming in a later session for
      bulk backfill — same file + dedup mechanism handles that already.
      **Also noted this session, unrelated**: user corrected the vacuum room
      map — "Corridor" in HA is called "Passage" in the Ecovacs app, and the
      room behind the bottom-right "add zone" icon on the app's room-select
      screen is "Reading room". Not applied to `SMART_CLEANING_CONTRACT.md`
      here — another session was actively committing vacuum/room-mapping
      work to that exact file at the same time (commits through `551383ec`,
      21:56) — flagging so it isn't lost, not overwriting concurrent work.

- [x] **2026-08-31 (later) — Alerts BUG-A23: dead duplicate MacBook battery entity
      found + removed, live one renamed.** Follow-up to the BUG-A22 investigation
      above — user asked "is ryan macbook stale alert correct as name is ap
      something?" about the "Ryan Macbook Pro" stale entry in the same dashboard
      screenshot. **Traced via `.storage/core.device_registry` + live states:**
      Ryan's MacBook (`Mac14,10`, hostname "AP-0223-1001") had TWO separate
      `mobile_app` device registrations — the original from 2025-01-10 (custom-named
      "Ryan Macbook Pro Mobile App" by a past session, labelled `battery_monitor`,
      hence the one showing in the alert) and a newer one from 2026-05-19 (no custom
      name, so it displayed as the raw hostname "AP-0223-1001"). The old one's
      `sensor.*_internal_battery_level` stopped reporting at exactly HA's
      2026-08-24T18:16 restart and never came back — confirmed dead, not just slow;
      the new one was live and current (`sensor.ap_0223_1001_internal_battery_level`,
      last reported minutes before the check, 79%→78% across the fix). This resolves
      the "flagged, not deduped, unclear if one Mac or two" open question from the
      2026-08-21 battery-fleet rollout entry below — confirmed: one Mac, re-registered.
      **Fix, user-approved via two explicit confirmations:** (1) stripped
      `battery_monitor` from the dead entity first as an interim measure; user then
      asked "aren't you removing the duplicate?", so (2) deleted the dead registration
      outright — `DELETE /api/config/config_entries/entry/01JH87XY6X1DQVYEHY2J4NNW81`
      via the Supervisor-proxied HA REST API (found after `config_entries/remove` and
      `config_entries/delete` over the WebSocket API both came back `unknown_command`
      — the REST config-entries-delete endpoint is the one that actually exists on
      this HA version). mobile_app registers one config entry per app install, so this
      cleanly removed the dead device and its ~23 entities with zero effect on the
      live one — confirmed via `GET .../states/<dead entity>` → 404 and a
      `core.device_registry` re-check showing the device gone. (3) Set
      `name_by_user: "Ryan Macbook Pro Mobile App"` on the live device
      (`config/device_registry/update`, WebSocket API) so the dashboard now shows the
      real name instead of the raw hostname. `sensor.device_battery_fleet` re-verified
      live post-fix: one MacBook row, correctly named, `severity: normal`. Docs: see
      ALERTS_CONTRACT.md BUG-A23 for the full incident writeup; Locked Entity Names
      below updated to remove the dead entity and annotate the live one.

- [x] **2026-08-31 (late) — /update-docs sweep on the vacuum work (this session's
      own commits, 33812e15 through f8cb4da9) — found and fixed real secondary
      drift in SMART_CLEANING_CONTRACT.md, not just a clean pass.** Per-commit
      docs were kept current throughout the session (PROJECT_STATE.md Group V,
      the contract itself), but the checklist's step 3a sweep caught what
      per-commit updates missed: (1) File Inventory said `~720` lines, real
      file was `892` (`wc -l`) — V4/V10/V11 work added ~170 lines after that
      estimate was written and nothing re-checked it. (2) Section 3 (Pipeline
      Architecture) had no 3d for V4 at all, and 3b's diagram/message
      description still described the pre-V11 fault-only shape (hardcoded
      error-description message) even though the actual YAML had already
      moved to a generic `devices`-attribute-built message covering both
      fault codes and lifespan warnings. (3) Section 4 (Entity Registry) and
      Section 5 (Pipeline Audit) were missing every V4 helper/automation and
      the V11 threshold helper — Future Scope (Section 8) had already been
      updated to say V4/V11 were Done, but the detailed registry tables
      hadn't caught up, exactly the "one line fixed, five references left
      stale" pattern this command exists to catch. All fixed — see the
      contract's own 3b/3d, Section 4, Section 5. **Alert_Test_Plan.md**: no
      TEST section exists for the new vacuum alert pipeline (same gap class
      as Garden, which also never got one) — added a flag note in the same
      style as the existing Device Battery Fleet / BUG-S77 warnings, not a
      full new TEST section (that's a bigger write-up, correctly out of scope
      for a doc-drift sweep). **SYSTEM_CONTRACT.md**: checked, no update
      owed — that doc's Section 3 only covers domains that publish a
      cross-domain interface another domain consumes, and vacuum doesn't
      (its aggregator pickup is via the generic alerts_summary.yaml naming
      convention, already documented as a pattern, not a per-domain entry).
      Vacuum also has no `docs/Context/*.md` quick-ref, consistent with
      Garden also not having one — not a gap. Grepped `deebot|vacuum` across
      every other domain contract + SYSTEM_CONTRACT.md + Context/ — one hit
      (SECURITY_CONTRACT.md's BUG-S77 citing the vacuum Cancel Alert pattern
      as a design precedent) and it's accurate, no fix needed. Other commits
      seen in `git log` from the same day (BUG-A22, BUG-S77, and a separate
      vacuum power-draw investigation in `private_docs/
      POWER_SYSTEM_PERFORMANCE_LOG.md`) were made by other session(s), not
      this one — their own commit messages show docs were already handled
      by whoever made them; not re-verified here, out of scope for this pass.

- [x] **2026-08-31 — Alerts BUG-A22: 9-10 Zigbee door/gate battery sensors falsely
      STALE together for 50+ hours, root-caused live, per-device stale override
      shipped.** User asked "what are these stale alerts?" from an Alerts dashboard
      screenshot showing every door/gate sensor (Main Gate, Front Door, Lounge Door,
      Front Security Gate, Bar Door, Kitchen Door, Reading Room Door, Garage Security
      Gate, Laundry Security Gate) reading "STALE... last seen 50.9h ago" — identical age
      across all of them — plus 3 unrelated genuinely-stale phones/watch/laptop.
      **Investigation (all live, via Supervisor REST API for states/logbook history + HA
      WebSocket API for entity/label registry, using `$SUPERVISOR_TOKEN` — no code
      changed for this part):** confirmed all 10 SNZB-04P battery entities'
      `last_reported` matched to the millisecond; traced via logbook to a single ZHA
      network (SONOFF Dongle-M) reload at 2026-08-29T16:18:35Z — every ZHA entity, not
      just door/gates (also washer/dishwasher/airfryer/fridge plugs, ZBSM switches) went
      `unavailable` then restored ~9s later at 16:18:44Z, resetting `last_reported` for
      all of them at once. Confirmed one-off, not ongoing (no further `unavailable`
      events on any ZHA device in the 50+ hours since). Root explanation: SNZB-04P are
      sleepy Zigbee end-devices that report a battery attribute only on an infrequent
      periodic checkin — naturally sparser than the pipeline's global 24h
      `device_battery_stale_hours` (BUG-A20) even under normal operation, so the whole
      fleet crossed it together and stayed flagged.
      **Fix (user asked to "tag zigbee devices and add an override to global 24hr"):**
      new label `battery_monitor_sparse_reporter` (`core.label_registry`) + new
      `input_number.device_battery_stale_hours_sparse` (96h/4-day default,
      `packages/alerts/alerts_device_batteries.yaml`) — an entity carrying both
      `battery_monitor` and the new label uses the sparse threshold instead of the
      global one in `sensor.device_battery_fleet`'s per-device `eff_stale_h` calc.
      Tagged the 10 real SNZB-04P battery entities live via `config/entity_registry/
      update` (see Locked Entity Names + core.label_registry below for the full list);
      deliberately did NOT tag `garage_door_sensor_battery` (Sonoff platform, not ZHA —
      unaffected, reports normally) or the EZVIZ doorbell. **Self-caught slip:** HA
      auto-slugified the label's real `label_id` as `battery_monitor_sparse_reporter`
      from the name given at creation, not the `battery_monitor_sparse` id assumed when
      first tagging entities — left a dangling label reference on all 10 for one
      round-trip, fixed with a second `config/entity_registry/update` pass before moving
      on to the YAML. `ha core check` passed; `input_number`/`template` reloaded live, no
      restart needed. **Verified live:** all 10 tagged entities read `severity: normal`
      post-fix; the 3 genuinely-stale mobile devices (71h/162.5h/169.3h) still correctly
      flag `stale` under the unchanged global threshold, confirming the override is
      scoped only to tagged entities. **Also corrected in the same pass:** this file and
      ALERTS_CONTRACT.md both still said BUG-A20 was "not yet restart-verified" —
      `sensor.home_assistant_uptime` shows HA's last restart was 2026-08-24T18:16Z, same
      day as and after that fix landed, so it's actually been live since; the caveat had
      just never been removed. Docs: ALERTS_CONTRACT.md (top changelog, File Inventory,
      Device Battery Domain section, new BUG-A22 entry, Section 6 threshold table,
      Section 9 — added a missing "Device Batteries" row that had never existed, Section
      10 table + tally), `docs/Testing/Alert_Test_Plan.md` (extended its existing
      2026-08-24 "needs a new TEST section" flag to cover the sparse override too).
      **⚠️ Process note, not a code issue:** the YAML edit for this landed on disk before
      `./gitupdate.sh` was run for it, and in the meantime a *different* concurrent
      session ran `./gitupdate.sh` for unrelated vacuum work (commit `ac8e216f`, "vacuum:
      V4 daily job summary..." at 21:32:28) — `gitupdate.sh` does `git add .`, so it swept
      up this session's already-written `alerts_device_batteries.yaml` change into that
      commit under a message that doesn't mention it at all. Already pushed to
      `origin/master` by the time this was noticed, so left as-is rather than rewriting
      pushed history — flagging here as the audit trail for anyone who goes looking for
      the battery-alert change under a battery-alert commit message and doesn't find one.

- [x] **2026-08-31 — Security BUG-S77: gardener/staff visitor-at-gate alert spam fixed
      (staff-aware cooldown + scoped mute + Cancel Alert button); also caught BUG-S40
      falsely marked FIXED for 3 months with no code shipped.** User reported the
      2026-08-29 incident (screenshots): a gardener working at the front gate for 30+
      minutes generated a "🚨 Visitor at gate" critical push every ~30s–6min, iOS Snooze
      had no effect (phone-OS only, never touches HA), and toggling
      `input_boolean.security_alert_notify` did nothing — traced to that boolean only
      gating the unrelated repeat-reminder pipeline in `alerts_security.yaml`, never
      wired to `security_event_router`'s one-shot classifier-transition push at all; only
      `security_system_enabled` (the whole-router kill switch) actually stopped it.
      **Root cause:** `sensor.security_event_classification` RUNG 5 classifies a
      gate-loitering staff member (gardener/maid, 7s+ continuous presence in the ipcam01
      zone) as `visitor` (critical) rather than the silent `service_person` — by design,
      since staff *passing through* the gate is filtered separately — but the visitor
      branch's cooldown was a flat 30s regardless, so every on/off `gate_loitering` edge
      re-fired a fresh critical for the whole work session. **This flat-30s cooldown is
      exactly what BUG-S40 (2026-05-23) was supposed to have already fixed** — its own
      session log line below (2026-05-23 S13) and `SECURITY_CONTRACT.md`'s Implementation
      Checklist both claimed "1800s staff cooldown shipped," but the actual code diff
      never happened; found only because the identical symptom recurred live 3 months
      later. **Fix (BUG-S77):** (1) `sensor.security_event_classification` gained a real
      `staff` boolean attribute (`security_logic.yaml`) so the router can read it directly
      instead of parsing the `reason` string; (2) visitor branch cooldown is now 30s for a
      genuine non-staff visitor (unchanged, safety-critical) / 1800s for a staff-loitering
      event (`security_automations.yaml`); (3) new scoped dashboard toggle
      `input_boolean.security_visitor_alerts_suppressed` — mutes only this branch, doesn't
      touch arrival/departure/intruder/perimeter_threat; (4) new Cancel Alert action
      button on the push (phone `CANCEL_VISITOR_ALERT` + Telegram
      `/cancel_visitor_alert`), following the exact BUG-A13/BUG-A19 pattern — sets
      `input_boolean.visitor_alert_snoozed`, auto-resets on whichever comes first: 10min
      of gate-zone quiet, or `binary_sensor.staff_on_site` turning off (the staff-off
      trigger added same session, per user follow-up, so an unrelated visitor arriving
      right after the gardener leaves can't inherit the mute). New automations
      `security_visitor_alert_cancel_from_notification` /
      `security_visitor_alert_snooze_reset`. **Also added:** expandable one-line
      descriptions (via `custom:fold-entity-row` + `custom:template-entity-row`, both
      already-installed HACS resources) on ~20 `input_boolean` rows across the Camera
      System Control / Camera Control (Manual) / Presence Control / Door Control cards on
      the live Operations dashboard (`.storage/lovelace.dashboard_operations`, gitignored
      — edited directly, requires an HA restart to pick up since dashboard storage is
      cached in memory) — user was toggling booleans (`security_alert_notify`, etc.)
      whose actual effect didn't match their names/instincts; descriptions now say what
      each one really does, including the security_alert_notify gap above. **Deferred,
      user's explicit call:** `gate_activity` (RUNG 5c) shares the same cooldown timer and
      nuisance class but was NOT given the same treatment this session ("leave gate for
      now"). **Files:** `security_logic.yaml`, `security_helpers.yaml`,
      `security_automations.yaml`. Reload: Input Helpers + Automations + Template
      Entities. Not yet exercised against a live gate event since shipping — see
      `docs/Testing/Alert_Test_Plan.md` TEST 6 flag. `SECURITY_CONTRACT.md` BUG-S40
      status corrected, BUG-S77 full entry added (Section 6), Section 9 checklist item
      corrected, Section 3 entity rows added, Section 2 file description updated.

- [ ] **2026-08-31 — Ecovacs Deebot: fault-alert pipeline (V3) + water/
      detergent estimator (V12) built; map bug self-resolved; dashboard
      layout correction (user's fix, not mine) + several small fixes.
      Group V: V1 partial, V2/V3/V6/V7/V9/V12 done, V4/V5/V8/V10/V11 open.**
      **User had manually fixed the dashboard layout session-start** ("Updated
      it myself as you messed it up") — the 2026-08-30 night session's
      single-section/columns:4 approach was wrong; the user's real fix uses
      **multiple sections, each card `grid_options: {columns: "full"}`, plus
      view-level `dense_section_placement: true`** — this pattern already
      existed precedented elsewhere in this dashboard (network-control,
      network-debug, both `max_columns: 3`) and would have saved a wasted
      round if checked first. All further dashboard edits this session built
      on the user's layout via a fresh `lovelace/config` fetch each time,
      never re-pushed the old cached file. **Map confirmed working** (user
      screenshot: full render, live position) — the `deebot_client`
      "rcp not support" bug from the prior session self-resolved within
      ~24h with no config change; caveat card removed, Known Integration
      Issues row closed (see below). **Alert pipeline (V3)** built —
      user's direct ask ("why am I not getting priority alerts... same
      pattern as normal alerts with cancel alert etc") — copied
      alerts_batteries.yaml's canonical shape exactly; real trigger the same
      day was error code 103 (wheel jam, confirmed meaning) + code 323
      ×3 (meaning unconfirmed, plausibly a newer water/station code — see
      V3 for the full citation trail). **Schedule correction**: user changed
      the app's workday Auto Clean start 06:00→07:00 in-app ("machine gets
      in way when they getting ready") — `input_datetime.
      vacuum_reminder_time_workday` was already correctly at 06:50 live
      (user must have updated it by hand too), YAML `initial` + dashboard
      schedule card text corrected to match. **Water/detergent estimator
      (V12)** built from scratch — no sensor exists for any of clean-water/
      dirty-water/detergent levels, so it's 3 manual-log buttons +
      self-calibrating EMA estimates, detergent math fully deterministic
      from the user's own stated 1:200/15mL-per-refill ratio. **User
      feedback on the whole-house Auto Clean trial** (V6): "seems overkill
      vacuum and mopping everywhere" — 2 dirty-water empties + 1 refill in
      one day, last job alone was 302 min. Not acted on this session — flagged
      here as a signal that the every-2nd-day per-room scenario schedule
      (still valid, currently unscheduled — see V6) may be the better
      default once duration data is available. **Dashboard also got**: the
      "Today's Activity" combined markdown split into two side-by-side cards
      ("Today" / "Lifetime", user request mid-session), a new "Alert Status"
      + "Water & Detergent" section. Full detail in Group V below.

- [ ] **2026-08-30 (night) — Ecovacs Deebot: dashboard built + mat reminder
      automation shipped. Group V: V1 partial, V7/V9 done; V3/V4/V5/V8/V10
      still open.** `packages/integrations/vacuum.yaml` created — mat-removal
      reminder (input_datetime × 2, input_boolean, 2 automations), validated
      + reloaded live via Supervisor API, not yet committed to CLAUDE.md's
      package table. New Operations → Vacuum dashboard view + a Home summary
      card pushed live via the Lovelace WebSocket API (no restart) — see V7
      for full content. Discovered but NOT used yet: `vacuum.clean_area`
      (targets HA Areas, unverified whether it maps to this device's rooms)
      and `vacuum.send_command` — see V10. Confirmed no camera entity exists
      for this device. Full detail in Group V below.

- [ ] **2026-08-30 — Ecovacs Deebot T80S Omni: mapping finished, scenario/schedule
      plan agreed, live telemetry confirmed working. Group V (see Verified
      Priority Work Queue) V2/V6 marked done; V1/V3/V4/V5/V7/V8 still open.**
      Map "Biesie Main House" done — 9 rooms: Bedroom Main, Bedroom Luke,
      Bedroom Tayla, Bathroom Main, Bathroom kids, Kitchen, Corridor, Living
      room, Reading room. Three app-native scenarios agreed with user: **Daily
      Clean** (Living room + Kitchen + Reading room + Corridor, vac+mop, daily
      09:00), **Bedroom Clean** (Bedroom Main/Luke/Tayla, Mon/Wed/Fri 10:30),
      **Bathroom Clean** (Bathroom Main/kids, higher water flow, Tue/Thu/Sat
      10:30) — Sunday runs Daily only. Trigger stays entirely inside the
      Ecovacs app scheduler, not HA — user confirmed no HA-side start/stop
      automation or solar-gating wanted (V6). User also confirmed no
      dogs-near-vacuum gate is needed (dogs fine around it / kept elsewhere).
      10:30 gap is a placeholder pending real duration data — user asked how
      long a full clean takes and this isn't known yet; told them to check
      `sensor.deebot_t80s_biesie_cleaning_duration` /
      `_total_cleaning_duration` / `_area_cleaned` after each scenario's been
      run for real, to revisit the gap. **Verified live via Supervisor API
      this session (mid-session user question "is anything coming through")**:
      `sensor.deebot_t80s_biesie_total_cleans` = 3,
      `_total_cleaning_duration` = 5.75h, `_total_area_cleaned` = 176 m²,
      last job = 13.0 min / 7 m², battery 99%, error = NoError, vacuum state
      = docked — telemetry pipeline confirmed working end-to-end.
      `event.deebot_t80s_biesie_last_job` reads `unknown`/`event_type: null` —
      **not a bug**, HA `event`-platform entities never backfill from before
      the entity existed, it will populate on the next job that completes
      while HA is running. Camera-misfire check (V5) and consumable/fault
      alerting (V3) remain genuinely untested/unbuilt — don't mark those done
      based on this session.

- [ ] **2026-08-30 (later same day) — Room renamed Sunroom → Reading room in the
      Deebot app map; renamed to match everywhere above and in Group V below.
      User reports "a couple cleans" done but not all three scenarios yet —
      re-checked live via Supervisor API, numbers are unchanged from the
      earlier check this session (`total_cleans` still 3, last job still
      13.0 min / 7 m²), confirming these were partial/spot runs, not full
      Daily/Bedroom/Bathroom scenario completions. Per-scenario duration
      still genuinely unknown — don't guess it. Once Daily, Bedroom Clean,
      and Bathroom Clean have each completed at least once, re-check
      `sensor.deebot_t80s_biesie_cleaning_duration` (last job) right after
      each one finishes (it gets overwritten by the next job) and report
      back or ask for a re-check — that's what tightens the 10:30 start gap
      in the schedule.

- [ ] **2026-08-30 (evening) — Plan changed: user is trialling whole-house
      "Auto Clean" (AI) via the app's own Schedule screen instead of the
      3-scenario split above, for now.** Two schedule slots created in-app:
      **06:00 Auto Clean / Workday** (weekdays) and **08:00 Auto Clean /
      Weekend**. Confirmed with user: "Auto Clean" is whole-house — AI decides
      coverage, not restricted to the Daily Clean room set (Living room/
      Kitchen/Reading room/Corridor). **This means bedrooms and bathrooms now
      get cleaned daily, not every 2nd day as originally requested** — flagged
      to the user as a real trade-off (extra mop/brush wear on rooms that
      didn't need daily attention), accepted as a trial, not walked back.
      The separate Bedroom Clean (Mon/Wed/Fri) / Bathroom Clean (Tue/Thu/Sat)
      scenario schedule from the 2026-08-30 morning entry above is **not
      currently active** — nothing is scheduled against those two named
      scenarios right now, only the two Auto Clean slots. If the whole-house
      daily trial doesn't work out, reverting to the original per-room
      3-scenario schedule (Daily 09:00 / Bedroom Mon-Wed-Fri 10:30 / Bathroom
      Tue-Thu-Sat 10:30) is the fallback — those scenario definitions are
      still valid, just currently unscheduled. Duration data still pending
      either way (see entry above) — once Auto Clean has run a few times,
      pull `sensor.deebot_t80s_biesie_cleaning_duration` /
      `_total_cleaning_duration` to see how long a real whole-house AI run
      actually takes, which also answers whether 06:00 finishes with enough
      margin before people are up on workdays.

- [ ] **2026-08-29 — NEW INTEGRATION: Ecovacs Deebot T80S Omni added (`ecovacs`
      core integration, config entry created 2026-08-29T12:20, account
      dunners1@gmail.com). Currently doing its initial house mapping run — no
      automations/alerts/dashboard built yet.** 31 entities confirmed live in
      the registry under device `deebot_t80s_biesie`: `vacuum.deebot_t80s_biesie`
      (main control), `sensor.deebot_t80s_biesie_battery`,
      `sensor.deebot_t80s_biesie_error`,
      `binary_sensor.deebot_t80s_biesie_mop_attached`,
      `sensor.deebot_t80s_biesie_main_brush_lifespan` /
      `_filter_lifespan` / `_side_brush_lifespan`, `image.deebot_t80s_biesie_map`,
      `select.deebot_t80s_biesie_active_map` / `_work_mode` / `_water_flow_level`,
      `switch.deebot_t80s_biesie_advanced_mode` / `_continuous_cleaning` /
      `_carpet_auto_boost_suction` / `_clean_preference` / `_true_detect`,
      `number.deebot_t80s_biesie_volume` / `_clean_count`,
      `button.deebot_t80s_biesie_relocate` / `_reset_main_brush_lifespan` /
      `_reset_filter_lifespan` / `_reset_side_brush_lifespan`,
      `event.deebot_t80s_biesie_last_job`, plus Wi-Fi/IP diagnostics
      (`sensor.deebot_t80s_biesie_wi_fi_rssi/ssid`, `_ip_address`). No package
      file exists yet — not in `packages/integrations/` and not in
      CLAUDE.md's package table. **Full buildout backlog tracked as Group V
      below — nothing implemented yet, this is planning only.** Don't build
      map-dependent automations (room/zone cleaning, no-go zones) until the
      Omni finishes its first full mapping pass and `select.active_map` /
      room labels stabilize.

- [x] **2026-08-29 — BUG-PWR-GEYSER04: Saturday used the weekday geyser schedule,
      tank cold by a 10am shower. ✅ Reload Helpers + Automations + Template
      Entities done and verified live.** User reported the geyser was cold at a
      10am Saturday shower and asked for the weekend schedule to start a little
      later, plus stay running past the old 08:00 cutoff even once at
      temperature (since showers happen later on weekends). Live history (via
      Supervisor API) showed the geyser reached temperature by 05:46 then was
      hard-turned-off at 08:00 — the winter *weekday* hard-off — because
      `geyser_automations.yaml` bucketed Saturday with Mon–Fri (`is_mon_sat`,
      `dow <= 5`) instead of with the weekend (dedicated
      `geyser_morning_start/end_weekend` helpers already existed but were wired
      to Sunday only). Tank cooled unmanaged for 2h with no coverage until the
      solar-gated 11:00 midday window. **Fixed:** weekday bucket is now Mon–Fri
      only, Sat+Sun share the weekend bucket, everywhere the geyser morning
      window is gated (`geyser_turn_on`, `geyser_turn_off`,
      `geyser_morning_backstop`, orchestrator emergency-off, and the two
      duplicate window-bounds copies in `power_state.yaml`). Weekend hard-off
      pushed later per user request — 09:30 non-winter (was 08:30), **10:30
      winter** (was 09:00) — via a new `geyser_weekend_end_winter_offset`
      helper (60min; the 1h weekend delta didn't match the existing 30min
      `geyser_winter_start_offset` used elsewhere). Also fixed
      `geyser_period_energy_snapshot`, found stale during this change: it
      captured `geyser_energy_at_morning_end` at a fixed 08:00/08:30 regardless
      of day, which would have snapshotted mid-run on the new later weekend
      schedule and inflated the midday-delta calc that gates the evening
      early-start safety net. Full detail: BUG-PWR-GEYSER04 in
      [POWER_CONTRACT.md](../docs/domains/POWER_CONTRACT.md) (Issue 31).
      **Validation done:** `check_config` via Supervisor API returned `valid`;
      Reload Helpers/Automations/Template Entities all done via Supervisor API;
      new `geyser_weekend_end_winter_offset` helper confirmed live at 60min;
      all 4 touched automations confirmed `on` post-reload. **Caveat found
      live:** Reload Helpers does not reset an existing `input_number`'s
      current value to a changed `initial` (only min/max/step/name update) —
      `geyser_morning_end_weekend` needed an explicit `input_number.set_value`
      to 9.5 after the reload. Worth remembering for any future `initial`
      change on an already-created helper.

- [ ] **2026-08-29 — BUG-PWR-GEYSER-DISPLAY01: `sensor.geyser_control_status`'s
      hardcoded 12:00–15:00 midday window predates the 11:00 midday trigger —
      found live, not fixed.** Spotted while verifying the BUG-PWR-GEYSER04 fix
      above (unrelated to it): at 11:41 SAST the geyser was genuinely `on` (the
      11:00 midday solar branch, added 2026-06-21), but the status sensor read
      "Outside active windows" instead of "Running — midday solar" — its
      `in_midday` template in `power_state.yaml` is still hardcoded
      `12 <= now_h < 15`, never updated when `geyser_turn_on` gained the
      earlier 11:00 trigger. Any midday run between 11:00–12:00 misdisplays.
      Fix: change `in_midday` to `11 <= now_h < 15`. See Issue 32 in
      [POWER_CONTRACT.md](../docs/domains/POWER_CONTRACT.md).

- [ ] **2026-08-29 — BUG-L20: stale-copy import of `lighting_arrival_night.yaml`
      re-introduced five closed bugs; caught before reload, all re-fixed.
      ⚠️ `automation.reload` still owed on the box.** User reported that
      "alert + lighting files" edited in a separate session against the git
      repo had been copied onto the HA box over SSH, and asked for
      verification. **Scope check first:** only
      [lighting_arrival_night.yaml](../packages/lighting/lighting_arrival_night.yaml)
      actually differed from git HEAD. Every file under `packages/alerts/`
      matched HEAD exactly; `alerts_network.yaml` and `network_helpers.yaml`
      were rewritten on disk during the copy with **byte-identical content**
      (mtime moved, content unchanged — they briefly showed as modified in
      `git status` purely as stale stat-cache entries). Local and
      `origin/master` were in sync. **If alert-side changes were intended,
      they never reached this box.** **The lighting file was built on a
      pre-2026-06-14 base**, so it silently reverted four closed fixes, and a
      fifth was undone by its auto-off restructure: BUG-L14 (garage entity
      back to `switch.stw_3gang_garage_switch_3` — an entity that has never
      existed in the registry; channel `_3` on that Sonoff is actually
      `switch.boundary_street_light`, and the real garage light is
      `switch.garage_light` on `_1`), BUG-L12 (the 60s `last_arrival_time`
      cooldown — structurally unworkable, since `presence_boundary.yaml`
      stamps `last_arrival_time = now()` immediately *before* setting
      `arrival_detected`, so the delta is ~0s and `> 60` never passes; the
      trigger had also lost its `from: "off"` guard), BUG-L13
      (`front_house_security_light` dropped from Scenario 1), BUG-L03
      (Scenario 1 back to `binary_sensor.anyone_home`, leaving it asymmetric
      with Scenario 2's `anyone_connected_home` — AP-dropped + phone-home
      would have matched neither branch), and BUG-L15 (`bedtime_mode == 'on'`
      dropped from the patio auto-off gate, restoring the exact
      user-reported symptom of patio-off 5 min after any evening arrival).
      A sixth, non-regression defect: `continue_on_error: true` stripped from
      all three `script.notify_lighting_event` calls, so a notify failure
      would abort Scenario 2 before its delay and auto-offs. **All six fixed
      in place.** **Kept from the incoming file** (verified correct, genuine
      improvements): the two post-delay gates are now separate `if`/`then`
      blocks instead of bare `condition:` steps — a real fix, since a bare
      `condition:` inside a `choose` sequence aborts the whole remaining
      sequence; and a new pre-state-aware 5-min auto-off for
      entrance_down/dining_room/laundry, gated on `bedtime_mode` being off
      and skipping any of the three already on before the arrival.
      **Changed by user request:** quiet-mode front-security auto-off is no
      longer a flat 15 min gated on `bedtime_mode` — now 5 min if after
      bedtime (`bedtime_mode` on **or** clock ≥ 21:30, incl. after midnight)
      and 10 min otherwise, ungated, so the light is always released.
      **Validation done:** YAML parses; all 15 entity references resolve
      against `.storage/core.entity_registry`. **Not done — no jinja2 on the
      SSH add-on container:** the `after_bedtime` and
      `entry_lights_to_turn_off` templates were not render-tested; run them
      through Developer Tools → Template, then Check Configuration +
      `automation.reload`. The copy process left
      `packages/lighting/lighting_arrival_night.yaml.bak.20260829001929` —
      that is the **pre-copy original**, byte-identical to the then-HEAD
      version, so it duplicates git history exactly and carries nothing not
      already recoverable via `git show`. It has no `.yaml` extension so HA
      won't load it; safe to delete whenever. `LIGHTING_CONTRACT.md` updated: Section 3 scenario
      table + trigger notes, Section 7 gate-open handoff row, BUG-L20 added,
      BUG-L03/L12/L13/L14/L15/L17/L19 annotated, Section 10 checklist, file
      inventory line count 167 → 235, footer changelog.

- [x] **2026-08-29 — BUG-NET10 score-display follow-up: re-verified 2026-08-27
      CRITICAL fix is live, closed a separate display-only gap it left
      standing.** Session opened re-litigating the original "why does 1 bad
      ping → CRITICAL" complaint against a stale draft that hadn't checked
      live state. Verified `sensor.wan_degraded_alert_severity`'s CRITICAL
      branch is `high_count >= 2` only (no score path) and
      `sensor.wan_noc_status` derives from it, not a raw score threshold —
      both confirmed correct in
      [alerts_network.yaml](../packages/alerts/alerts_network.yaml) and
      [network_helpers.yaml](../packages/network/network_helpers.yaml) as of
      2026-08-27; **no change needed there.** Found a real residual gap: the
      *displayed* `sensor.wan_health_score` still used
      `sensor.wan_max_5min_latency` (max of 3 targets) for its latency term,
      so a single diverging target could still crater the number to ~0 even
      while severity correctly read WARNING — the WAN Degraded push
      interpolates this score verbatim, producing a self-contradicting
      "⚠ WARNING – WAN health score: 0" notification. **Fix:** switched the
      score's latency term to the mean of the 3 existing `wan_*_5min_avg`
      sensors (same ones jitter already reads — no new sensors needed).
      Confirmed via grep this cannot affect CRITICAL/degraded detection:
      nothing downstream of `wan_health_score` except the notification
      display and the now-removed `wan_health_crit` sensor reads it. Also
      removed `sensor.wan_health_crit` (`score <= 30`) — orphaned since
      2026-08-27, zero remaining references. Considered switching
      `binary_sensor.wan_degraded_alert_active`'s trigger threshold to mean
      too (per an earlier session's stated intent) — explicitly declined:
      it's a real detection-sensitivity change, not a display fix, and the
      existing `delay_on: minutes: 5` already anti-flaps transient spikes;
      left for a dedicated decision if it recurs. `ha core check` run
      post-session by the user via Developer Tools → YAML → Check
      Configuration — **valid, no errors or warnings.** Reload Template
      Entities still owed before the new formula/removed sensor take effect
      live (template reload, not a full restart — matches every other
      template-only change in this file). `NETWORK_CONTRACT.md` updated:
      Section 4 (formula + NOC mapping), Section 6 (BUG-NET10 dated
      follow-up), Section 7, Section 8 summary table, header changelog,
      footer changelog.

- [x] **2026-08-27 — BUG-NET10 correction: the 2026-08-23 fix's `score <= 10`
      fallback reproduced the same single-target-CRITICAL bug via a different
      formula term; removed.** User reported still getting CRITICAL alerts with
      only one WAN ping target down (MS or Google), same complaint as the
      2026-08-23 session. Pulled real recorder history for the 2026-08-26
      16:27:56–16:38 UTC incident: `sensor.unifi_gateway_microsoft_wan_latency`
      spiked to 141-144ms for ~10 min while Cloudflare/Google stayed normal
      (`sensor.wan_degraded_reason` = "Microsoft", `high_count` correctly = 1)
      — yet `sensor.wan_health_score` dropped to 0 and tripped the `score <= 10`
      fallback added 3 days prior. Root cause: `wan_health_score`'s **jitter**
      term (`max(3 targets) - min(3 targets)`) is driven almost entirely by a
      single diverging target when the other two are close together — the same
      single-target-max flaw the 2026-08-23 fix addressed in the latency term,
      just reached via jitter instead. **Fix:** removed the `score <= 10`
      branch entirely from `sensor.wan_degraded_alert_severity`
      ([alerts_network.yaml](../packages/alerts/alerts_network.yaml)) — CRITICAL
      now requires `high_count >= 2` and nothing else; no real scenario the
      fallback caught isn't already covered by that count check. Also removed
      the now-unused `score` template variable. `wan_noc_status`
      ([network_helpers.yaml](../packages/network/network_helpers.yaml))
      needed no code change (already derives from the severity sensor), just a
      corrected comment. `ha core check` valid; Reload Template Entities
      applied and live-verified. `NETWORK_CONTRACT.md` updated: BUG-NET10
      correction entry, Section 7, header changelog, summary table row.

- [ ] **2026-08-24 — Tayla iPhone14 entity rename: two rounds of loose ends, one
      was a real functional break (BUG-P22).** Renamed all
      `tayla_iphone14_mobile_app_*` entities to `iphone14_tayla_mobile_app_*` via
      Settings → Entities. Confirmed via `.storage/core.entity_registry`: all 27
      entities on the device now consistently `iphone14_tayla_mobile_app_*`.
      **Round 1 loose ends, both now fixed:** (1) BUG-P21 — `person.tayla_
      dunnington.device_trackers` still listed the dead `device_tracker.tayla_
      iphone14_mobile_app` string; user re-picked the renamed tracker in
      Settings → People → Tayla Dunnington, confirmed live. (2) 13 entities
      (app_version, audio_output, battery_level, battery_state, bssid,
      connection_type, geocoded_location, last_update_trigger, location_
      permission, sim_1, sim_2, ssid, storage) still carry a `name` override
      ("Tayla iPhone14 Mobile App X") instead of falling back to `original_name`
      like the rest of the device's entities — **still open**, clear the Name
      field on each to finish the consistency pass (cosmetic only).
      **Round 2 — BUG-P22 (High, found + fixed same day):** re-verifying the
      BUG-P21 fix surfaced that `device_tracker.tayla_iphone_tracker` — the
      UniFi tracker `presence_core.yaml`/`presence_validation.yaml` actually
      read for Tayla's AP location, 4 template references — had been
      collaterally renamed to `iphone14_tayla_tracker` during the same
      cleanup, outside anything either of us intended to touch. Same failure
      class as BUG-P08: nonexistent entity → `state_attr` silently returns
      `None` → `sensor.tayla_ap_location` reporting "Disconnected" and her
      unknown-AP contribution silently zero since the rename. Fixed: all 4
      template references updated to `device_tracker.iphone14_tayla_tracker`.
      `ha core check` passed. **Not yet live** — needs the same reload/restart
      as the battery fix below; restart scheduled by user as of this writing.
      See PRESENCE_CONTRACT.md BUG-P21/BUG-P22 for full writeups.
- [x] **2026-08-24 — Device battery pipeline: added staleness detection**
      (`packages/alerts/alerts_device_batteries.yaml`). Root cause found via
      `.storage/core.restore_state`: `sensor.iphone14_tayla_mobile_app_battery_
      level` (then `tayla_iphone14_mobile_app_battery_level`) was frozen at 5%
      since `2026-08-23T16:16:39Z` (mobile_app had stopped reporting, likely
      iOS Low Power Mode background-refresh throttling) and the pipeline kept
      re-firing CRITICAL on the dead reading. New `input_number.device_battery_
      stale_hours` (24h default) checks each labelled entity's `last_reported`
      (ticks on every device write, unlike `last_changed` which only moves on
      value change); past the threshold the device's severity becomes `stale`
      instead of trusting the frozen SOC — still feeds the alert pipeline
      (binary_sensor/alert_context/`alert:` all treat it like warning/critical)
      but the message reads "last seen Xh ago" instead of a false battery
      claim. ⚠️ Requires HA restart (new `input_number` in a package that
      already needs one for its `alert:` entity) — restart scheduled by user,
      not yet completed as of this writing. BUG-P22's presence fix (below)
      rides along on the same restart.
- [x] **2026-08-23 — BUG-S76 fixed: `sensor.security_event_classification` title/body/
      camera could disagree; ladder fallback mislabelled non-perimeter events as
      "Perimeter activity".** User got a push titled "⚠️ Perimeter activity" whose body
      read `zones: Beds | ... | cam: cam15_passage` and whose attached photo was the
      bedroom passage — zero perimeter signal anywhere. Traced via the recorder DB
      (`home-assistant_v2.db`) to a `template.reload` at 18:50:26 SAST that flapped
      several presence sensors (`family_departing`, `anyone_connected_home`) through
      `unavailable` for ~3s; during that window the classifier's ladder fell through
      RUNG 7/7b (grounds) and RUNG 8d (inside)'s `not departing` guards — deliberately
      there so a departing car/family member doesn't escalate to intruder/critical — with
      nothing below them to claim the event correctly, so it dropped into the bottom
      `else: perimeter_threat` fallback despite zero perimeter motion (same bug *class* as
      BUG-S52, which only closed this for the plain grounds+anyhome:false case). Root
      architecture cause: the sensor was a legacy `state:`+`attributes:` template where
      title, body (`reason`), and camera/image routing (`zone_label`) were three
      independently-rendered Jinja templates that could disagree (same class as
      BUG-S69/S70/S74/S75, each of which patched one symptom without fixing the
      architecture). **Fix:** rewrote `sensor.security_event_classification`
      (`packages/security/security_logic.yaml`) as a **trigger-based** template sensor —
      classification + zone + camera are now decided together in one ladder pass via a
      shared `variables:` block, and the bottom fallback is now zone-aware (`perimeter_
      threat` can only fire when `perim` is actually true; grounds/inside-only events that
      fall through their dedicated rungs now correctly land on `family_movement` with their
      real zone/camera). All rung trigger conditions/order preserved verbatim — this was a
      rendering-architecture + fallback fix, not a reclassification-logic change. `ha core
      check` → `{"result":"ok"}`; `homeassistant.reload_all` applied live (plain `template.
      reload` did not hot-swap the platform type); confirmed live with real motion active
      (Perim+Grounds+Beds, family home) rendering consistently (`family_movement` /
      `bedroom passage` / `cam15_passage`); manually traced the original bug's exact inputs
      through the new ladder → now resolves correctly instead of mislabelling perimeter.
      `SECURITY_CONTRACT.md` updated: BUG-S76 full writeup (Section 6) + Section 3 entity
      row. `SYSTEM_CONTRACT.md` line 219 also corrected in the same sweep — the Security
      Domain Published Interface's `sensor.security_event_classification` output-state list
      was stale independent of this bug (was missing 7 of the 12 live states; not something
      this session broke, just noticed while in the area). **Note:** `packages/alerts/
      alerts_doors.yaml`, `packages/presence/presence_boundary.yaml`, the `ids_hyyp` custom
      component, `solcast_solar/*.json` cache files, `www/community/kiosk-mode/*`, and
      `www/security_latest.jpg` all had pre-existing uncommitted changes at commit time,
      unrelated to this fix (the door/gate sensor session below, plus other in-flight work)
      — deliberately excluded from this commit/doc-update pass per user instruction ("only
      this session").

- [x] **2026-08-23 — New Zigbee door/gate sensors onboarded (kitchen, laundry, reading
      room, garage security gate) + entity-ID fix + garage door alert redesign + battery
      monitoring restored.** Five new SNZB-04P (eWeLink) Zigbee contact sensors were
      physically added: `bar_door_sensor`/`front_door_sensor`/`front_security_gate_sensor`/
      `lounge_door_sensor` already existed pre-session; `kitchen_door_sensor`,
      `laundry_door_sensor`, `laundry_security_gate_sensor`, `garage_security_gate_sensor`,
      `reading_room_door_sensor` are new this session.
      - **Entity-ID bug found + fixed:** 4 of the 5 new sensors' Area wasn't set before HA
        auto-named them, producing doubled-prefix entity_ids
        (`binary_sensor.kitchen_kitchen_door_sensor`,
        `binary_sensor.reading_room_reading_room_door_sensor`,
        `binary_sensor.garage_garage_security_gate_sensor`,
        `binary_sensor.laundry_laundry_security_gate_sensor`). Renamed to the clean
        `binary_sensor.kitchen_door_sensor` / `reading_room_door_sensor` /
        `garage_security_gate_sensor` / `laundry_security_gate_sensor` — 21 + 7 = 28
        entities total (binary_sensor + battery/tamper/lqi/rssi/update/button) via direct
        `.storage/core.entity_registry` edit, `unique_id` untouched. **Lesson learned
        live:** editing `core.entity_registry` is only safe to do BEFORE HA has loaded
        the file this run — an edit made after HA is already up gets silently reverted by
        HA's own registry flush on the next `ha core restart` (reproduced once with
        `laundry_security_gate_sensor`; fixed by having the user rename that one via the
        UI instead, which doesn't have this race). Two `ha core restart`s were needed
        this session; areas (Kitchen/Reading Room/Garage/Laundry) were already correctly
        assigned at the device level throughout, only the entity_id was broken.
      - **`presence_boundary.yaml`:** new `laundry_entry_event`/`laundry_departure_event`
        automations, exact mirror of `house_entry_event`/`house_departure_event`
        (laundry_door + laundry_security_gate pair, same 30s cross-check, same shared
        `input_boolean.arrival_detected`/`house_entry_event` writes) — laundry inherits
        front door's exact arrival-lighting behavior for free since that's driven off
        `arrival_detected`, not a per-door rule. `kitchen_door_sensor` deliberately NOT
        wired into this boundary-resolver pattern — no `kitchen_security_gate_sensor`
        hardware exists to pair it with.
      - **`alerts_doors.yaml` tier placement:** `kitchen_door_sensor`, `laundry_door_sensor`,
        `laundry_security_gate_sensor`, `garage_security_gate_sensor` → Tier 2 (entry);
        `reading_room_door_sensor` → Tier 3 (house control, same as lounge/bar). User
        decision: Tier 1 (perimeter) is reserved for the property boundary only — the new
        gate sensors are inside the property, so they join Tier 2 with their paired doors
        rather than Tier 1 alongside main_gate/front_security_gate. Wired into both the
        `group:` block (dashboard tier-summary) and the real severity engine
        (`sensor.door_alert_context` — trigger list, rank computation, duration tracking,
        devices attribute) — all new sensors automatically inherit the existing
        night/nobody-home escalation and sustained-open Cancel-Alert notification with no
        further wiring.
      - **New "House Secured Check" automation** (`alerts_doors.yaml`): sweeps all 11
        doors/gates (every tier, incl. garage) and alerts if anything's still open, at
        bedtime (`input_datetime.house_secured_check_time`, default 21:30,
        dashboard-adjustable) and whenever everyone leaves
        (`binary_sensor.anyone_connected_home` on→off) — suppressed while
        `binary_sensor.low_trust_present` is on (covers maid Mon/Thu + gardener Sat
        automatically, no separate day-of-week logic needed). Silent when all clear;
        warning at bedtime, critical when everyone's left.
      - **`garage_door_sensor` alert logic redesigned** (existing Sonoff sensor, unrelated
        to the new Zigbee gate): split out of the shared Tier 2 night-based logic into its
        own condition block, per user request — garage is left open most of the day
        regardless of who's home, so the old "night + anyone home" trigger was noisy.
        Away/nobody-home branch unchanged (info → warning after ~5 min). Home branch now
        suppressed unless `binary_sensor.security_lighting_required` (the dusk/dark
        trigger `lighting_boundary.yaml` uses for the boundary/street lights — NOT the
        generic `night` flag) AND `binary_sensor.all_family_home` are both on; same
        warning/critical thresholds as before once that condition holds. `front_door_sensor`
        and the other Tier 2 members are unaffected, still on the shared logic.
      - **Battery monitoring — real regression found + fixed, not just "5 new added":**
        `alerts_device_batteries.yaml`'s `battery_monitor` label (see Device Battery Alert
        Entities below) had **zero entities labelled** despite this file documenting a
        15-entity rollout as done 2026-08-21 — `sensor.device_battery_fleet` was live but
        reporting an empty roster. Restored the original 15 AND added the 5 new
        door/gate sensors' battery entities, 20 total, applied live via the HA WebSocket
        API (`config/entity_registry/update` — no restart needed, avoids the same
        registry-edit race noted above). Root cause of the label loss not identified —
        flagged as open below.
      - `ha core check` valid; `automation`/`template`/`input_datetime`/`group` reloaded
        live (package YAML changes), no restart needed for those. Functional checks:
        `sensor.door_alert_context` live-verified reaching `critical` for a genuinely-open
        garage door (48.7 min, lights on, all home) after the redesign; `sensor.device_battery_fleet`
        live-verified showing all 20 labelled entities after the battery fix.
      - **Docs:** `ALERTS_CONTRACT.md` (Doors Domain, Device Battery Domain, File
        Inventory), `PRESENCE_CONTRACT.md` (Section 5, Section 9, File Inventory), this
        file's Hardware Summary + Locked Entity Names + Device Battery Alert Entities, all
        updated same session.
      - **Note:** `custom_components/ids_hyyp/*` and `packages/security/security_logic.yaml`
        had unrelated uncommitted changes sitting in the working tree at commit time from
        outside this session — deliberately left untouched and uncommitted here per user
        instruction ("only this session"); not this session's work, not documented here.
      - **Open follow-up:** root cause of the `battery_monitor` label registry going from
        15 entities to 0 was not identified — worth a look if it recurs, since it means
        label-based onboarding (the documented mechanism for this whole alert domain) can
        silently regress with no alerting on the regression itself.

- [x] **2026-08-23 — BUG-NET10 follow-up: WAN Degraded CRITICAL no longer triggers off
      a single bad target.** User asked why a network alert read CRITICAL when only one
      of the 3 WAN ping targets (they suspected Microsoft) was affected, expecting
      WARNING until 2+ targets degrade. Traced to the exact architectural gap the
      2026-08-18 BUG-NET10 fix had explicitly flagged and left unfixed pending more
      data: `sensor.wan_health_score`'s latency term is `max(cf, google, ms)`, and both
      `sensor.wan_degraded_alert_severity` and `sensor.wan_noc_status` escalated to
      `critical` on `score <= 30` alone — a threshold one target can trip solo — with
      the severity sensor's `high_count >= 2` branch present but unreachable (ordered
      after the score check). Fixed in
      [alerts_network.yaml](../packages/alerts/alerts_network.yaml) (severity sensor now
      requires `high_count >= 2` or `score <= 10` for critical) and
      [network_helpers.yaml](../packages/network/network_helpers.yaml) (`wan_noc_status`
      now derives from the corrected severity sensor instead of duplicating the raw
      threshold). `ha core check` valid via Supervisor API
      (`{"result":"valid"}`); Reload Template Entities applied and live-verified — all 4
      affected entities recomputed cleanly, no errors. `NETWORK_CONTRACT.md` updated:
      BUG-NET10 entry, Section 7 (also corrected a separate long-stale claim there —
      "for: minutes: 5 on the numeric_state trigger" was never the real implementation;
      actual mechanism is `delay_on: minutes: 5` on the binary sensor, BUG-NET10's own
      2026-08-18 writeup already said this but Section 7 was never corrected to match),
      and the Section 9 summary table — which was also missing BUG-NET10 and BUG-NET11
      rows entirely despite both having full Section 6 writeups (same drift pattern the
      2026-08-21 audit found domain-wide). **Note:** `packages/alerts/alerts_doors.yaml`
      and `packages/presence/presence_boundary.yaml` had substantial uncommitted changes
      at commit time (new kitchen/laundry/reading-room door + gate sensors, dated
      2026-08-23) from a concurrent peer session — deliberately excluded from this
      commit/doc-update pass per user instruction; that session owns its own docs update.

- [ ] **2026-08-22 — RESUME-HERE: consolidated action items surfaced by the 2026-08-21
      docs audit (9 domain contracts + POWER_CONTRACT.md + PROJECT_STATE.md master
      tables + SYSTEM_CONTRACT.md + Testing/ + Context/), not yet actioned.** These are
      genuine code/content gaps the audit *found*, distinct from the doc-drift
      corrections the audit *fixed* (already done — see the dated session-log entries
      below). Nothing here needs more investigation — each was independently confirmed
      live during the audit. Pick any one and go straight to implementing.
      - [ ] **`alerts_power.yaml` battery-low alert reads the wrong SOC sensor**
            (SYSTEM_CONTRACT.md IV-04 / POWER_CONTRACT.md). Reads
            `sensor.inverter_1_battery` (per-inverter, slave-only) in ~4 places instead
            of the published aggregate `sensor.inverter_battery_soc`. Breaks if inverter
            role assignment ever changes. **2-minute fix, 4 entity-name replacements,
            zero design decision needed** — the smallest genuinely-open item from the
            whole audit.
      - [ ] **`water_notifications.yaml` reads raw Tuya sensor, not validated depth**
            (SYSTEM_CONTRACT.md IV-06). Line ~158, `tank_level` value in a notification
            message uses `sensor.water_tank_level_sensor_liquid_level` (raw %) instead of
            a validated-depth-derived percent. Low priority, 1-line fix.
      - [ ] **8 alert domains still double-deliver via a redundant `notifiers: [STD_Alerts]`**
            (NOTIFICATIONS_CONTRACT.md BUG-N18). Water and network were fixed 2026-08-18/21
            respectively (removed the now-redundant `notifiers:` line from the `alert:`
            entity, since `route_*_alert` automations are the real delivery path and
            `STD_Alerts` itself started working again 2026-08-09, BUG-N16). Same fix
            needed in: `alerts_temperature.yaml` (×4 streams), `alerts_doors.yaml`,
            `alerts_presence.yaml`, `alerts_device_power.yaml`, `alerts_power.yaml`,
            `alerts_media.yaml`, `alerts_batteries.yaml`, `alerts_garden.yaml`. One
            restart-batch session (⚠️ `alert:` entity changes require a full HA restart —
            batch all 8 together per CODING_STANDARDS).
      - [ ] **`input_text.security_event_session` 255-char overflow risk**
            (SECURITY_CONTRACT.md ISSUE 9, still open). Needs a pyscript-managed state
            object or 3 discrete fixed-size `input_text` entities — see CODING_STANDARDS
            Rule 5.
      - [ ] **`www/` snapshot retention — 31,812 files and climbing, no cleanup**
            (SECURITY_CONTRACT.md ISSUE 10, re-verified worse than documented this
            session — was 1,871 when originally filed). No shell_command/pyscript purge
            exists anywhere in `packages/`. Add a daily cleanup for snapshots older than
            N days.
      - [ ] **`intruder_high` correlation state has no distinct escalated treatment**
            (SECURITY_CONTRACT.md ISSUE 14, downgraded not closed). It's consumed now
            (treated identically to plain `intruder`) but the original ask — a distinct
            critical-severity path for the holiday-mode-escalated case — was never built.
      - [ ] **pyscript load group health check** (POWER_CONTRACT.md Recommendation 7).
            No binary_sensor detects `group.known_power_loads` being empty/unsynced at
            startup, and no re-sync trigger exists if `sync_power_groups.py` fails
            silently.
      - [ ] **`packages/notifications/power_notifications.yaml` — 0-byte empty stub**,
            unchanged since at least 2026-02-03. `notify_power_event.yaml` is the real
            handler. Needs a delete-or-populate decision, not urgent either way.
      - [ ] **`docs/Testing/Alert_Test_Plan.md` needs an actual run.** Created
            2026-04-14, has exactly one Test Run Log entry with no Pass/Fail results
            filled in anywhere, while the alert pipeline it tests changed substantially
            since (BUG-A08–A19, BUG-N15–N18). Re-read ALERTS_CONTRACT.md's current
            Domain Pipeline Audit before trusting a specific expected-result cell,
            especially Test 6 (Security) and Test 7 (Quiet Hours).
      - [ ] **IMP-IDS02 — document which weather integration is canonical for what**
            (INFRA_CONTRACT.md, low priority). OWM + OWM History + Met.no 6h + Met
            Nowcast all installed; only OWM is marked canonical, the other 3 have no
            documented purpose.
      - [ ] **`MI-06` — no home-detection reconciliation sensor** (SYSTEM_CONTRACT.md,
            needs a design decision first, not just implementation). AP-based
            (`binary_sensor.anyone_connected_home`) and Mobile-App-based
            (`binary_sensor.anyone_home`) can disagree (phone home but off WiFi, or on
            guest WiFi) with no `binary_sensor.anyone_probably_home` to reconcile them.
            Decide the combination logic before building.
      - [ ] **Cosmetic only, not urgent:** `water_tank_refill_control.yaml` still passes
            a dead `rate_limit_minutes: 60` data key to `script.notify_water_event` at 2
            call sites (emergency-refill branches) — the script has no such field, this
            was already a no-op before today's session (found while implementing WATER_CONTRACT
            Recommendation 5, out of scope to fix there). Harmless; safe to remove
            whenever that file is next touched.

- [x] **2026-08-22 — `/update-docs` skill itself updated with the 9-domain audit's
      findings (step 3a + routing-table extensions); never had its own session log
      entry across either edit.** Two changes to `.claude/commands/update-docs.md`,
      made during the 2026-08-21 audit arc but only now logged: (1) step 3a
      "secondary-drift sweep" added — the checklist previously only routed a fix to
      wherever step 3's table sent it, with nothing telling a session to also check
      File Inventory rows, summary tables, other domain contracts, and stale code
      comments for the same fact; every one of the 9-domain audit's passes found this
      exact pattern. (2) Step 3's routing table extended with 3 new rows found missing
      only after the audit was already underway: cross-domain interface changes now
      route to `SYSTEM_CONTRACT.md` (which had gone stale for 4+ months with zero
      mechanical trigger pointing anyone at it); package file changes now also route to
      the matching `Context/*.md` quick-ref's Package Files list (found stale across
      all 6 Context files in the 17th pass); alert/notification pipeline mechanics
      changes now flag the affected `Testing/Alert_Test_Plan.md` section as due for
      re-run (found never-executed since creation in the same pass). Step 3a's own
      "every other contract" bullet widened to explicitly include
      `SYSTEM_CONTRACT.md`/`Context/*.md`/`Testing/*.md`, not just `docs/domains/*.md`
      — the first several audit passes only grepped `domains/`, and several stale
      cross-references outside it were only caught in later follow-up passes as a
      result. No functional/YAML change — tooling only.

- [x] **2026-08-21 (17th pass, same day) — Testing/ + Context/ deep drift sweep (4th
      follow-up after the 9-domain audit; user said "carry on" a second time after
      SYSTEM_CONTRACT.md).** Doc-only. Covered `Testing/Alert_Test_Plan.md` and all 6
      `Context/*.md` quick-reference files (explicitly lower-fidelity than the domain
      contracts per CLAUDE.md — scoped this pass to banners + mechanical fixes
      (Package Files lists, the clearest stale "Known Problems" claims) rather than a
      full line-by-line audit at contract-depth, since that would duplicate work already
      done in the authoritative contracts today):
      - **Alert_Test_Plan.md** — never actually run (one Test Run Log entry, from
        creation day, no Pass/Fail filled in anywhere) while the alert pipeline changed
        substantially since (BUG-A08–A19, BUG-N15–N18). Fixed 2 concrete stale claims:
        "Temperature alerts bypass notify script — BUG-A03 pending" (fixed 2026-06-19)
        and a known-issue row about `cam01` snapshots (cam01 deprecated 2026-05-08, no
        longer part of the active fleet). Added a banner flagging the rest as unverified
        against current delivery mechanics.
      - **PRESENCE/ALERTS/WATER/SECURITY/POWER_CONTEXT.md** — every one of these had a
        stale Package Files list naming files that don't exist (3-5 phantom files each)
        and/or misspelled/wrong-plural real filenames — all corrected against the live
        tree. Fixed one or two of the clearest "Known Problems" claims per file where
        directly confirmable from today's other audits (Presence #2 security-trust-model,
        Alerts #3/#4 notify-bypass, Security #4 trust-model, Water all 3 known problems +
        its Safety Abort Logic table describing disabled dry-run protection as active and
        a deleted max-runtime helper as current). POWER_CONTEXT.md's Hardware Architecture
        also had stale battery/PV capacity figures (pre-2026-06-17 swap, pre-2026-08-21
        correction) — fixed to match PROJECT_STATE.md's Hardware Summary.
      - **NOTIFICATIONS_CONTEXT.md** — Package Files had 4 wrong filenames + 2 missing
        files; rest of the document is pattern/convention guidance, not live-state
        claims, so left otherwise unaudited (lower risk by nature).
      - **POWER_DEPENDENCY_ANALYSIS.md** — a different kind of document (technical
        dependency graph, not a context quick-ref) with real value but real drift: file
        count stale (20 vs live 26, missing 3 files from the analysis entirely), and its
        Load Shedding section describes content that migrated out of `power/load_shedding.yaml`
        into a separate top-level package back in April — that section's own "should
        split this out" recommendation already happened. Flagged clearly with a banner
        rather than attempting a full graph re-derivation (out of proportion for a
        drift pass) — pointed at POWER_CONTRACT.md Section 15 as the place to
        cross-check current claims.
      Every file's own top banner now says "superseded by `docs/domains/X_CONTRACT.md`
      — use the contract for real work," consistent with SYSTEM_CONTRACT.md's existing
      closing note. No `ha core check` needed — zero YAML touched, Markdown-only session.
      **This is the last item on the "anything else to audit" backlog raised after the
      original 9-domain sweep** — remaining lower-value surface (SESSION_STARTERS.md,
      AUDIT_PROMPTS.md, AUTOMATIONS_AUDIT.md, FETCH_URLS.md) is process/meta
      documentation, not live-state description, and wasn't flagged as worth auditing.

- [x] **2026-08-21 (16th pass, same day) — SYSTEM_CONTRACT.md deep drift sweep (3rd
      follow-up after the 9-domain audit + POWER_CONTRACT.md + PROJECT_STATE.md's own
      master tables; user said "carry on").** Doc-only, but this was the most-drifted
      single document all session — dated 2026-04-13/16, essentially untouched since,
      while describing a system state that changed almost immediately after (most of its
      "critical" findings were fixed within days). Re-verified all 6 Interface
      Violations (Section 4) + all 8 Missing Interfaces (Section 5) + the Section 6 risk
      tables + Section 7 execution plan + Section 8 architecture violations +
      spot-checked Sections 2/3 live:
      - **10 of 14 IV/MI items were already resolved**, most within days of the document
        being written (IV-01/02/03/05, MI-01/02/03/04/05/08 — dates 2026-04-14 through
        05-17). **3 are genuinely still open, confirmed live, not stale claims:
        IV-04** (`alerts_power.yaml` reads `sensor.inverter_1_battery`, not the
        published aggregate `sensor.inverter_battery_soc`), **IV-06** (water
        notifications read raw Tuya %, not validated depth), **MI-06** (no
        `binary_sensor.anyone_probably_home` reconciliation sensor exists — AP-based and
        Mobile-App-based home detection can still disagree). **MI-07 done via a
        different mechanism** than literally described (an `energy_saving_mode` boolean
        intermediary, not lighting reading `sensor.power_state` directly).
      - Section 6's "Most Fragile Shared Helpers" and "Highest-Risk Failure Scenario"
        both described risks already closed (notifications_enabled now in YAML; water
        finally has an alert pipeline). Section 7's "Safe to Do Immediately" table was
        7/8 done, only the IV-04 fix (`E1`) still outstanding — a genuine 2-minute fix
        that's been sitting untouched since April.
      - Section 8 (Architecture Violations) had 3 of 8 rows stale: trust-model-in-context/
        and power_helpers.yaml layering both already fixed elsewhere; "Lighting bugs (9
        open)" is actually 0 open (all L01-L19 fixed, confirmed during today's LIGHTING
        sweep).
      - Section 2/3 spot-checks found 4 more: `home_context`'s stale "depends on
        security_mode" note (already fixed, BUG-CTX03); `printer_cartridge_state`'s
        stale "broken" flag (BUG-INF01 fixed 2026-06-19); the Context Domain published-
        interface list still named `guest_mode`/`holiday_mode`/`entertaining_mode` as
        Context outputs (moved to `presence_trust.yaml`, BUG-P11); and
        `sensor.global_house_state`, listed as a Context output, doesn't exist anywhere
        in `packages/` (grep-confirmed) — likely a naming confusion with the real
        `sensor.home_context`, corrected.
      - Appendix (Bug Cross-Reference) doesn't claim status (relationship index only) so
        wasn't rewritten, but added a note warning most rows are closed while
        **SEC-BUG-02 is confirmed still open** — don't assume every row is resolved.
      - **Not actioned:** IV-04/IV-06/MI-06 are real code gaps surfaced by this doc
        audit, not fixed — flagging for the user to decide whether to action as a
        follow-up, since they're runtime-behavior changes, not doc corrections.
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (15th pass, same day) — PROJECT_STATE.md's own master tables swept
      (2nd follow-up after the 9-domain audit; POWER_CONTRACT.md was the 1st).** User
      asked "anything else to audit?" a second time after POWER — pointed at this file's
      own Locked Entity Names / Hardware Summary / Domain States / Priority Work Queue /
      Document Index, since they're exactly the kind of master reference table that
      rotted in every domain contract today. Findings:
      - **Hardware Summary + Locked Entity Names** — spot-checked against everything
        found moved/renamed/removed today (trust model location, `door_alert_context`,
        `security_correlation` unique_id, Cancel Alert rollout entities); all already
        accurate. No changes needed here — unlike the domain contracts, these sections
        get touched almost every session.
      - **Domain States table** ("Post-Audit 2026-04-16") — added a banner clarifying
        it's a historical changelog, not current state (some rows untouched since April
        despite months of further change). Fixed 2 concrete wrong claims: Alerts' file
        count (14→16) and Lighting's "SOC energy saving trigger remains future work"
        (shipped 2026-06-19 — confirmed during today's LIGHTING sweep). Garden has no row
        at all — noted, not backfilled (out of scope for a drift pass).
      - **Priority Work Queue, Group U3 (Apple device battery pipeline)** — found still
        marked `[DEFERRED]`, blocked behind the iCloud integration (U1/U2). It's actually
        done: `alerts_device_batteries.yaml` (2026-08-21) covers 4 iPhones + 2 Apple
        Watches, and — the real finding — it was never blocked on iCloud at all, since
        the battery entities are HA Companion App (`mobile_app`) sensors, confirmed live
        in `core.entity_registry`, unrelated to the broken `icloud` integration. Fixed
        U3's entry and the matching "Known Integration Issues" row for `icloud`, which
        had claimed Apple battery/charging state was "lost" to the same breakage.
      - **Document Index** — every domain contract's "Authority" date was stale
        (2026-04-16/30, predating today's sweeps); this file's own self-reference said
        "Updated 2026-04-16"; Lighting's description claimed "9 open bugs" (actually 0,
        all fixed); GARDEN_CONTRACT.md had no row at all. All corrected/added.
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (14th pass, same day) — POWER_CONTRACT.md deep drift sweep (follow-up
      to the 9-domain audit — user asked "anything else to audit on docs?" after all 9
      were done; POWER wasn't in the original list since it'd had targeted issue-closures
      earlier the same day, but never the systematic sweep).** Doc-only, biggest contract
      in the repo (3106 lines, 26-file domain). Findings:
      - **File Inventory + "Layering Violation" section** both still described
        `power_helpers.yaml` as mixing `group:`/`template:` content into the helpers
        layer — stale. Issue 17 (same file's own Known Issues, resolved earlier the same
        day) already documents this fixed; confirmed live — `power_helpers.yaml` now
        contains only `input_boolean`/`input_select`/`input_number`/`input_datetime`.
        Corrected both.
      - **Section 13 (Optimization Recommendations) — 5 of 7 were stale, none marked:**
        Rec 1 done (Issues 1-4 all resolved elsewhere in the same doc); Rec 2 done
        (`sensor.prepaid_balance_confidence` exists live); Rec 3 superseded by a
        different 4-factor Buy Score v2 design (Section 14's Implementation Checklist
        already knew Buy Score v2 shipped 2026-06-15 — Section 13 didn't reflect it, and
        the two sections actually disagreed on which factors it uses); Rec 5 done
        (superseded by the modern alert pipeline, not migrated verbatim — also already
        correctly recorded in the Sprint 3 checklist, just not in Section 13); Rec 6 done
        (`automation.prepaid_auto_reconcile` exists live, does exactly this). Rec 4 was
        moot (the sensor it proposed fixing was deleted outright per Issue 7). Rec 7
        confirmed still genuinely open — not everything was stale.
      - **Section 12 (Error Signatures)** had a stale `group.security_power_sensors`
        "missing" row — actually a naming confusion (real group is
        `group.house_security_power_sensors`) already resolved and explained in the
        Sprint 2 checklist and Issue 8, just never reflected in this table.
      - Known Issues (Section 11) itself was already accurate and current — same pattern
        as every other domain today: the bug catalog gets maintained, everything else
        (File Inventory, Recommendations, Error Signatures, cross-references) doesn't.
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (13th pass, same day) — INFRA_CONTRACT.md deep drift sweep (9th and
      FINAL domain of this session's multi-session audit; WATER + ALERTS + SECURITY +
      PRESENCE + LIGHTING + CONTEXT + NOTIFICATIONS + NETWORK + GARDEN all done same
      day — all 9 domain contracts now swept).** Doc-only. Findings:
      - **Part 8 ("admin/ — Package Directory (Empty)")** — stale since as early as
        2026-06-28. `packages/admin/tablets.yaml` (95 lines) is a real, actively-used file
        managing dashboard tablet screen brightness (day/night, presence-gated). Rewrote
        the section to document it.
      - **IMP-IDS01 ("IDS Hyyp has no package file")** — stale since 2026-04-29.
        `packages/security/security_alarm.yaml` was created that exact day, doing exactly
        what the recommendation asked (a documented interface stub, automations
        deliberately left in `automations.yaml` pending integration setup). Marked done;
        also fixed 2 other "no package file" mentions for `ids_hyyp` in the Integration
        Registry and Integration Notes to match. (This file was separately found missing
        from SECURITY_CONTRACT.md's own File Inventory earlier in this session — fixed
        there too.)
      - **Integration Registry table (Part 7)** — versions re-verified against every
        `custom_components/*/manifest.json` live. 6 of 15 were stale: `solcast_solar`
        v4.5.2→v4.6.1, `sonoff` 3.11.1→3.12.2, `load_shedding` 1.5.2→1.7.0, `pyscript`
        1.7.0→**2.1.0** (a major version bump — flagged for a changelog check next time
        pyscript is touched, not investigated further here), `alarmo` 1.10.18→1.10.19,
        `openweathermaphistory` 2026.04.03→2026.05.03. The other 9 (solarman,
        hikvision_next, tuya, localtuya, ids_hyyp, hacs, watchman,
        met_next_6_hours_forecast, metnowcast) matched exactly.
      - Parts 1-6 (core/, backup/, office/, weather/, sensors/, integrations/) — file
        lists, entity names, and Known Bugs sections all re-verified live and found
        already accurate/well-maintained; no changes needed.
      No `ha core check` needed — zero YAML touched, Markdown-only session.
      **This completes the 9-domain deep drift audit requested at session start.** See
      the closing summary at the top of this session's work for the full cross-domain
      tally.

- [x] **2026-08-21 (12th pass, same day) — GARDEN_CONTRACT.md deep drift sweep (8th of 9
      domain contracts; WATER + ALERTS + SECURITY + PRESENCE + LIGHTING + CONTEXT +
      NOTIFICATIONS + NETWORK already done same day).** Doc-only, but the biggest gap
      found all session — this contract hadn't been touched since creation (2026-04-29)
      despite `alerts_garden.yaml` growing from ~120 to 282 lines with real feature work
      since, most recently 2026-08-18:
      - **Real delivery path entirely undocumented.** `automation.route_garden_alert`
        (added 2026-07-06) is the actual notification sender —
        `script.notify_system_event` on a 20s/1h/2h schedule — not the
        `alert.garden_alert → STD_Alerts` path this contract solely described. Added a
        full Section 3 diagram + Section 4/5 rows.
      - **Cancel Alert feature entirely undocumented.** `input_boolean.garden_alert_snoozed`
        + `garden_alert_cancel_from_notification` + `garden_alert_snooze_reset` (BUG-A19
        pattern, added 2026-08-18) were missing from every table. Added.
      - **Surfaced, not fixed:** `alert.garden_alert`'s own `notifiers: [STD_Alerts]` is
        live again (BUG-N16 fixed it 2026-08-09) and is now redundant alongside
        `route_garden_alert` — this domain is very likely double-delivering every event.
        This is exactly NOTIFICATIONS_CONTRACT.md's BUG-N18, already known to be one of
        the 8 still-open files from this session's NOTIFICATIONS pass — cross-referenced
        both ways, not actioned here (restart-batch fix, out of scope for a doc pass).
      - File Inventory line count corrected (~120 → 282).
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (11th pass, same day) — NETWORK_CONTRACT.md deep drift sweep (7th of 9
      domain contracts; WATER + ALERTS + SECURITY + PRESENCE + LIGHTING + CONTEXT +
      NOTIFICATIONS already done same day).** Doc-only, small domain (3 files +
      alerts_network.yaml), otherwise well-maintained. Findings:
      - **IMP-NET01 (Section 8)** — "verify wired" item for whether
        `sensor.network_alert_context` feeds `sensor.alert_device_entities`. Confirmed
        live: it matches the `_alert_context` substring filter (unlike
        `sensor.camera_health_context`, which ALERTS_CONTRACT.md documents as the one
        that doesn't match) — always has been wired correctly. Marked done.
      - **Section 3's Alert Pipeline block** had 3 stale claims about `alert.network_alert`:
        said "repeat 60 min" (live is a 4-stage `[3, 10, 30, 60]` schedule, matching
        ALERTS_CONTRACT.md's own Alert Entity Inventory), "skip_first:true" (not set on the
        live block at all — defaults to false), and implicitly still delivering via
        `STD_Alerts` (its `notifiers:` was removed 2026-08-18 as part of BUG-N18, already
        corrected in this session's NOTIFICATIONS_CONTRACT.md pass).
      - **Header "Source" line** was missing `network_ups.yaml`/`network_nas.yaml` despite
        the contract having full sections (9, 10) about both — expanded.
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (10th pass, same day) — NOTIFICATIONS_CONTRACT.md deep drift sweep
      (6th of 9 domain contracts; WATER + ALERTS + SECURITY + PRESENCE + LIGHTING +
      CONTEXT already done same day).** Doc-only. Findings:
      - **Files Audited table** — 3 filenames didn't match live files
        (`notify_security_event.yaml`/`notify_presence_event.yaml`/
        `notify_lighting_event.yaml` vs live `notify_security_events.yaml`/
        `notify_presence_events.yaml`/`notify_light_events.yaml`), and
        `power_notifications.yaml` (the domain's 13th file, a 0-byte empty stub
        unchanged since at least 2026-02-03) was missing entirely — added, flagged as an
        orphan needing a delete-or-populate decision (not actioned, out of scope for a
        drift pass).
      - **BUG-N18 (double-delivery via stale `notifiers: STD_Alerts`)** — still listed
        `alerts_network.yaml` as one of 9 open files. Re-verified live: it was fixed the
        same day as water (2026-08-18) — the live YAML's own comment explicitly says so
        and even names this exact contract entry as needing an update that never
        happened. Corrected to 2-fixed/8-open (temperature, doors, presence,
        device_power, power, media, batteries, garden still carry an active
        uncommented `notifiers: [STD_Alerts]` block, re-checked one by one).
      - **Section 6 "Priority 1" bypass table** — all 3 rows (`presence_notifications.yaml`,
        `alerts_temperature.yaml`, `alerts_device_power.yaml`) described violations
        already fixed by BUG-N05/BUG-A03/BUG-A04 (the latter two already correctly
        marked Fixed in this same session's ALERTS_CONTRACT.md pass). Re-verified live —
        all 3 route through the correct `script.notify_*_event` calls now — and struck
        through the table plus its "Migration Priority" list.
      - **Section 8 ("Water Notifications — Dead Trigger")** — described
        `water_tank_full_notification` as triggering on a `sensor.water_state` value
        that can never fire. Live code — and this same day's WATER_CONTRACT.md Issue 2
        re-verification — shows it already triggers on
        `binary_sensor.water_tank_full_depth` instead. Corrected here plus the matching
        Section 9 cross-domain-dependency row.
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (9th pass, same day) — CONTEXT_CONTRACT.md deep drift sweep (5th of 9
      domain contracts; WATER + ALERTS + SECURITY + PRESENCE + LIGHTING already done same
      day).** Small, doc-only, otherwise well-maintained (2 files, 169 lines). Two findings:
      - **Section 3's `home_context` note** still said `sensor.home_context` depends on
        `sensor.security_mode` (a layering inversion), directly contradicting BUG-CTX03 a
        few sections below in the same file, which already documents this fixed
        2026-04-30. Confirmed live: `home_context` reads only `binary_sensor.
        security_nobody_home` + `binary_sensor.night_confirmed`, both self-contained in
        `context/` — the old security_mode-dependent block is commented out for history.
      - **Section 5 (Published Outputs)** still listed `input_boolean.guest_mode`/
        `holiday_mode`/`entertaining_mode` as context/-published outputs. All three
        actually live in `presence/presence_trust.yaml` (moved 2026-04-30, same session as
        BUG-CTX01 — Section 1's own note already said context/ no longer owns trust model
        entities, this table just never got updated to match). Removed the 3 rows,
        cross-referenced PRESENCE_CONTRACT.md; also noted `holiday_mode`/`entertaining_mode`
        are now read by `sensor.security_correlation` (this session's SECURITY_CONTRACT.md
        ISSUE 14 finding).
      All other entities in Sections 3/5 re-verified live and confirmed accurate. No `ha
      core check` needed — zero YAML touched.

- [x] **2026-08-21 (8th pass, same day) — LIGHTING_CONTRACT.md deep drift sweep (4th of 9
      domain contracts; WATER + ALERTS + SECURITY + PRESENCE already done same day).**
      Doc-only — the Bug Catalog (Section 7) was already fully accurate; drift was in the
      surrounding sections:
      - **File Inventory (Section 2)** — all 14 line counts were stale `~N` approximations,
        corrected to live exact values (`lighting_morning.yaml` grew the most, ~120→257).
      - **Section 3 "Known Scene Gap (BUG)"** — still described `scene.scene_night_away`
        as missing `switch.entrance_down_lights` and citing BUG-L02 as an open gap; both
        stale. `entrance_down_lights` is present live, and BUG-L02 itself is marked "Not a
        bug" (main_entrance_light's exclusion from the scene is intentional design, per an
        inline comment). Corrected the scene table row too — it listed `office_entrance`
        turning on, which isn't part of the live scene at all.
      - **Section 5 (Helper Inventory)** — `patio_second_wake_time` was listed twice with
        contradictory status (once as "defined but not referenced — potential orphan",
        once correctly as "REMOVED 2026-04-28, BUG-L08"); removed the stale duplicate.
        Also two BUG-L09 references still described `lighting_entertainment.yaml`/
        `lighting_energy_saving.yaml` as empty — both long populated (161/108 lines).
      - **Section 8 (Cross-Domain Dependencies)** — 4 rows named `context_presence.yaml`/
        `context_schedules.yaml` as providers for `low_trust_present`, `entertaining_mode`,
        `holiday_mode`, `bedtime_mode` — neither file exists any more (migrated to
        `presence_trust.yaml` 2026-04-30 per PRESENCE_CONTRACT.md BUG-P11, and to
        `lighting_helpers.yaml` 2026-04-28 respectively). Also cross-referenced this
        session's SECURITY_CONTRACT.md ISSUE 14 finding — `entertaining_mode`/
        `holiday_mode` are now actively read by `sensor.security_correlation`, not just
        "not consumed yet" as the row implied.
      - **Section 10 (Implementation Checklist)** — one item flagged "still OPEN (M2
        remainder)" for a power-domain SOC auto-trigger; confirmed shipped 2026-06-19
        (`energy_saving_mode_auto_enable`/`_auto_disable` in `power_automations.yaml`).
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (7th pass, same day) — PRESENCE_CONTRACT.md deep drift sweep (3rd of 9
      domain contracts; WATER + ALERTS + SECURITY already done same day).** Doc-only — no
      code changes, the presence pipeline needed none (Bug Catalog, Section 10, was
      already fully accurate — every BUG-P entry correctly marked Fixed with a date).
      The drift was entirely in the *other* sections describing pre-fix states as current:
      - **File Inventory (Section 2)** — all 6 line counts were stale/approximate
        (`~220`, `~280` etc.) — corrected to live exact values. Header's "Scope: All 5
        packages/presence/*.yaml files" corrected to 6 (matches Section 2's own heading,
        which already said 6 — only the top banner disagreed).
      - **Section 7 (Cross-Domain Dependencies)** — the entire section described the
        pre-BUG-P01/P02/P03 broken trust chain ("⚠️ Reads wrong entity", "❌ Always
        false/off") as the current state, even though those three bugs are marked Fixed
        earlier in the same document. Re-verified live: `security_trust_mode` and
        `security_low_trust_active` both now read the derived `binary_sensor.*`, not the
        orphaned `input_boolean.*`; `boundary_permissive_window` is the S1.3 OR-chain, not
        always-false. Also found `alerts_doors.yaml` no longer references
        `sensor.security_trust_mode` at all (uses `sensor.security_mode` +
        `binary_sensor.security_night_mode`/`_nobody_home` instead) — updated that
        subsection to match.
      - **Section 8 (Unknown AP Detection)** — described the `*_iphone` vs
        `*_iphone_tracker` entity-name mismatch as unverified/current; it's BUG-P08, fixed
        2026-07-10 — confirmed live, both `presence_core.yaml` and `presence_validation.yaml`
        consistently use the `_tracker` suffix now.
      - **Section 9 (Trigger Integrity Audit)** — still listed `presence_test_arrival` as
        a live "⚠️ TEST AUTOMATION" risk with "it should be removed"; it was removed
        2026-04-15 (BUG-P07) — confirmed gone from `packages/presence/`, row struck
        through.
      - **Section 11 (Trust Model Design)** — an 8-item "Migration Path (Priority Order)"
        that is now fully complete (every item maps 1:1 to an already-Fixed BUG-P entry,
        including the orphaned `input_datetime.low_trust_start/end` — confirmed 0 matches
        in `core.entity_registry`). Added a resolved-banner so it stops reading as an open
        TODO; kept the content as historical design record.
      - **Section 13 (Summary of Issues) undercounted** — BUG-P14 and BUG-P18 both have
        full Fixed writeups in Section 10 but were missing from this summary table;
        "Fixed/closed: 15" corrected to 17.
      No `ha core check` needed — zero YAML touched, Markdown-only session.

- [x] **2026-08-21 (6th pass, same day) — SECURITY_CONTRACT.md deep drift sweep (2nd of 9
      domain contracts; WATER + ALERTS already done same day).** One tiny code fix (a stale
      misleading comment, not behavior); rest doc-only. Findings, all cross-checked against
      live `packages/security/`:
      - **File Inventory (Section 2)** was missing 2 of the 9 live files —
        `security_alarm.yaml` (IDS Hyyp alarm stub, not yet wired) and
        `security_history_cleanup.yaml` (one-shot manual cleanup script) — added both.
        Noted CLAUDE.md's "security/ (7 files)" is the same stale count, flagged not fixed
        (out of scope for this contract).
      - **ISSUE 1 (duplicate snapshot files, HIGH) — found already fixed, never marked.**
        `security_capture_best_snapshot`'s `filename_cam` has no `security_` prefix
        (Option A implemented); confirmed 0 orphan `security_cam*.jpg` files live. Sprint 2
        checklist boxes ticked to match. www/ retention cleanup (the one still-open part)
        tracked under ISSUE 10.
      - **ISSUE 10 (www/ snapshot retention) — still open, worse than documented.** File
        count grew from the doc's 1,871 to **31,812** live; no cleanup mechanism exists
        anywhere in `packages/`. Flagged as worth re-prioritizing above MEDIUM given the
        ~17x growth, not changed unilaterally.
      - **ISSUE 11 (duplicate condition block) and ISSUE 13
        (`sensor.security_intruder_level`) — both already fixed, never marked.** The
        duplicate condition is gone; the sensor was removed outright (not just given a
        unique_id) — Section 3's Entity Reference table still listed it as a live LOCKED
        core sensor, corrected.
      - **ISSUE 14 (`intruder_high` state) — downgraded, not closed.** Now confirmed
        consumed (`binary_sensor.security_intruder_active` treats it same as plain
        `intruder`) but still not given the distinct escalated severity the issue actually
        asked for — left open with corrected status.
      - **Recommendations 8.3 (Telegram photo) and 8.4 (cam14/cam15 priority) — both
        already implemented, never marked done.** 8.1 (last-active-camera coverage)
        corrected to current camera names (9/11 sources, cam09/14/15 still missing) — the
        doc's camera list (cam01/06/11) predated the ipcam* rename. 8.5's "mode: restart"
        claim was stale — live is `mode: single`; left open with the corrected mechanism
        noted since the underlying risk is different but not eliminated. 8.7 (EZVIZ
        doorbell) — confirmed still disabled by user choice, but noted the doorbell
        hardware itself is confirmed in use (new device-battery monitoring, see ALERTS
        entry above) so the doc's "if removed, delete" branch no longer applies.
      - **One live code fix:** `security_automations.yaml`'s header "Toggles" comment
        table claimed `holiday_mode` "❌ DOES NOTHING" and `entertaining_mode` "❌ NOT
        USED" — both false, both are read in `sensor.security_correlation`
        (`security_logic.yaml`) to produce `intruder_high`/`ignore` respectively. Comment
        corrected in place; no behavioural change.
      Validated: local YAML parse + `ha core check` both pass (only file touched with real
      YAML content was the one comment fix; docs are Markdown).

- [x] **2026-08-21 (5th pass, same day) — ALERTS_CONTRACT.md deep drift sweep (1st of 9
      domain contracts in this multi-session audit; WATER already done same day, GARDEN/
      PRESENCE/LIGHTING/SECURITY/NETWORK/NOTIFICATIONS/CONTEXT/INFRA remain).** Doc-only —
      no code changes, ALERTS pipeline itself needed none. Findings, all cross-checked
      against live `packages/alerts/`:
      - **File Inventory (Section 2) line counts** — every single one of the 16 rows was
        stale (2026-04-13 baseline never refreshed as files grew via bug fixes/features
        since); corrected all 16 against live `wc -l` (e.g. `alerts_water.yaml` doc said
        123 lines, live is 614; `alerts_doors.yaml` doc said 639, live is 1103). Also fixed
        the header's "Scope: All 14 packages/alerts/*.yaml files" → 16.
      - **Network Domain "In aggregator trigger: Missing / 60s lag"** — stale, and
        self-contradicting: this same file's Section 3 and Section 10 already record
        `alert.network_alert` fixed into the aggregator trigger list 2026-04-14 (BUG-A05);
        confirmed live at `alerts_summary.yaml:410`. Only this one table cell had never
        been updated to match.
      - **BUG-A04, BUG-A05, BUG-A06 detail entries (Section 8) had no Status line** —
        unlike BUG-A01/A02/A03/A07/A08 which each say "✅ Doc-drift correction... already
        fixed", these three read as still-open even though Section 10's summary table
        already correctly listed all three as Fixed (with dates). Added matching Status
        lines, re-verified each fix live (route_device_power_alert no longer calls
        notify.STD_*; alert.network_alert/alert.media_alert both in the trigger list;
        sensor.doors_open_alert_severity confirmed deleted).
      - **Section 5 entity table, `alert.security_alert` row** — still described the
        pre-BUG-A19 state ("UI button not yet wired... see BUG-A10"); BUG-A19 (2026-08-18)
        already wired a Cancel Alert button (`CANCEL_SECURITY_ALERT`, confirmed live in
        `alerts_security.yaml`) — updated the row.
      - **Confirmed accurate, no change needed:** Section 6 (Threshold Configuration) —
        all 17 `input_number` helpers exist (most are UI-managed, in `core.entity_registry`
        not YAML, consistent with CODING_STANDARDS Rule 4). Section 7 (Cross-Domain
        Dependencies) — all referenced entities/groups exist live. Camera Health's
        `sensor.camera_health_context` naming-mismatch finding (doesn't match the
        `_alert_context` substring the flattened aggregator filters on) — re-verified live,
        still true, correctly documented already.
      Validated: no YAML changed this pass (docs-only), so no `ha core check` needed.
      Pacing: user chose "one domain, then check in" — reporting this one before starting
      the next.

- [x] **2026-08-21 (4th pass, same day) — WATER_CONTRACT Recommendations 3-5 actioned.**
      Evaluated all three open Recommendations against live `packages/water/` code
      rather than implementing blindly:
      - **Recommendation 3 (explicit `"full"` state on `sensor.water_state`) —
        evaluated, declined.** The practical need is already covered by
        `binary_sensor.water_tank_full_depth` (Issue 2's 2026-08-21 resolution) and
        `sensor.borehole_control_status`'s "Tank full" line. Adding `"full"` would be
        a breaking change to every `== 'ok'` / `not in ['ok', ...]` check in
        `alerts_water.yaml` (would newly treat a full tank as alert-worthy) and
        `water_state_extensions.yaml` — not a clear win for the stated benefit.
      - **Recommendation 4 (single `sensor.water_lifecycle_state`) — evaluated,
        declined.** `sensor.water_refill_cycle_summary` already is the documented
        single source of truth for cycle state (own header comment says so),
        already merges the two flags into one state + full attributes, already
        consumed elsewhere. A new sensor would be a third, drift-prone
        representation, not a consolidation, and risks reading as a second source
        of truth competing with `a_water_lifecycle_contract.yaml`'s locked
        "REQUIRED FLAGS". Minor gap noted (no true "idle" state on the existing
        summary sensor) but not worth a new sensor to fix.
      - **Recommendation 5 (rate-limit spike-rejection notification) — implemented.**
        `water_depth_spike_rejected` (`water_protection_automations.yaml`) now gates
        its `script.notify_water_event` call behind a cooldown
        (`input_datetime.water_spike_notify_last_sent` +
        `input_number.water_spike_notify_cooldown_minutes`, default 30 — new helpers
        in `water_helpers.yaml`), same `input_datetime.<x>_last_alert` idiom already
        used by `unknown_draw_warning` in `power_automations.yaml`. `logbook.log`
        stays unconditional (audit trail intact); only the push/Telegram notify is
        throttled. **Finding along the way:** the `rate_limit_minutes: 60` data key
        already present at 2 call sites in `water_tank_refill_control.yaml`'s
        emergency-refill branches is a no-op — `script.notify_water_event`
        (`notify_water_events.yaml`) has no such field and no rate-limit logic; not
        fixed this session (out of scope for Recommendation 5), flagged in
        WATER_CONTRACT.md for awareness.
      Full writeups in WATER_CONTRACT.md §8 (Recommendations 3/4/5). Validated via
      local YAML parse + `ha core check` (both pass). No entities added need dashboard
      wiring (cooldown helpers are internal-only). No restart required — helpers and
      automations reload via Developer Tools.

- [x] **2026-08-21 (3rd pass, same day) — New Device Battery Monitor domain: dashboard +
      alert pipeline for every battery device NOT already tracked.** User asked for a
      battery dashboard covering all sensors/trackers with batteries (door/gate sensors,
      security gate sensors, etc.), excluding inverter SOC and UPS (already on Power),
      with dedicated sections for iPhones/Watches/non-dashboard tablets, a critical/warning
      alert (critical under 5% OR <1 day left, warning under 10%, with a Cancel Alert
      button), and — key requirement — devices added to a monitoring group at onboarding
      time rather than hand-edited into the dashboard/alerts per device.
      **New `battery_monitor` label** (`.storage/core.label_registry`) is the onboarding
      mechanism: apply it to a device's battery entity (or the device itself — HA's
      `label_entities()` resolves both, same as the vendored auto-entities card's own
      label filter) and it flows into everything below with zero YAML/dashboard edit,
      picked up within 5 minutes. Applied it now to the 15 existing candidates found via
      `original_device_class == 'battery'` in the entity registry — full list in this
      file's "Device Battery Alert Entities" locked-entity block below.
      **New `packages/alerts/alerts_device_batteries.yaml`** (~330 lines), same canonical
      shape as every other alerts_*.yaml domain: `sensor.device_battery_history_log`
      (self-referencing trigger template, one daily snapshot per labelled entity, 14-day
      window — feeds the "estimated time to expiry based on usage patterns" requirement
      via a fitted %/day drain rate, not a guess from a single reading) →
      `sensor.device_battery_fleet` (full roster every 5 min, the actual source of truth
      the dashboard reads) → `sensor.device_battery_alert_context` (thin severity filter,
      matches the `_alert_context` naming convention so `alerts_summary.yaml`'s
      aggregator auto-discovers it with zero wiring) → `binary_sensor.device_battery_alert_active`
      → `alert.device_battery_alert` (30/60 min repeat, BUG-A13 Cancel Alert pattern
      reused verbatim). Added `alert.device_battery_alert` to alerts_summary.yaml's
      aggregator trigger list.
      **New "Batteries" dashboard view** in `.storage/lovelace.dashboard_operations`
      (storage-mode edit, backed up first — same risk class already established for
      lovelace in this repo), inserted right after "Mobile Devices" per the user's
      request, with iPhone / Watch / Tablet (non-dashboard) / Other Devices sections each
      rendered as a single templated markdown card reading `sensor.device_battery_fleet` —
      genuinely zero-touch onboarding, a new labelled device just appears.
      **Findings surfaced, not silently fixed:** two laptop battery entities both
      registered under device name "AP-0223-1001" (`sensor.ap_0223_1001_internal_battery_level`,
      `sensor.ryan_macbook_pro_mobile_app_internal_battery_level`) — unclear from the
      registry alone whether this is one Mac re-registered or two; left both labelled and
      flagged for the user to check. **Resolved 2026-08-31, BUG-A23** (see this file's
      top session log) — confirmed one Mac, re-registered; the second entity was a dead
      orphaned registration, deleted outright. No iPad currently has a battery entity (only unifi
      presence trackers) — Tablets section will read empty with an onboarding hint until
      one exists. `sensor.ha_system_monitor_battery` (HA host/Pi — no real battery)
      deliberately excluded from the label.
      **⚠️ Requires HA restart** — new `alert:` entity (can't be reloaded) + the two
      direct `.storage/` edits (label registry + entity registry labels + lovelace
      dashboard) all need one to take effect; batched into a single restart per
      CODING_STANDARDS' "batch all alert changes into one session" rule. `ha core check`
      not yet run this session — do that before/instead of trusting the local YAML parse
      validation performed here.

- [x] **2026-08-21 (2nd pass, same day) — WATER_CONTRACT Issue 5 redesigned per user
      pushback, Solcast PV spec pinned down with exact numbers, BUG-S61 DNS theory
      retracted, `ha core check` confirmed passing by user.** Follow-up to the audit
      session below, addressing the user's direct corrections:
      - **Issue 5 rebuilt, not the originally-drafted fix.** User pushed back on the
        flat "max runtime minutes" idea as arbitrary (a fill from empty legitimately
        takes longer than a top-up). Agreed and replaced it: new
        `water_borehole_degraded_rise_rate_protection` automation
        (`water_protection_automations.yaml`) stops the pump if
        `sensor.water_tank_depth_rate` stays between the existing no-rise floor
        (0.01 m/h) and a new tunable healthy floor
        (`input_number.water_refill_degraded_rate_threshold`, default 0.05 m/h) for a
        tunable sustained duration (`input_number.water_refill_degraded_rate_minutes`,
        default 60 min) — rate-based and self-scaling, not wall-clock. Replaced the old
        unused `water_refill_max_runtime_minutes` helper with the two new ones.
      - **Solcast PV array spec resolved with real numbers**, sourced from the 2021
        install test report + 2024 battery-upgrade CoC PDFs: 24× JA Solar 430W panels
        (documented **"north facing"** at install), 2× SunSynk 5.5kW hybrid inverters,
        2× 15kWh Freedom Won batteries. User confirmed the broken panel is rated 435W.
        Recommended Solcast portal correction: azimuth 41°→0° (matches "north facing";
        Solcast's own site editor UI labels 41° as "North-West", confirming the
        mismatch), capacity_dc 10.8kW→**9.89kW** (23 working × 430W). Full spec now in
        this file's Hardware Summary "Solar PV Array" entry; POWER_CONTRACT Issue 28
        updated with the same numbers.
      - **BUG-S61 (Telegram photo) root-cause theory retracted.** User confirmed local +
        public + VPN access via `ha.dunners.tech` all work correctly — the
        "resolves to 10.10.1.5, nothing listening" diagnosis was stale. Live DNS lookup
        this session resolves correctly to Cloudflare. Told explicitly to stop guessing;
        this bug needs a fresh live diagnostic (real error text from an actual
        occurrence) next time it happens, not more theorizing from old notes.
        SECURITY_CONTRACT.md BUG-S61 updated to record the retraction.
      - **`ha core check` — user ran it, passed.** Closes the one open item from the
        prior session's checklist (functional verification in the live HA UI).

- [x] **2026-08-21 — Pool pump dashboard ghost-entity cleanup + prioritized doc/code audit
      pass across power + water (7 issues closed, 1 downgraded, 3 confirmed-already-fixed
      doc-drift corrections).** User flagged `sensor.pool_pump_solar_headroom` showing
      "Unavailable — no longer provided by the template integration" on the Pool Pump
      Status card. Root cause: this entity was supposedly removed from
      `core.entity_registry` back in the 2026-04-22 Session D cleanup (renamed to
      `sensor.solar_available_surplus`), but the registry removal didn't stick and two
      `.storage/lovelace.dashboard_operations` cards still referenced the old name (an
      entities card + a 24h history-graph card). Both fixed to
      `sensor.solar_available_surplus`; user then deleted the stale registry entry.
      **Follow-on cross-check of PROJECT_STATE.md against the domain contracts surfaced
      a prioritized fix list; user actioned it same session:**
      - **Confirmed already-fixed (stale doc/TODO status corrected, no code needed):**
        pool pump's Tuya `continue_on_error` safety net (this file's "RESUME-HERE" Tuya
        entry below — `pool_pump_solar_control` got it 2026-08-09, entry hadn't been
        updated); WATER_CONTRACT Issue 2 (`water_tank_full_notification` — already
        triggers off `binary_sensor.water_tank_full_depth`, not the nonexistent
        `sensor.water_state == 'full'`); POWER_CONTRACT Issue 8
        (`group.house_security_power_sensors` confirmed to have 1 real member, not
        empty); network dashboard AP-button restart fix (confirmed live — single
        consolidated button card + "Total Known Clients" line both present).
      - **Code fixes shipped:** WATER_CONTRACT Issue 10 (dual-write duplicate Telegram
        on emergency refills — removed the direct `notify.send_message` calls in both
        CRITICAL branches of `water_tank_refill_control.yaml`; `script.notify_water_event`
        already Telegram-mirrors severity:warning unconditionally, so this was 2
        messages per emergency event — see note below re: the now-obsolete "accepted
        exception" in WATER_CONTRACT's Locked Design Decisions table); Issue 9
        (re-enabled the commented-out 1-min debounce on the max-depth safety stop in
        `water_safety.yaml` — confirmed it's an independent backstop, not covered by the
        separate stop-at-target-depth automation); Issue 12 (`water_refill_never_reached_full`'s
        condition was tautologically always-true — `sensor.water_state` can never equal
        `'full'` — now checks `binary_sensor.water_tank_full_depth` instead); Issue 11
        (deleted `water_policy_helpers.yaml` outright — 4 orphaned input_numbers, zero
        references anywhere in packages/ or dashboards, confirmed via grep). POWER_CONTRACT
        Issue 7 (deleted the dead `sensor.inverter_today_energy_import` doubling-workaround
        block from `power_core.yaml` — confirmed zero consumers anywhere in the repo; the
        contract's claimed consumer, `solar_used_by_house_today`, actually reads different
        source sensors. The real authoritative daily grid-import figure is
        `sensor.grid_energy_import_today`, a UI-defined `utility_meter` helper — not YAML —
        wrapping an `integration` sensor off inverter 1, per user). POWER_CONTRACT Issue 17
        (layering violation — moved the `group:` block, 11 inverter comparison groups, and
        `template:` sensor block, ~30 combined-inverter sensors + 1 throttled-sensor block,
        out of `power_helpers.yaml` into `power_templates.yaml`'s existing single
        `group:`/`template:` keys; verified via YAML parse — 20 groups, 37 sensors, zero
        duplicate `unique_id`s, exactly 2 template list items, same as before the move.
        `power_helpers.yaml` now contains only `input_boolean`/`input_select`/
        `input_number`/`input_datetime`, matching the `*_helpers.yaml` convention).
      - **Reviewed, decision made:** Program 4 SOC target (90%→100%, tested since
        2026-08-04) — recorder data pulled (`sensor.inverter_battery_soc`,
        `sensor.grid_energy_import_today`, 2026-07-28 to 2026-08-21): morning-floor SOC
        rose from a ~30-45% pre-test baseline to ~45-55% at 100%, and daily grid import
        fell to 1.6-7.7 kWh/day from ~08-15 onward (vs. a rough 10-39 kWh/day spread
        before). **Decision: keep 100%.** Caveat: this window also contains a tree-felling
        event (~mid-August, improved solar/reduced shading, not yet in any doc — see
        Claude's memory file) that the 08-10 to 08-14 high-grid-import days and the
        post-08-15 improvement both plausibly reflect, so the 100%-target effect and the
        felling effect aren't cleanly separable from this data alone.
      - **Reviewed, external action (not a repo change):** Solcast site geometry
        (`solcast_solar/solcast-sites.json`) — capacity 10.2kW/10.8kWdc, **azimuth 41°**,
        tilt 26°, loss_factor 0.9. A 41° azimuth doesn't match a "north-facing" roof
        description (true north ≈ 0° in Solcast's convention) — flagged as a likely
        contributor to POWER_CONTRACT Issue 28's persistent forecast-overshoot bias,
        independent of the `estimate10` percentile fix already applied there. Recommended
        the user re-check/correct azimuth (and derate `capacity_dc` for the 1 broken
        panel) on the Solcast Rooftop site portal directly — this config lives on
        solcast.com, not in this repo; the local JSON is a read-only sync cache.
      - **Still open, needs the user, not re-flagged as a bug:** WATER_CONTRACT Issue 5
        (max-runtime cutoff) — user questioned whether it's redundant given the existing
        depth hard-stop (1.95m) + 15-min no-rise protection. Assessment: not fully
        redundant (covers a 3rd failure mode — slow-but-continuous fill that trips
        neither guard) but genuinely lower priority than originally scored; downgraded
        P2→P3, no automation added pending the user's call. Telegram inline-photo
        delivery (SECURITY_CONTRACT BUG-S61) — still broken, root cause needs the user's
        cloudflared tunnel ingress target (not discoverable from this repo) before a fix
        can be proposed.
      **Not yet run:** `ha core check` / Developer Tools → YAML → Check Configuration —
      this session validated the changed water/power YAML files with a local YAML parse
      (syntax + no duplicate top-level keys/unique_ids confirmed) but that is not a
      substitute for HA's own config check. Run it, then reload Template Entities +
      Automations (no core restart needed — no automation trigger/id changes, no new
      integration entities) before treating these live.

- [x] **2026-08-18 — Cancel Alert rolled out to every critical alert domain, BUG-A19 fixed.**
      User asked to audit whether every critical alert has a way to cancel its own escalation,
      using the garage-door "left open" critical push as the example. Investigation found the
      garage-door case itself was already covered (BUG-A13/BUG-A18 — the doors/gates domain has
      a working "Cancel Alert" button on phone + Telegram, per-cycle mute via
      `input_boolean.gate_alert_snoozed`, auto-reset once `sensor.door_alert_context` returns to
      normal). But auditing every other domain's live repeat-reminder automation found **none of
      the other 11 had any snooze/cancel mechanism at all** — the `alert.*` entities' native
      Acknowledge button is a dead end (their `notifiers:` route through the dead `STD_Alerts`
      group, and the real delivery automations don't read the entity's acknowledged state
      either), so a false-positive critical repeat in power/water/temperature/etc. could only be
      stopped by waiting out the underlying condition or flipping that domain's notify toggle
      off globally (silencing genuinely new events too). User chose to roll the BUG-A13 pattern
      out to all domains in one session rather than a partial rollout. **Fix:** extended
      `notify_power_event`/`notify_water_event`/`notify_presence_event` with the same
      `actions`/`telegram_action` passthrough `notify_security_event` already had (`notify_system_event`
      already had `actions`, gained `telegram_action`); all four Telegram sections switched from
      `notify.send_message` (can't carry `inline_keyboard`) to native `telegram_bot.send_message`.
      Added a per-cycle `input_boolean.<x>_alert_snoozed` + condition gate + Cancel Alert button
      (phone action + Telegram `/cancel_*`) + cancel-handler automation + auto-reset-on-clear
      automation to every repeat-reminder stream: Power (1), Water (3: tank/safety, borehole
      fault tier-2, borehole critical fault tier-3), Temperature (4: WAN/LAN/device/storage),
      Device Power (1), Media (1), Network (4: device down, WAN down, WAN degraded, device
      restart), Security (1 — also retires the ad-hoc global-mute workaround the repeat reminder
      had been using), Dash Batteries (1), Presence (1), Garden (1 — alongside the existing Turn
      Off Pump button). Camera Health audited and left alone — its `alert:` `repeat: [60,240]`
      has no live notifier at all (BUG-A11 removed the dead one, no replacement repeat automation
      was ever built), so there's nothing repeating to cancel; flagged as a separate, smaller gap.
      **Validated:** `POST /api/config/core/check_config` → `{"result":"valid"}` after every one
      of the 15 touched files (4 notify scripts + 11 alert files). Live-reloaded `automation`,
      `script`, and `input_boolean` domains via the Supervisor-proxied Core API — no restart
      needed (no `alert:` entity definitions were touched). Spot-checked several new entities
      live post-reload, all present and correctly `off`/`on`. **Not yet live-verified:** an
      actual notification tap on each new Cancel Alert button end to end — the pattern is a
      direct structural copy of the already-proven BUG-A13 implementation, but a real tap on
      every domain hasn't been individually confirmed yet. **Docs updated:** ALERTS_CONTRACT.md
      (BUG-A19 full writeup, Section 9 domain table, Section 10 issues table — also backfilled
      the missing BUG-A18 row there).

- [x] **2026-08-18 — WAN Degraded notification storm investigation, BUG-NET10/11 fixed
      (⚠️ requires HA restart — lovelace + input_boolean changes).** User reported being
      spammed with "Network Alert — WAN Degraded" critical pushes overnight and into the
      evening, suspecting Microsoft DNS specifically, and said toggling the notify helpers
      off hadn't stopped it. Live-traced via the recorder DB (not doc guesswork): 9 separate
      "degraded" episodes in a 6-hour window alone, each a smooth ~90s-7min score sawtooth
      from 100 down to 0 and back. **Root cause was not Microsoft** — in every episode
      `sensor.unifi_gateway_google_wan_latency` spiked alone to 100-144ms while Cloudflare
      and Microsoft stayed at 2-4ms throughout. Two compounding bugs: (1) **BUG-NET10** —
      `binary_sensor.wan_degraded_alert_active` had no anti-flap delay at all (every sibling
      binary sensor in the file requires 250s sustained; NETWORK_CONTRACT.md had already
      documented a `for: minutes: 5` that was never actually implemented) — a single
      elevated ping on just one of the 3 WAN targets flipped it on instantly, then the
      `value_max` statistics sensors (20-sample/5min window) held that one spike as the
      reported max until it aged out, stretching a few-second blip into a multi-minute
      "critical" alert. Fixed with `delay_on: minutes: 5`. Also confirmed the user's
      architectural instinct was right in principle: `wan_health_score`'s latency term uses
      the **max** of the 3 targets, so one bad target with the other two healthy can still
      swing the composite score to 0/critical — left as-is pending more data (a sustained
      5-min bad reading on even one target may be a real issue), but the notification now
      states which target and its raw ms value so this is no longer a mystery on the phone.
      (2) **BUG-NET11** — all 4 network notify toggles (`network_device_down_notify`,
      `wan_down_notify`, `device_restart_notify`, `wan_degraded_notify`) had `initial: true`
      — the same restart/reload-reset gotcha as BUG-PWR-ORCHSOC01 — silently undoing the
      user's own attempt to turn off WAN Degraded notifications on the next restart or
      "Reload Helpers". Removed `initial:` from all four. **Also found + fixed the BUG-N18
      double-notification pattern for the network file** (flagged, not fixed, in the
      2026-08-18 water session below): `alert.network_alert` still carried
      `notifiers: STD_Alerts`, dead until 2026-08-09 (BUG-N16) then silently became a
      second push channel alongside the working `route_*_alert` automations, doubled again
      by `force_network_alert_retrigger_on_escalation`'s off/on retrigger re-sending the
      alert's own initial push each time — removed. **Dashboard gaps found + fixed:**
      `input_boolean.wan_degraded_notify` was missing from the "Alert Notifications
      Control" card entirely (the other 3 network toggles were there) — likely why the
      user couldn't find/confirm the toggle; added it. Also found **no `alert.*` entity of
      any domain is on any dashboard** — tapping the `binary_sensor.*_alert_active` rows
      that are on dashboards has no Acknowledge action, only the `alert.*` entity itself
      gets HA's native Acknowledge button. Added `alert.network_alert` to the same card as
      a first fix; the same gap almost certainly exists for every other domain and is
      flagged for a future session, not fixed here. `ha core check` valid after all YAML
      edits. **Docs updated:** NETWORK_CONTRACT.md (BUG-NET10, BUG-NET11, Section 6).

- [x] **2026-08-18 — Borehole no-rise fault investigation + 4-bug water/notifications fix
      (Issues 16-19, BUG-N18).** User asked what happened with the automated borehole refill
      that morning and why "Refill Status" showed blank. Reconstructed the full timeline from
      the recorder DB (`home-assistant_v2.db`, states+context, not doc guesswork): pump auto-on
      08:56:01 (`water_state=safety` → Branch 1 emergency bypass); `water_borehole_no_rise_protection`
      hard-stopped it 09:11:01 after exactly 15:00 with zero depth rise
      (`sensor.water_tank_depth_rate` < 0.01, depth flat 0.28-0.33m) — **confirmed real fault, not
      a false trigger:** `sensor.borehole_pump_power` held a steady ~1200-1350W / 6.2-6.8A the
      entire 15 minutes (~0.31 kWh burned for zero tank gain), so the pump was genuinely running
      against no flow, not idling. Pump restarted 09:12:02 — `context_user_id_bin` on that state
      row resolves to Ryan Dunnington in `.storage/auth`, confirming a direct manual switch flip
      (not one of the `water_manual_borehole_run_*min` scripts — `water_refill_manual_run` never
      went `on`), and it caught successfully (`water_state` safety→low by 09:26:40, still filling
      normally as of investigation). **4 bugs found and fixed, all live-tested with `ha core
      check` (passing):**
      1. **Issue 16 (`water_protection_automations.yaml`):** `input_text.borehole_last_fault` was
         only ever written by the disabled dry-run automation — the live no-rise automation never
         wrote it, so every fault notification for months quoted a stale 2026-02-18 fault. Added
         the same write action to `water_borehole_no_rise_protection`.
      2. **Issue 17 (`water_helpers.yaml`):** `sensor.water_refill_last_outcome`'s `else` branch
         defaulted to "Completed Normally" any time neither safety-abort nor manual-run flags
         were set — including mid-refill, since the abort flag clears the instant a new cycle
         starts. This is what the user saw as a false "complete" status while still actively
         refilling. Added a `water_refill_cycle_active` check → reports "Refilling…" instead.
      3. **Issue 18 (`.storage/lovelace.dashboard_operations`, ⚠️ required full restart):** the
         dashboard's "Refill Status" row read `attribute: reason` off
         `binary_sensor.water_refill_allowed` — an attribute that was never defined on that
         entity's template (only `grid`/`battery_soc`/`required_soc`/`borehole_enabled`/
         `orchestrator_state`/`conserve_blocked`/`last_sun_blocked` exist), so the field has
         always been blank regardless of refill state. Repointed to the existing
         `sensor.water_refill_blocked_reason`.
      4. **Issue 19 / BUG-N18 (`alerts_water.yaml`, ⚠️ required full restart):** `alert.water_alert`
         + the two borehole-fault alert entities notified via `notifiers: STD_Alerts` — dead when
         the 2026-07-06 `route_water_tank_alert`/`route_water_borehole_fault_alert_tier_*`
         automations were built as the working replacement (calling `script.notify_water_event`
         directly). `STD_Alerts` mobile delivery was fixed 2026-08-09 (BUG-N16) but the redundant
         `notifiers:` lines were never removed — confirmed the user got **3 separate pushes**
         for this one 70-second fault (route automation at 09:11:01, `alert.water_alert`'s own
         STD_Alerts delivery at 09:11:31, plus the independent first-fault-of-day notification,
         also 09:11:01). Removed `notifiers: STD_Alerts` from all 3 water alert entities, matching
         the existing `alerts_security.yaml`/BUG-A10 precedent (alert entity kept for
         dashboard/ack/`repeat:` only). **Flagged, not fixed:** the identical dead-STD_Alerts +
         parallel-route-automation pattern exists in 9 other alert files
         (temperature/doors/presence/device_power/power/media/batteries/garden/network) — all
         built during the same 2026-07-06 window, all likely double-notifying the same way since
         the 2026-08-09 fix. Logged as `NOTIFICATIONS_CONTRACT.md` BUG-N18 for a future
         cross-domain sweep; out of scope for this water-only session.
      **Docs updated:** `WATER_CONTRACT.md` (Issues 16-19 + entity table),
      `NOTIFICATIONS_CONTRACT.md` (BUG-N18). `ha core check` passed after all 4 edits.
      **Same-day follow-up (still 2026-08-18) — Issue 20, auto-retry added.** User pushed back
      on the "flat/no-rise = maybe dry" framing: net depth was actually -0.04m over the 15min
      run (declined, not flat), and `sensor.water_tank_consumption_rate` read 0.24-0.48 m/h the
      *entire* window (never zero) — real concurrent house draw, not a stuck sensor (raw vs
      validated depth matched exactly throughout; `water_tank_depth_sensor_stable`/
      `tuya_cloud_health` stayed on/healthy, no Tuya dropout). No automated irrigation valve
      exists in this system (`GARDEN_CONTRACT.md` — garden watering is manual/untracked) and
      `gardener_on_site` was off, so garden use specifically isn't provable, but ordinary
      household draw is entirely sufficient explanation — the no-rise protection only sees net
      depth change, with no way to separate borehole output from concurrent consumption. User
      asked for automatic retry (matching what a manual restart already proved fixes it) instead
      of always requiring a human. Added to `water_borehole_no_rise_protection`'s own action
      sequence (`water_protection_automations.yaml`) — urgency-scaled delay (2min if
      `water_state=safety`, 5min otherwise, both user-selected), capped at fault-counter `<=2`
      today (auto-retry stops at fault #3, which is also where the existing tier-2 alert already
      fires — user-selected cap), re-checks pump/master-switch/depth state before retrying,
      fires an `information`-severity notification before each retry. Deliberately scoped to
      this automation only, not the shared `water_refill_aborted_due_to_safety` flag broadly —
      battery-hard-stop and max-depth-stop are different failure modes. `ha core check` passed.
      Automations-reload only, no restart needed (unlike the Issue 18/19 restart-required
      changes above). `WATER_CONTRACT.md` Issue 20 added.

- [x] **BUG-A18 — garage-door critical escalation showed no image, then wrong camera — fixed
      2026-08-14/15.** User flagged a garage-door "Door/Gate Left Open" critical push (open
      38+ min overnight) with no photo. Root cause: `route_door_sustained_open_escalation`'s
      camera-selection logic (`alerts_doors.yaml`) only ever recognized the two Tier-1 gates
      (main gate, front security gate) — a Tier-2 entry door (garage/front door) escalating
      alone always fell through with no snapshot taken. **Fix:** added garage_door →
      `camera.cam05_inside_garage` and front_door → `camera.cam07_front_kitchen` branches,
      matching image logic. **Correction same investigation:** user pointed out cam05 faces
      into the garage interior, not the door — switched garage's camera to
      `camera.cam04_car_port_front` (outside, facing the door). **Also flagged, not a code
      bug:** `sensor.garage_door_sensor_battery` found `unavailable` continuously (twice,
      40min apart, in lockstep with the door sensor's own updates) — likely dying battery,
      plausible cause of the door reading stuck "open"; recommended physical check + battery
      replacement. `ha core check` valid, `automation.reload` confirmed live both times. Full
      writeup: ALERTS_CONTRACT.md BUG-A18.

- [x] **2026-08-14 — Power/solar/battery/prepaid 2-year system audit + BUG-PWR-DOCDRIFT01
      fixed.** User asked for a full audit across the recorder's full history (~18 months
      available) to peg performance against the real hardware/software timeline and confirm
      whether the June 2026 battery swap (Freedom Won → 3× Greenrich AF1600) and the pending
      panel/inverter swap actually helped. Timeline corrected against DB evidence: a
      2025-05-16 lightning event damaged the old battery BMS (inverter offline 2025-05-22→
      06-03, ran on a 5kW loan unit meanwhile, stable again 2025-06-05 — DB-corroborated,
      grid import 2.06× higher / solar 47% lower during the degraded window vs after);
      2025-09-09→13 roof maintenance disconnected the panels entirely (DB shows exactly 4
      days of 0.0 kWh production, resolving the earlier open question in
      `SOLAR_UPGRADE_ROI_2026.md` about whether that stretch was weather or an outage — it
      was neither, it was maintenance). **Notable finding, not yet explained:** the June
      2026 battery swap's year-over-year grid-import comparison (June/July 2026 vs 2025,
      same months) shows grid import *up* 31%/21%, not down — house load also grew (+12%/
      +6%) but not enough to fully explain it, and the window is confounded by the
      concurrent E1–E8 orchestrator rewrite (same week) plus the since-fixed
      ORCHSOC01 SOC-reset bug. Flagged as inconclusive rather than forced into a false
      payback number — needs re-checking once Aug/Sept data clears both confounds (see
      `/power-audit` command). **BUG-PWR-DOCDRIFT01 (fixed):** POWER_CONTRACT.md's Entity
      Reference table pointed at `sensor.prepaid_solar_savings_monthly`, which doesn't exist
      — corrected to the real entity, `sensor.solar_savings_this_month`. See POWER_CONTRACT.md
      Issue 29 and its Hardware/Cost History section (now includes panel specs — 24× 435W JA
      Solar, 1 damaged — and the battery-swap system cost, previously missing entirely).
      **New artifacts:** `private_docs/POWER_SYSTEM_AUDIT_2026.md` (full audit, private —
      financial/ROI detail lives here, not synced publicly),
      `private_docs/POWER_SYSTEM_PERFORMANCE_LOG.md` (recurring dated log), and
      `/power-audit` command (`.claude/commands/power-audit.md`) — a manually-run monthly/
      quarterly habit, not an automation, for extending this audit over time. A new "ROI /
      Audit" view was added to the existing `operations_debug` ("Debug") Lovelace dashboard.
      **Same-day follow-up (still 2026-08-14):** root-caused the grid-import increase —
      user confirmed the old Freedom Won units had 2 damaged cells (explains the non-linear
      SOC-collapse symptom, running damaged the whole 13-month gap); tested and rejected
      "divide the `statistics.sum` spikes by 2" (checked against the sensor's own raw
      `state` reading — wrong for anything but the smallest spikes), fixed via
      state-fallback instead (55 days corrected, monthly solar table now complete in the
      audit doc); found the P4 force-charge window (14:00-17:00) carries 35.5% of daily
      grid import in 2026 vs 2.9% in 2025 (~17× jump) — large enough to fully explain the
      post-swap increase on its own, and now **live-tracked** via two new automations
      (`p4_window_capture_1400_snapshot`, `p4_window_compute_daily_share` in
      `power_automations.yaml`) plus `input_number.p4_window_share_of_day_percent` and
      `sensor.p4_window_share_7d_mean` (power_helpers.yaml / power_statistics.yaml) — see
      POWER_CONTRACT.md's P4 entity reference addition. Also fixed the new Debug dashboard's
      Grid Import chart: `apexcharts-card`'s `statistics.type: sum` pulls the raw cumulative
      value, not a period delta — corrected to `type: change`.

- [ ] **REVIEW 2026-08-12 — BUG-PWR-FORECASTBIAS01: Solcast forecast runs ~38% hot on
      average, two mitigations applied, monitoring.** User noticed peak PV briefly hit
      4kW+ despite bad weather while the day's forecast (40.5 kWh) badly missed actual
      (~12.3 kWh, 30% ratio) — then asked why Solcast isn't combined with the live weather
      forecast, believing it already was. Pulled 91 days of recorder data: mean actual/
      forecast ratio **62%**, only 1/91 days landed within 85-115%, 87% of days under 70% —
      a persistent one-directional bias across every season sampled (mid-May–August), not
      weather noise. Confirmed `sensor.solar_weather_correlation` is misleadingly named — no
      live OpenWeatherMap signal feeds it at all; it's purely retrospective (today's + 7d
      actual-vs-forecast ratio), so it can't flag a bad day until same-day production data
      has accumulated (~midday). No genuine Solcast+weather-forecast fusion exists in this
      repo. **Applied:** (1) `select.solcast_pv_forecast_use_forecast_field` switched from
      `estimate` (P50) to `estimate10` (Solcast's conservative percentile) — immediately cut
      `forecast_today` ~31% (37.8→26.1 kWh) at the source, benefiting every consumer. (2)
      `sensor.house_energy_resilience_hours` and `sensor.prepaid_topup_strategy` — the two
      decision sensors found reading raw undamped forecast directly — now apply the same
      live `ratio_today` correction the P4 grid-charge evaluator already used since
      2026-07-01. Deployed live: `check_config` valid, `template` reloaded, confirmed both
      sensors picked up the change. **Action next session:** compare `estimate10` forecast
      vs actual over the following several days — if still running hot, the persistence/
      magnitude points to a genuine Solcast site-calibration issue (array capacity/tilt/
      azimuth/shading as configured in the Solcast site) rather than something the
      percentile switch alone fixes. **Not yet built:** a real prospective weather-fusion
      (e.g. derating the forecast using OWM cloud-cover before sunrise) — flagged, needs its
      own scoped design. See POWER_CONTRACT.md Issue 28.

- [x] **BUG-S74 — grounds_low_confidence push showed wrong camera name + 25min-stale wrong-
      zone image (trigger_camera propagation race) — fixed 2026-08-13.** User flagged a
      07:26 "⚠️ Activity in grounds" push whose "Camera:" field said Cam09-Back-Bedroom but
      whose photo (timestamp 07:00:55, 25min stale) showed the driveway/a car leaving — two
      separate mismatches in one notification. Queried `home-assistant_v2.db` directly (the
      classification/trigger-camera sensors themselves aren't recorded — diagnostic
      exclusion) and confirmed the real trigger was `camera.cam12_back_pond` at 07:26:21, a
      grounds-**rear** event; gate was closed, nobody arriving/departing/staff — low-risk,
      not an intrusion. Root cause: `sensor.security_trigger_camera` is a template sensor
      derived from `*_motion_valid` one hop removed and read `'none'` for a render cycle
      before catching up to cam12 — `zone_label` (security_logic.yaml) defaulted grounds
      events to `'grounds front'` off that stale read (wrong zone → wrong, stale image
      slot), and `notify_security_events.yaml`'s camera-name resolver fell through to the
      volatile global `security_last_motion_camera` tracker, which itself lost a race
      against its own writer's deliberate 1s snapshot-settle delay and still held cam09's
      value from ~10 minutes earlier. **Applied:** `zone_label`/`reason` in
      security_logic.yaml now fall back to a direct `*_motion_valid` scan of the grounds
      cameras when the derived sensor hasn't caught up; `security_capture_each_camera_
      motion` (security_automations.yaml) now writes the global camera-name tracker
      immediately (before its snapshot delay) instead of after. `ha core check` valid.
      **Follow-up same session — audited the rest of the pipeline for the same defect
      class, found + fixed 4 more open gates (BUG-S75): grounds-front/perimeter image
      slots had no unconditional-confidence write (BUG-S65's fix never extended past
      grounds-rear); the router's freshness check only ever covered the 3 inside zones,
      not grounds/perimeter; `gate_activity`/departure branches read `ipcam03_driveway_
      history` with no age check (flagged unaddressed in BUG-S72, never closed); the
      arming stop-guard blocked the camera-name tracker write for cam14/cam05/cam15, not
      just images. **Then user caught a live regression the same day** — a genuine
      front-perimeter push showed "Camera: Cam15-Passage": the BUG-S74 fix only handled
      `sensor.security_trigger_camera` reading `'none'`, not the more common case where
      it holds an unrelated camera (cam14/cam15 rank top of its global priority list and
      often active for ordinary at-home reasons, regardless of which zone actually fired).
      **Corrected:** `reason`'s `cam_s` no longer reads the global trigger-camera sensor
      at all — rewritten to scan only the cameras belonging to the zone this block already
      determined (mirrors `zone_label`'s own zone logic instead of a separate, zone-
      agnostic sensor). `ha core check` valid, live `template.reload`/`automation.reload`
      confirmed. Full writeup: SECURITY_CONTRACT.md BUG-S74 (+ correction) and BUG-S75.

- [x] **BUG-PWR-ORCHSOC01 — `orchestrator_target_soc_by_sunset` reset to 90 on every HA Core
      restart, not the Program 4 SOC test — fixed 2026-08-12.** User: "why do I keep having to
      reset target SOC to 100 as keeps defaulting back to 90?" First checked whether the
      Program 4 SOC test (see "REVIEW 2026-08-11" below) had regressed — it hadn't:
      `number.inverter_1/2_program_4_soc` confirmed rock-solid at 100 via recorder DB back to
      2026-08-04 16:09:37 SAST, zero drift, every restart re-polls 100 live from the inverter.
      The real culprit was a different, similarly-named entity — `input_number.
      orchestrator_target_soc_by_sunset` (feeds the 14:00–16:30 P4 grid-topup evaluator,
      actively used despite POWER_CONTRACT.md previously flagging it "future use" — also
      corrected). Its recorder history showed it snapping to 90 at almost the exact timestamp
      of every restart, even hours after being manually set to 100 (one case reverted the
      same evening it was set). Root cause: `power_helpers.yaml` defined it with `initial: 90`
      — a well-known HA `input_number` gotcha where `initial:` is re-applied on every
      restart/reload, overriding the last real value instead of restoring it. No automation
      in this repo ever writes to this entity (read-only everywhere referenced) — only the
      user's manual resets and this restart-reset behaviour ever touched it. **Fix:** removed
      `initial: 90` from `power_helpers.yaml` so normal restore-state behaviour applies.
      `input_number.reload` ("Reload Helpers") was tried first per CODING_STANDARDS.md but
      tested ineffective live (confirmed with a control entity — `last_updated` didn't move
      across two reload calls); fell back to a full HA Core restart (confirmed with user
      first). Verified live post-restart: entity's `initial` attribute now `null` (was
      `90.0`), state restored to `100` instead of resetting. See POWER_CONTRACT.md Issue 27.
      **Flagged, not yet done:** no sweep of other `input_number` helpers in this package for
      the same `initial:` pattern — only this one had confirmed symptoms.

- [x] **Log-triage session 2026-08-09 (user: "check logs for updated issues") — 5 live issues
      found and fixed, 1 partially mitigated, 1 deferred (no urgency).** All via direct
      Supervisor/core API access (`ha core logs`, `/api/repairs/issues/fix`,
      `config/entity_registry/update` websocket, `sqlite3 -readonly` against the live recorder
      DB) — no user-reported symptom, purely proactive log/repairs review.

      **1. BUG-N15 (NOTIFICATIONS_CONTRACT.md) — `omit` sentinel undefined, breaking every
      WARNING/CRITICAL security/system push's action buttons + non-critical Telegram, since
      the 2026.7.4→2026.8.0 core upgrade (2026-08-08 evening onward).** Fixed: `else omit` →
      `else []` at 19 call sites across `notify_security_events.yaml` /
      `notify_system_event.yaml`. `check_config` valid, scripts reloaded.

      **2. BUG-N16 (NOTIFICATIONS_CONTRACT.md) — `notify.STD_Alerts` group (feeds 16 `alert:`
      defs) still dead, hourly `ServiceNotFound` — 2026-06-28's fix only covered the other 3
      STD_* groups.** User pushed back on the initial diagnosis ("those aren't dead service
      names though?") — correctly, turned out the FIRST specific names proposed were wrong too;
      re-verified against live `GET /api/services` before touching anything. Fixed: corrected to
      the 4 verified-working `mobile_app_<device>` legacy services; `telegram_bot_5527` dropped
      (genuinely unreachable via this mechanism, no drop-in fix exists). Required a full core
      restart (notify groups can't hot-reload) — confirmed with user first. Live-tested post-
      restart: `ServiceNotFound` gone.

      **3. BUG-N17 (NOTIFICATIONS_CONTRACT.md) — Vicky missing every warning/critical push for
      3+ weeks, dead mobile_app registration (re-registered 2026-07-18, YAML never updated).**
      Two-stage fix: interim retarget to the then-live `iphone13promax_vicky1`, then — after
      user had Vicky delete + reconnect the Companion App — the fresh registration landed clean
      (`iphone13promax_vicky`, no suffix) on the service name itself; 2 entities
      (`device_tracker`/`notify`) snagged a leftover naming-collision "1" suffix from the
      deletion race, renamed both via the entity registry once the clean names were confirmed
      free. All 6 YAML call sites (security/water/power) now point at the final clean name,
      live-tested. See Section 3A note in NOTIFICATIONS_CONTRACT.md for the "don't trust this
      name without re-checking" caveat this saga earned.

      **4. BUG-CORE03 (INFRA_CONTRACT.md) — `repairs.backup.automatic_backup_failed`, root
      cause found, partially mitigated.** Today's 04:50 native HA backup failed: `Could not
      lock database within 30 seconds`. `home-assistant_v2.db` = 216M rows / 37.9GB, dominated
      by power/UniFi sensors updating every 1–5s at the global 90-day `purge_keep_days`. User
      wants the full 90-day window kept for power/solar specifically (real analysis need) — so
      added a separate `recorder.purge_entities` automation trimming only
      `unifi_*`/`udm_*`/`wan_*` to 30 days, independent of the global purge. Honestly flagged to
      user (and here): this is ~7% of total rows and doesn't reduce write *rate* at backup time,
      so it may not fully prevent recurrence — the real lever if it recurs is reducing
      `scan_interval` on the noisiest power sensors, not touched this session.

      **5. BUG-CORE04 (INFRA_CONTRACT.md) — 3× `template.composite_device_id_*` repairs, FIXED;
      `http.deprecated_yaml` repair, deferred.** Both surfaced from the same 2026.8.0 upgrade
      (device-registry migration). The 3 composite_device_id issues (dangling device_id on 3 UI
      template helpers) fixed by driving the repair's fix flow directly via
      `POST /api/repairs/issues/fix` — no YAML involved, `.storage`-only. `http.deprecated_yaml`
      left alone — `breaks_in_ha_version: 2027.2.0`, no urgency, and touches the reverse-proxy
      trusted_proxies config which needs live UI verification to change safely.

      **6. Dashboard graph audit (no dedicated contract — logged here only).** Per user request
      ("check all graphs... for longer than 3 months"), scanned all 5 `.storage/lovelace.*`
      dashboards (111 graph-type cards). Confirmed `statistics-graph` cards and
      `custom:plotly-graph` cards with `"statistic": "sum"` set are already correctly wired to
      long-term statistics (verified live against the DB: `sensor.inverter_load_power` stats go
      back to 2025-02-17, `sensor.house_power_losses` to 2026-03-12 — both unaffected by any
      purge setting). Found 7 `custom:plotly-graph` cards using raw history at 30–31 day windows
      (fine today, fragile long-term) across `dashboard_operations`/`dashboard_testing`/
      `operations_debug` — added `"statistic": "sum", "period": "day"` to the 22 eligible entity
      fields, matching the pattern the dashboards' own "Monthly Production" cards already prove
      works. 2 entities (`sensor.prepaid_drift_rate_per_day`, `sensor.prepaid_drift_percentage`)
      have a `statistics_meta` row but neither mean nor sum enabled — genuinely not fixable this
      way, left on raw history, flagged to user. `input_number.prepaid_month_fixed_paid` not
      statistics-eligible at all (helper, not a sensor) — also left as-is, flagged. Backed up
      the 3 lovelace files before editing; required a full restart (`.storage/lovelace` changes
      per CODING_STANDARDS.md); `check_config` valid, clean boot.

      **Commits:** `105d55d6`, `8b34ff7a`, `bbf07162` (this session's own commits — two other
      commits landed on `master` interleaved with these from a concurrent session/agent working
      the same repo at the same time, `35610658` and `0cdf739a` — not this session's work, not
      described here, see their own commit messages / doc updates).

- [ ] **RESUME-HERE — Tuya "sign invalid" is a house-wide, 34h+-running
      command-signing fault, not a geyser-specific issue (E10+E11,
      BUG-INFRA-TUYA01) — 2026-08-06/07.** User: "Tuya app credentials
      expired... did auth them last night... but nothing turned on this
      morning," then "also no alerts at all this morning for no run."

      **E10 (morning incident).** Live investigation (Supervisor API
      history/logbook replay) found `sensor.tuya_cloud_health` went stale
      twice overnight, the second window (~02:57–06:57 SAST) covering the
      entire scheduled morning turn-on sequence (5 attempts). E9's
      stale-feedback branch silently reloaded on every attempt instead of
      alerting, assuming "stale == command probably landed" — wrong this
      time (`sensor.geyser_heat_pump_power` stayed flat 0.0W throughout,
      confirming a real expired Tuya OAuth token, not just dead MQTT
      feedback with REST still working). Zero critical alerts reached the
      user across 5 real failures. User fixed it with a manual run + fresh
      Tuya QR reauth ~08:07 SAST (matches the Tuya config entry's
      `modified_at` exactly — not the previous night as believed). **Fix:**
      `geyser_verified_turn_on`/`_off` now baseline
      `sensor.inverter_load_power` and cross-check real power/load evidence
      before trusting a stale-feedback verdict — only genuine evidence
      suppresses the alert. 700W threshold verified against 3 real ramp-up
      curves (crosses within 1.5–4.5 min every time, well inside the ~10-min
      check point).

      **E11 (root cause of the midday failures, found via UI script
      traces).** Same afternoon, 4 midday turn-on attempts
      (11:00/11:30/12:00/14:00, ~6.4kW PV available) all failed silently —
      each completed in <1s, no alert. Pulled actual script traces via the
      HA websocket API (no REST equivalent; used a throwaway venv +
      `websocket-client` against `ws://supervisor/core/websocket`,
      `trace/list` + `trace/get`) and found the real cause:
      `switch.turn_on` was throwing an **uncaught exception** —
      `network error:(-9999999) sign invalid` — on the very first action,
      crashing the whole script (`script_execution: "error"`) before it
      ever reached the retry/evidence/alert logic from E10. Confirmed 4
      occurrences over 2 days this way (3 turn-on today, 1 turn-off the
      evening before), zero alerted. **Fix:** added `continue_on_error: true`
      to all 4 `switch.turn_on`/`switch.turn_off` calls (initial + retry,
      both scripts) — a hard Tuya API error now flows into the same
      wait/retry/evidence/alert path instead of crashing silently.
      `check_config` valid, `script` domain reloaded, both scripts idle.

      **Scope is bigger than geyser — confirmed via user-supplied HA Logs
      screenshot.** Filtering HA's own error log for "sign invalid" shows:
      this has been happening since **2026-08-05 04:00:00 — 34+ hours**, not
      just 2026-08-06 (14 occurrences at the `geyser_turn_on_morning_midday_evening`
      automation level, 11 at the `geyser_verified_turn_on` script level, 2
      more on `geyser_verified_turn_off` at the 2026-08-05 21:00 evening
      hard-off). **And it hits Pool Pump too** — "Pool Pump — Solar-Aware
      Daily Control: Choose at step 1: choice 7... sign invalid," 14
      occurrences since 12:00:01 — a completely different automation/device
      (`switch.pool_pump_switch`, `power_automations.yaml` id
      `pool_pump_solar_control`), confirming this is a **Tuya Cloud
      command-signing fault at the integration level**, not anything
      specific to the geyser scripts' design. `pool_pump_solar_control` has
      NOT been given the same `continue_on_error` treatment — flagged, not
      yet fixed, pending user confirmation before touching a different
      domain's automation.

      **Positive data point:** the evening of 2026-08-06 ran cleanly with no
      errors — on by 16:18 SAST (ahead of the 17:00 check), at-temperature
      by 19:06 SAST (163 min heat-up), through Sports Night, off at 22:00
      SAST with confirmed state + notification, 4.46 kWh delivered. So the
      underlying fault is intermittent, not a hard-broken integration —
      consistent with the 3 stale/reload cycles seen this week rather than a
      single persistent outage.

      **Still genuinely open — not resolved:**
      - [ ] Root cause of "sign invalid" itself is unknown — candidates not
            yet checked: host clock/NTP sync vs. Tuya's server (Tuya signs
            requests with a timestamp; `timedatectl`/`chronyc` weren't
            reachable from this session's shell), or the frequent
            `tuya_reload_and_verify` auto-reloads themselves corrupting
            signing state (ties to the user's recalled "6 second reload"
            mechanism, still not located in this repo's history).
      - [x] `pool_pump_solar_control` given the same `continue_on_error` safety
            net — done 2026-08-09 (see that automation's header comment in
            `power_automations.yaml`), confirmed still in place 2026-08-21.
            Does NOT add geyser's fuller evidence/retry wrapper, just stops a
            Tuya API exception from silently killing the automation. The pond
            pump (Sonoff, not Tuya) is unaffected — this fault is Tuya-specific.
      - [ ] `continue_on_error` only makes failures visible again; it does
            not fix why Tuya is rejecting the signature. Real fix is
            upstream — likely needs a genuine Tuya re-auth or a deeper look
            at `custom_components/tuya`'s token/signing refresh, not another
            HA-side patch.

      See INFRA_CONTRACT.md BUG-INFRA-TUYA01 (E10/E11), POWER_CONTRACT.md
      Issue 26 (E10/E11 follow-up).

      **E12 update (2026-08-09) — both open root-cause candidates checked,
      both ruled out; pool pump given the same safety net.** Host clock is
      confirmed NTP-synced (Supervisor `/host/info`: `dt_synchronized: true,
      use_ntp: true`) — rules out clock/signing-timestamp drift. The only
      scheduled auto-reload (`tuya_cloud_stale_alert`) only fires after 4h+
      of stale feedback, nowhere near a "6 second" cadence — rules out our
      own reload frequency as self-inflicted (the user's recalled "6 second
      reload" mechanism still wasn't located anywhere in this repo).
      Confirmed this house runs HA core's **built-in** `tuya` integration
      (not a custom component — only unused `localtuya` lives in
      `custom_components/`), so the fault is most likely upstream in HA
      core's own Tuya cloud client; core 2026.8.1 is available (installed:
      2026.8.0), not confirmed whether it touches Tuya signing. Applied the
      previously-flagged follow-up: `continue_on_error: true` added to all 7
      `switch.turn_on`/`switch.turn_off` call sites in
      `pool_pump_solar_control` (`power_automations.yaml`) — same scope as
      the geyser E11 fix (stops a silent script crash from swallowing the
      alert), not a full verified-turn-on rebuild, since this automation has
      no separate verification script to extend evidence-checking into.
      Deployed live: `check_config` valid, `automation` domain reloaded,
      confirmed `automation.pool_pump_solar_aware_daily_control` state `on`
      post-reload. **Still genuinely open:** the actual upstream cause of
      "sign invalid" remains unconfirmed — next real recurrence likely still
      needs a genuine Tuya re-auth, or watching for a fix in a future HA
      core release. See INFRA_CONTRACT.md BUG-INFRA-TUYA01 (E12),
      POWER_CONTRACT.md Issue 26 (E12).

- [x] **Force charge had no recovery path if HA restarted mid-cycle
      (BUG-PWR-FORCECHARGE01) — fixed 2026-08-09, found while investigating the
      user's restart concern from the Program 4 SOC review above.** Traced
      `input_text.force_charge_saved_charging` (the `force_charge_batteries` →
      `force_charge_restore` P1-P6 snapshot) and found it's read "unknown" across
      every restart in the recorder's history back to 07-31 — but also unknown
      *between* restarts, meaning `force_charge_batteries` has apparently never
      actually been invoked in this house, so the snapshot's restore-across-restart
      behaviour was genuinely untested, not confirmed working. The real gap:
      nothing re-triggers recovery if a restart lands while `force_charge_active`
      is still on (mid-cycle, target not yet reached) — `force_charge_monitor`'s
      SOC-reached trigger does re-evaluate at startup, but only self-heals once SOC
      climbs back to target, which could be hours away (or never, on a poor solar
      day), leaving `inverter_programme_auto_enabled` off and all 6 programs stuck
      on Grid charging silently in the meantime. **Fix:** new
      `force_charge_recovery_on_restart` automation (`power_automations.yaml`) —
      triggers on `homeassistant: event: start`, and if `force_charge_active` is
      still on, waits 30s (let Solarman finish its post-boot poll) then calls
      `script.force_charge_restore` directly rather than waiting on the SOC
      trigger. `force_charge_restore`'s own guard already handles a missing/corrupt
      snapshot safely (unconditionally clears `force_charge_active` and re-enables
      `inverter_programme_auto_enabled` before the guard, added in the 2026-06-19
      fix), so this is safe either way — the automation always sends a
      warning (snapshot survived, auto-restored) or critical (snapshot lost, check
      the Solarman app manually) alert so a restart-interrupted force charge is
      never silent again. Deployed live: `check_config` valid, `automation` domain
      reloaded, confirmed `automation.force_charge_recovery_after_restart` state
      `on`. Not yet live-tested against a real mid-cycle restart (none occurred
      with `force_charge_active` on during the fix session). See POWER_CONTRACT.md
      §"Force Charge" for the script pair this extends.

- [x] **Garage light didn't turn on for a real 21:43 arrival; front security light's
      15-min auto-off looked odd but wasn't — investigated live, found a 4-month-old
      self-inflicted Sonoff reload storm (BUG-A17/BUG-L19) — 2026-08-05.** User: "why
      didn't garage lights turn on when came home after 9:30? front security lights came
      and turned off after time — seems need to optimise?" Investigated via Supervisor
      REST API history + logbook replay of the actual 2026-08-04 21:43 arrival (not
      re-derived from docs). Front security light: **not a bug** — `lighting_gate_open_assist`
      turned it on at gate-open (21:43:46), then `lighting_arrival_night.yaml`'s documented
      15-min bedtime-gated auto-off turned it off at 21:59:14, exactly as designed. Garage
      light: **real bug.** Gate opened 21:43:45 → `lighting_gate_open_assist` tried
      `switch.turn_on` on `switch.garage_light`; `garage_occupied` went on at 21:44:07 →
      `lighting_garage_smart_control`'s presence branch also tried. Both calls landed inside
      a window (21:40:50–21:45:15) where `switch.garage_light`'s physical Sonoff device was
      `unavailable` — both silently no-opped, and neither automation retriggers once a
      device recovers (edge-triggered, already fired). **Root cause:**
      `binary_sensor.garage_door_stale` (`alerts_doors.yaml`) was defined as "door sensor
      state hasn't changed in >300s" — true almost constantly for a door that legitimately
      sits open/closed for hours — which fired `automation.recover_sonoff_if_stale`
      (`homeassistant.reload_config_entry` on the whole Sonoff integration) roughly **every
      6 minutes, 24/7, since 2026-03-31** (~4 months, confirmed via daytime history too, not
      just night), briefly dropping every Sonoff entity in the house each cycle. Normally the
      reconnect is ~4s and invisible; this time the garage 3-gang device took ~4.5 min, and it
      happened to overlap the arrival. **Fix:** (1) `garage_door_stale` now requires the
      entity to actually report `unavailable`/`unknown` for >300s, not just "hasn't toggled" —
      stops the reload storm entirely. (2) Added a verify+retry safety net to both
      `lighting_gate_open_assist` and `lighting_garage_smart_control`: wait 3s after
      `switch.turn_on`, retry once (with a logbook warning) if the target didn't confirm
      `on` — covers any future transient device drop independent of fix (1). Deployed live:
      `check_config` passed, `template` + `automation` reloaded via Supervisor API, confirmed
      `binary_sensor.garage_door_stale` now reads `off` correctly. No restart required, no
      entity renames. See ALERTS_CONTRACT.md BUG-A17, LIGHTING_CONTRACT.md BUG-L19.

- [x] **BUG-INFRA-TUYA01 follow-up (E9) — geyser Tuya-stale alerts now self-heal
      immediately instead of just warning and waiting — 2026-08-05.** User: "geyser
      went off HA again today but alert didn't include the reload option?" Investigated
      via logbook replay: at 08:10 SAST, `geyser_verified_turn_off`'s stale-feedback
      branch (E8, added 2026-08-04) correctly detected `sensor.tuya_cloud_health` was
      stale and sent a reworded warning — but that warning routes through
      `script.notify_power_event`, which has no `actions`/button field at all (unlike
      `script.notify_system_event`, which is what carries the "🔄 Retry Reload" button
      on `tuya_health.yaml`'s own watchdog alert). Nothing actually reloaded the Tuya
      integration until the separate `tuya_cloud_stale_alert` watchdog hit its own 4h
      trigger 36 minutes later at 08:46 and fixed it. Rather than add button support to
      `notify_power_event`, the real fix is not making the user wait or tap anything:
      `geyser_verified_turn_on`/`_off`'s stale-feedback branches
      (`packages/power/geyser_automations.yaml`, E9) now call
      `script.tuya_reload_and_verify` directly the moment they detect stale feedback,
      instead of just alerting. The shared script's own notification (info on success,
      warning + retry button on failure) is now the only message for the event — the
      branches no longer send a separate `notify_power_event` in the stale case, to
      avoid a duplicate/conflicting alert. Deployed live: YAML validated, `check_config`
      passed, `script` domain reloaded via Supervisor API, confirmed live. Full writeup:
      INFRA_CONTRACT.md BUG-INFRA-TUYA01 (E9 update), POWER_CONTRACT.md Issue 26 (E9
      follow-up).

- [x] **Critical Sensor Health push notification had stray blank lines/indentation before the
      sensor list (BUG-A16) — 2026-08-04.** `alerts_system_health.yaml`'s critical-severity
      notification action re-derives its bad-sensor list inline in the `message:` template
      (same live cross-check as BUG-A14); the block tags had no `-` whitespace-trim modifiers,
      so each `set`/`for`/`if`/`endfor` line's own indentation/newline leaked into the rendered
      push text — message content was correct, but arrived as "...critical sensors:" followed
      by several blank lines before the actual list instead of one clean line. Fixed by adding
      `-` trim modifiers throughout; since full trimming alone would glue the list directly onto
      the preceding text with no separator, added an explicit `{{ " " ~ (...) }}` single-space
      prefix on the final expression. Confirmed live via the Supervisor API that the two
      pushes the user actually got weren't false positives — `switch.water_pressure_pump` has
      been genuinely `unavailable` since 12:05 SAST (hardware confirmed dead, not investigated
      further per user); the blank-line bug was hiding a real critical hit, not just a cosmetic
      annoyance. See ALERTS_CONTRACT.md BUG-A16.
- [x] **Tuya Cloud MQTT push thread died overnight, 4 false "geyser turn-on failed" critical
      alerts while it was actually on and heating (BUG-INFRA-TUYA01) — 2026-08-04.** User
      flagged from phone lock-screen notifications: 4 critical "🔴 Geyser — turn-on failed,
      still off" alerts between 04:10–~06:50 SAST, but the Tuya app showed the geyser
      connected, drawing power, and heating the whole time. Investigated live via Supervisor
      REST API + a direct recorder DB (`home-assistant_v2.db`) query (this session runs
      directly on the HAOS host, so both were reachable). Root cause: the Tuya integration's
      background MQTT push client (`tuya_sharing/mq.py`) died at 02:25:03 SAST — a reconnect
      attempt hit Tuya's cloud `API_QPS_LIMIT_OR_DEGRADE` and the exception propagated
      uncaught out of the worker thread, killing it with no auto-restart and no
      `unavailable` transition. Every Tuya entity (geyser switch, pool pump, pond pump,
      water tank depth/liquid sensors) froze at its last value simultaneously. Command
      delivery is a separate REST path and kept working — so the geyser's scheduled
      04:00/04:15/04:45/05:15/05:45 turn-on attempts genuinely reached the device, but
      `script.geyser_verified_turn_on` (BUG-PWR-GEYSER01's verification wrapper) could never
      see the confirming state change and raised 4 false criticals. **Confirmed the actual
      fix from recorder history, not guesswork:** a manual Tuya config-entry reload at
      09:28:16 SAST shows as all 5 entities going `unavailable` for ~4s then returning with
      live values (recorder stays up through a reload, unlike a full restart) — everything
      was healthy from 09:28 onward, hours before an unrelated 14:04 SAST restart (done for
      an update) that some earlier analysis had wrongly credited as the fix. **Built:** new
      `packages/core/tuya_health.yaml` — `sensor.tuya_last_activity_age` (most-recent update
      across all 5 Tuya entities, since no single one is a safe staleness canary alone) /
      `sensor.tuya_cloud_health` (healthy/delayed/stale, mirrors weather_api_health) /
      `automation.tuya_cloud_state_feedback_stale` (4h+ trigger → auto-reloads just the Tuya
      config entry via `script.tuya_reload_and_verify`, no full restart) / a "🔄 Retry
      Reload" mobile action button (`TUYA_RETRY_RELOAD`) for manual retry from the phone if
      auto-reload doesn't clear it. Also updated `geyser_verified_turn_on`/`_off` (E8,
      `geyser_automations.yaml`) to check `sensor.tuya_cloud_health` before declaring a
      genuine command failure — downgrades critical→warning and rewords the alert when
      feedback is already known stale. Side effect: copying `weather_api_health`'s
      healthy/delayed/stale pattern surfaced a real pre-existing bug there (BUG-WEA03, fixed
      separately same day — see INFRA_CONTRACT.md Part 4). Deployed live: YAML validated,
      `check_config` passed, `template`/`automation`/`script` reloaded via Supervisor API,
      all new entities confirmed live. Full writeup: INFRA_CONTRACT.md BUG-INFRA-TUYA01,
      POWER_CONTRACT.md Issue 26.

- [x] **FIXED 2026-08-09 — Network "AP Garage down" critical alerts fired from harmless ~2s
      UniFi reconnect blips; anti-flap bypassed (BUG-NET09) — found 2026-08-04.** User got 3 critical "Device(s) down: AP Garage
      Connected, AP Lounge Connected, AP Office Connected, AP Passage Connected, AP Bar
      Connected, ZenWiFi XD6 Connected" pushes in 40 min (14:02/14:07/14:12 SAST) while the
      UniFi console showed everything healthy (AP Garage uptime continuous at 15d+, never
      rebooted). Confirmed via state-history API: `group.network_devices` (all 5 APs +
      ZenWiFi) blipped `unavailable` for ~2 seconds each time then self-recovered — the
      signature of a UniFi integration/websocket reconnect, not a real outage (matches the
      same class of blip BUG-NET06 previously documented). The domain's real anti-flap gate
      worked correctly: `binary_sensor.network_device_down_alert_active` never reached `on`
      (its 250s `for:` requirement was never met) and `alert.network_alert` stayed `idle`
      throughout. **Root cause of the false pushes:** `automation.route_network_device_down_alert`
      (`alerts_network.yaml` ~line 722) has a *second* trigger —
      `sensor.network_device_down_alert_severity` transitioning to `critical`, with **no
      `for:` duration at all** — that bypasses the 250s anti-flap entirely. A full-group
      blip always computes `severity=critical` (6 devices down at once), so this trigger
      fires instantly on every such blip regardless of duration. Logbook confirmed all 3
      pushes fired via this exact trigger (`script.notify_system_event` called ~12-15ms
      after each blip started). **Fix (applied 2026-08-09):** added `for: "00:00:20"` to that
      trigger, matching the pattern already used on the sibling from/to trigger in the same
      automation. `check_config` valid, `automation` domain reloaded live via Supervisor API,
      confirmed `automation.route_network_device_down_alert` state `on` post-reload. Not yet
      live-verified against a real UniFi blip (none occurred during the fix session) — next
      ~2s multi-AP reconnect is the real test. See NETWORK_CONTRACT.md Section 6 + Section 8.

- [x] **REVIEWED 2026-08-21 (9 days late — target was 2026-08-11) — Program 4 SOC target
      test (90% → 100%), started 2026-08-04. DECISION: keep 100%.** See the 2026-08-21
      session entry at the top of this list for the full data pull and the tree-felling
      caveat.
      User asked to push Inverter Program 4 (14:00–17:00 window) target SOC from 90% to
      100% on both inverters, reasoning: coming out of winter, days getting longer/warmer,
      wants more midday solar banked into the battery for better overnight/morning grace
      before the batteries run down. `number.inverter_1_program_4_soc` and
      `number.inverter_2_program_4_soc` set to 100 live via Supervisor API call
      (`number.set_value`) — **not a YAML/packages change**, so it won't show in
      `git diff` and isn't covered by the E5 inverter-programme automation (confirmed no
      automation in `power_automations.yaml` overwrites `program_4_soc` based on season/
      forecast — only `force_inverter_sync`/force-charge save-restore touch it, and only
      on their own triggers). `select.inverter_1/2_program_4_charging` left on **Grid**
      (unchanged) — Program 4 already defaults to solar-only through the window and only
      falls back to grid to close the gap to target near 16:00 if solar is short, so the
      100% target raises the grid-topup ceiling too, not just solar utilization. Plan: run
      a couple of days at 100%, compare morning battery SOC / grid draw against the prior
      90% baseline, decide whether the extra grid-charging cost (on shortfall afternoons)
      is worth the added morning runway. **Action next session (target 2026-08-11):**
      pull a few days of `number.inverter_1_program_4_soc`/battery SOC history and
      grid-import-during-P4-window history, compare to the pre-2026-08-04 baseline, and
      either keep 100% or revert both `program_4_soc` entities to 90. See POWER_CONTRACT.md
      §7 (Strategy & Decision Layer) for the Program SOC entity reference.

      **Data pulled early 2026-08-09 (2 days ahead of the 08-11 target — decision still
      pending, not made unilaterally). First pass had two real errors, both corrected
      same session after user pushback:**

      1. **Miscounted which days were actually at 100%.** The `number.set_value` call
         that raised the target landed at **2026-08-04 16:09:37 SAST** (confirmed from
         the entities' own state-change history, not assumed from the date) — so 08-04's
         14:00–17:00 window ran at 90% for ~2h10m and only the last 51 min at 100%. First
         pass wrongly counted it as a full 100% day. Only **08-05 through 08-08 (4 days)**
         actually ran the whole P4 window at 100%; 08-09 still in progress at pull time.
         No actual reversion to 90% shows anywhere in the recorder history since — a live
         HA Core restart happened mid-session today (~15:40 SAST, cause unconfirmed, not
         triggered by this session) and `program_4_soc` came back correctly at 100% on
         both inverters afterward (read live from the inverter via Solarman, not a cached
         HA value).
      2. **Cooking-jump proxy was too narrow.** First pass used `sensor.house_kitchen_power`
         as an evening-activity signal — user correctly flagged that this sensor is only
         `sensor.main_fridge_plug_power` + `sensor.philips_airfryer_plug_power`
         (confirmed via `group.house_kitchen_power_sensors`), missing the oven/stove/
         microwave entirely since those aren't individually metered. Redone using
         `sensor.inverter_load_power − sensor.geyser_heat_pump_power` (whole-house load
         minus the geyser) as a broader evening-activity proxy, evening peak 17:00–21:00:

      | | n | ΔSOC (14→17h) | Grid import (P4 window) | SOC @ next 06:00 |
      |---|---|---|---|---|
      | 90% baseline, all 7 days | 7 | +17.0 pts avg | 4.08 kWh avg | 44.4% avg |
      | 90% baseline, low evening load (<5kW) | 5 | +16.2 pts avg | 4.30 kWh avg | 45.0% avg |
      | 100% clean, all 4 days | 4 | +21.0 pts avg | 7.73 kWh avg | 46.5% avg |
      | 100% clean, low evening load (<5kW) | 3 | +21.8 pts avg | 6.89 kWh avg | **50.5% avg** |

      On evenings with comparable household activity (excluding the one high-load evening
      in each bucket — 08-02/08-03 baseline, 08-06 test), the 100% group shows both more
      grid draw during the window **and** a meaningfully better next-morning SOC (+5.5 pts)
      than the 90% baseline — a coherent, intuitive result (spend more grid now, bank more
      battery, more overnight/morning runway), the opposite of first pass's "no benefit,
      revert" lean, which was mostly an artifact of the two corrected errors above.
      **Revised recommendation: keep at 100%** through the original 08-11 target for 2 more
      days to firm up the still-thin sample (n=4, only 3 low-load), rather than reverting.
      `program_4_soc` left untouched at 100 either way — this remains the user's call, not
      a bug fix.

      **2026-08-12 update:** user reported the target kept "defaulting back to 90" —
      re-checked `program_4_soc` specifically for this and it's still confirmed rock-solid at
      100 with zero drift since 08-04 (see BUG-PWR-ORCHSOC01 above). The actual reverting
      entity was a different, similarly-named helper (`orchestrator_target_soc_by_sunset`),
      now fixed — this A/B test's data and recommendation above are unaffected.

- [x] **Critical/intruder inside-zone notifications could carry an hours-stale image
      (BUG-S73) — 2026-08-04.** User flagged from a live phone screenshot: a "🚨 INTRUDER —
      INSIDE (main house)" push at 23:00 (`home: all`, `cam: cam14_lounge`) carried a photo
      timestamped 05:04 that same morning — 18 hours stale. Root cause, found via Supervisor
      API logbook replay of the actual event: (1) `security_capture_each_camera_motion`'s
      arming stop-guard (skips the whole automation, including the BUG-S47 unconditional-
      of-confidence image write, whenever `inside_cameras_armed`/`_passage_armed` is off)
      means every real cam14/cam05/cam15 motion event while the family is home never
      refreshes the zone image slot at all — BUG-S47's fix was never actually unconditional
      for that case. (2) RUNG 2.5 (stay-mode lounge intrusion) can fire `critical_intrusion`
      the instant `inside_cameras_armed` auto-arms at 23:00 bedtime, off a motion sensor
      that was already latched on from before arming — no new motion_valid edge fires the
      capture automation at that moment, so no fresh snapshot is taken either. Confirmed
      live: cam14 fired 22:54/22:58 (unarmed, writes skipped), critical fired at exactly
      23:00:00 with zero new capture-automation trigger in between. **Fix:** consolidated
      the router's two duplicated zone→slot image lookups into one; for the three inside
      zones only, it now checks the slot's embedded `?v=<timestamp>` age (<60s) and takes a
      real `camera.snapshot` fallback if stale — same live-snapshot pattern already used for
      BUG-S72 (arrival image lock). Perimeter/grounds slots untouched (not affected — no
      arming guard on those cameras). `security_automations.yaml` modified, config-checked
      valid via Supervisor API, `Reload Automations` applied live and confirmed firing
      cleanly post-reload. See SECURITY_CONTRACT.md BUG-S73 for full detail. **Not
      addressed:** the same arming stop-guard still skips dashboard history/timeline writes
      (`security_last_motion_camera`/`_image`, `security_event_session`) for cam14/cam05/
      cam15 while unarmed — only the notification image path was fixed.
- [x] **`sensor.weather_api_health` never actually equaled `"healthy"`/`"delayed"` in
      production — inline `#` comments leaking into rendered template state (BUG-WEA03) —
      2026-08-04.** Found as a side-effect of building `packages/core/tuya_health.yaml`,
      which had copied this exact pattern from `weather_core.yaml` (already fixed in the
      copy). The `healthy`/`delayed` branches of the Weather API Health template had
      trailing `# < 2 hours` / `# < 4 hours` annotations meant as comments — but since
      they sit inside a Jinja `>` folded block scalar fed to the template engine, YAML
      never treats `#` there as a comment; it's literal text rendered into the state.
      Confirmed live via Supervisor API: the entity's real state was the string
      `"healthy        # < 2 hours"`, never a clean `"healthy"`. This means BUG-WEA01's
      2026-07-10 "gap closed" claim (removing `weather_api_recovery`'s `not_from` guard)
      was necessary but not sufficient — that automation triggers on `to: "healthy"`,
      which could never match the dirty string, so recovery has never actually fired in
      production. **Fix:** stripped the trailing comment text from both branches in
      `weather_core.yaml`. Grepped the rest of the file for the same anti-pattern — no
      other live occurrences. Checked every exact-state consumer: `security_core.yaml`
      only checks `== 'stale'` (unaffected). Validated via Supervisor `check_config`
      (valid), reloaded `template` + `automation`, confirmed live state now exactly
      `"healthy"`. `input_boolean.openweathermap_api_limited` was already `off` at fix
      time, so the recovery automation's actual firing remains unverified live — next
      real limited→healthy cycle is the first real test. See INFRA_CONTRACT.md Part 4,
      BUG-WEA03.
- [x] **Gate-open assist lighting + garage light no longer driven by AP presence alone —
      2026-08-03.** User request, two parts. **(1)** Inside the same window the boundary
      security lights run in (`binary_sensor.security_lighting_required` = on), a main gate
      open now turns on `switch.garage_light` + `switch.front_house_security_light` for 10
      minutes — new `lighting_gate_open_assist` in `lighting_boundary.yaml`. This fills a
      real gap: `boundary_security_on` only lights street + main entrance when **nobody is
      home**, so driving in to an empty house after dark had no garage or front security
      light until the arrival pipeline caught up ~3.5 min later. The auto-off captures
      pre-state in `variables:` and only reverts lights that were *off* at gate-open time,
      and additionally defers to `lighting_garage.yaml` (garage occupied + door open) and
      `lighting_arrival_night.yaml` (`last_arrival_time` within 10 min, which owns the front
      security light on its own 5/15 min bedtime-gated schedule). **(2)** `lighting_garage.
      yaml`: the garage UniFi AP covers that whole side of the house, so a phone in an
      adjacent room at night set `binary_sensor.garage_occupied` and lit the garage with
      nobody in it. Presence branch now requires `garage_door_sensor` = on; new door-closed
      trigger turns the light off; door-open branch condition widened `night and not
      occupied` → `night or occupied` to cover the case where the AP connects before the
      door opens. Trade-off accepted by design: entering via the internal house door with
      the roller door shut gives no automatic light — a real garage motion sensor would be
      the proper fix. **No restart required — `automation:` YAML only, `Reload Automations`
      covers it (done).** Not yet live-verified: an actual after-dark gate open, and the
      10-min auto-off handoff to arrival lighting. See LIGHTING_CONTRACT.md §4 (Boundary
      Security + Presence-Aware Rooms).
- [x] **Main Gate notification camera field wrong + duplicated the vehicle-classifier push
      (BUG-S71) + stale arrival image with no freshness check (BUG-S72) — 2026-07-27.**
      User flagged, from live phone screenshots: "Main gate closed"/"Main gate opened"
      pings showing unrelated cameras (Cam09-Back-Bedroom, Cam12-Back-Pond), a duplicate,
      differently-formatted "🚗 Gate opened — vehicle entering" push for the same event,
      and an "Arrival confirmed... home at 14:43" push carrying a 09:49 domestic-staff
      photo. Root causes: (1) `notify_gate_opened`/`notify_gate_closed` (`alerts_doors.
      yaml`) never embedded a `cam:` hint or a slug-matching image filename, so
      `notify_security_events.yaml`'s `cam_name` fell through every tier to the volatile
      global `security_last_motion_camera` tracker (BUG-S56/58/70's fixes only ever
      covered branches that embed `reason`/`cam:` — these two never did). (2) Same two
      automations fire on the same `main_gate_sensor` transitions `security_gate_vehicle_
      stage1` uses for its own arrival/departure branches, with zero coordination between
      them — every vehicle event produced two separate pushes. (3) The Stage 1 arrival-
      image lock (BUG-S41/S48) read whatever was last in `ipcam03_driveway_history` with
      no age check, so a quiet/slow arrival with no fresh ipcam03 event in the 5s lock
      window silently inherited an old frame. **Fixes:** added an optional
      `camera_override` field to `script.notify_security_event` (top-priority tier, above
      `reason_cam`) and wired it into both gate pings; added a `condition:` to each
      mirroring `security_gate_vehicle_stage1`'s own arrival/departure gating booleans
      exactly (ipcam01 recent <180s / ipcam03 exit recent <120s — same tuned values from
      BUG-S38, no new thresholds invented) so the plain ping only fires when the
      classifier won't; added a real `camera.snapshot` + freshness check (<20s) to the
      arrival-image lock, falling back to a fresh capture instead of a stale one. See
      SECURITY_CONTRACT.md BUG-S71/BUG-S72 for full detail. **No restart required —
      `automation:`/`script:` YAML only, `Reload Automations` + `Reload Scripts` covers
      it.** Not yet live-verified: a real vehicle arrival/departure and a real pedestrian
      gate-open, to confirm the merge suppression lines up in practice. Known residual,
      not in scope for this fix: the same stale-history-lookup pattern (no age check) is
      still present verbatim in the departure branch's inline image and the RUNG 5c
      `gate_activity` router branch — flagged in BUG-S72 for a follow-up if it surfaces
      live.
- [x] **Geyser morning turn-on silently skipped + "Today's Cycles" stale + Temperature tile wording
      (BUG-PWR-GEYSER03) — 2026-07-27.** User: geyser not heated this morning, no alert, cold by ~6am.
      Recorder root-cause: `geyser_turn_on` had two time triggers at the identical 04:00:00 (and 04:30:00)
      — `_weekday` + `_saturday`; HA delivered only one on 07-27 (the Saturday one → default no-op), so the
      morning heat never started (manual rescue at 06:16, 1.67 kWh). **Fix 1:** collapsed Mon–Sat to one
      trigger per slot (`morning_winter_monsat`/`morning_standard_monsat`) + Branch 1 `dow <= 5`. **Fix 1b:**
      new `automation.geyser_morning_backstop` self-heals a missed start (turns on + warns ~15 min into the
      window). **Fix 2:** midnight-reset the `geyser_energy_at_morning_end`/`_at_midday_end` snapshots +
      made `sensor.geyser_daily_status` morning/midday/evening kWh attributes time-aware (they showed
      yesterday's figures 00:00→08:00). **Fix 3 (⚠️ needs full HA restart, `.storage` — NOT yet applied):**
      Temperature tile now distinguishes "🔥 Hot · reached temp earlier today" (switch off but heated today)
      from "💤 Not heated yet". Reload-only fixes (1/1b/2) deployed live via Supervisor API; `.storage` tile
      staged (backup `.bak.20260727_154658`). Full writeup: POWER_CONTRACT.md Issue 25.
- [x] **Water % full dashboard card recalibrated off `sensor.water_state` + predictive-fill threshold recalibration — 2026-07-17.** User asked to review the `card_mod` styling on the borehole/water status cards, which led to two fixes. (1) The `sensor.water_tank_level` status card on both Home Overview and the `water-control` page (`.storage/lovelace.dashboard_overview` + `.storage/lovelace.dashboard_operations` — same card duplicated verbatim on both) re-derived its own bucket cutoffs (`pct<15`/`pct<30`/`pct≥60`) independently of `sensor.water_state`'s real classification; converted to %, the actual live thresholds (`water_depth_critical`=12.8%, `water_depth_minimum_safety`=17.9%, `water_depth_low`=41.0%) don't match 15/30/60 at all — a second, drift-prone threshold system same class as the orphaned `water_policy_helpers.yaml` problem already flagged in WATER_CONTRACT.md Recommendation 2. Fixed by making the card read `sensor.water_state` directly instead of recomputing its own buckets, and added a `fault`/`unavailable`/`unknown` branch that didn't exist before (a Tuya dropout previously rendered as green "System Normal"). Both files are `.storage`-only (gitignored, NOT covered by `gitupdate.sh`) — **requires a full HA restart** per CODING_STANDARDS.md; do not open the dashboard UI editor first. (2) Separately recalibrated `input_number.water_predictive_fill_threshold_percent` YAML `initial:` 50 → 75 (`water_helpers.yaml`) per WATER_CONTRACT.md's own existing "Threshold calibration" recommendation (50% sat below all demand targets, so Branch 4.7 predictive fill rarely fired independently of the ordinary demand-target fill). **Live entity value is still 50%** — `initial:` only seeds a helper on first creation, does not retroactively update an existing one; needs a manual dashboard slider bump to 75% to actually apply. See WATER_CONTRACT.md "Predictive Fill Helpers" and "Dashboard % full card recalibration" (Section 5).
- [x] **Load-control disabled-too-long reminder alert (Issue 24) + Appliance Control dashboard
      cleanup + Geyser Temperature tile display fix — shipped 2026-07-17.** Started from the user
      asking why they got no alert for the geyser not running midday; investigation (live logbook +
      history via Supervisor API) found the critical "poor midday, forcing 60-min run" alert *did*
      fire correctly at 15:00 as designed — the real gap was the ~5.3h (09:32→14:55) window before
      that, while `input_boolean.load_control_geyser_enabled` sat manually disabled with zero
      proactive alerting (only a one-time info-tier "just disabled" push at 09:32). Fixed with a new
      shared `input_number.load_control_disabled_alert_hours` (default 2.5h) and
      `automation.load_control_disabled_too_long_alert` in `load_control.yaml` — fires once per
      continuous off-streak on any of the three `load_control_*_enabled` toggles once past the
      threshold (geyser/borehole → critical, pool → warning). See POWER_CONTRACT.md Issue 24 for
      full detail. Same session, on the live dashboard (`.storage/lovelace.dashboard_operations`,
      not git-tracked): added the new threshold helper to the existing shared "Appliance Controls"
      card (Power view); folded the long per-device tuning-slider tails on the separate "Appliance
      Control" view's Geyser/Pool cards behind a closed-by-default `custom:fold-entity-row` section
      (space + prevents accidental slider drags while scrolling); fixed the geyser "🌡️ Temperature"
      tile, which showed "⏳ Heating" any time the tank wasn't at temperature — including when the
      switch was off and the geyser was simply idle outside an active window — now distinguishes
      idle-off from actually-heating. **Required a full HA restart** (`.storage/lovelace` changes)
      — completed same session, confirmed Core API back online and the new automation present/`on`
      afterward. Also noteworthy: like other entries below, the `load_control.yaml` code change
      landed inside another session's concurrent commit (`88433c8d`) rather than its own — content
      correct and unaffected, just not reflected in that commit's message.
- [x] **Four alert-reliability bugs found and fixed, same session, 2026-07-17 — triggered by the
      user asking "why do I still get this alert after reload/restart".** (1) **BUG-NET06 real fix**
      (see entry below, and NETWORK_CONTRACT.md) — added the missing `time_pattern` re-evaluation
      trigger; reproduced the bug live via a `template.reload` right before fixing it. (2)
      **BUG-PWR-BATTERY01** (POWER_CONTRACT.md Issue 23) — `Power Battery Low Alert Active` used
      `float(0)` as its fallback, so `sensor.inverter_1_battery` going `unavailable` read as a
      phantom 0% (always "below threshold"), firing a false CRITICAL "Battery Low" push after a
      "Reload Helpers"; fixed with a `has_value()` guard. (3) **BUG-A14** (ALERTS_CONTRACT.md) —
      Critical Sensor Health trusted `sensor.watchman_missing_entities`'s periodic scan without
      cross-checking live state, so a reload-induced blip on `inverter_1_battery`/`inverter_1_grid`
      stayed cached as `unavail` for ~20 min after those entities actually recovered (confirmed via
      REST API and Developer Tools showing them healthy while the dashboard still listed them
      critical); fixed by cross-checking `states(id)` everywhere the Watchman cache is consumed.
      (4) **BUG-L18** (LIGHTING_CONTRACT.md) — `bar_bedtime_cutoff` sent "Bar closed" on every
      bedtime regardless of whether the bar was ever opened that evening; added a guard requiring
      the base bar light / patio lights / Apple TV to actually be on first. **All four deployed live
      via `template.reload` + `automation.reload` (Supervisor API), no restart required.**
      Also found and fixed, but **not yet live**: **BUG-A15** — the "Known Problem Escalation
      (Per-Device)" dashboard card (`.storage/lovelace.dashboard_system`) has never once rendered
      anything since it was introduced, due to a classic Jinja `{% set %}`-inside-`for`-loop scoping
      bug in its filter template (the list it built never escaped the loop, so the filter always saw
      an empty list). Fixed using the `namespace()` pattern already used correctly elsewhere in this
      repo; file backed up to `.storage/lovelace.dashboard_system.bak.20260717_221823` before
      editing. Per CODING_STANDARDS.md this needs a full HA restart to take effect — **deferred,
      confirm with user before running.** Also noteworthy: like the concurrent-session note above,
      all four of the live-deployed fixes landed via `git add .`-style sweeps into two of that other
      session's commits (`88433c8d` network/power/bar-bedtime, `06713d21` critical-sensor-health) —
      content correct and unaffected, just attributed to unrelated commit messages (BUG-P20/L17 and
      BUG-A13/S68 respectively), same as flagged in the entry above.
- [x] **Follow-up to the grounds-rear fix above — two more bugs found live-testing it, shipped 2026-07-17 same day.** After reloading, user got a real "⚠️ Perimeter activity" push whose body read `zones: Beds | ... | cam: cam15_passage` (an inside passage camera) — title and body flatly disagreed. Root cause: `security_event_classification`'s `reason` attribute computed `perim` from the *derived* `binary_sensor.security_perimeter_motion` (itself built from `security_perimeter_front_motion`/`_rear_motion` one hop removed), while the `state:` block that produces the title reads those two front/rear sensors directly — same "chained template lags its inputs" class as BUG-NET05/06/S63. Fixed by making `reason` read the same two sensors directly. Separately, the same push still showed the wrong "Camera:" field (Cam12-Back-Pond instead of the correct Cam15-Passage shown in the actual attached image) — this one was a regression risk from the fix above: making `security_last_motion_camera` update on every motion event (instead of only medium/high-confidence ones) means, for the common case of per-zone-image notifications, any camera firing anywhere on the property in the few seconds before delivery now overwrites it. Fixed by having `notify_security_events.yaml` prefer the camera name already embedded correctly in the message text (`security_event_classification`'s `reason`, captured atomically at classification time — guaranteed consistent with the image) over the volatile global tracker, falling back to the old logic unchanged for branches that don't embed `reason`. See SECURITY_CONTRACT.md BUG-S69/BUG-S70. **No restart required — automation/script YAML only, `Reload Automations` covers it.** Also noteworthy: discovered mid-session that another concurrent editing session was active on this same repo (see its commits `88433c8d`/`06713d21`, BUG-P20/L17/A13/S68) — my in-progress `security_logic.yaml` edit got swept into their commit via a `git add .`-style script rather than landing in its own commit. Content is correct and unaffected, just attributed to an unrelated commit message — flagging in case the commit history looks confusing later.
- [x] **Gate alert camera evidence + Cancel Alert button (BUG-A13); dogs_inside_prompt buttons actually wired for the first time (BUG-S68) — shipped 2026-07-17.** User asked for two things on the same theme (main gate / front security gate alerts, and the "everyone left, are the dogs inside?" prompt): (1) attach a *fresh* camera view — not a cached frame — to a gate alert at the moment it fires and again on every escalation; (2) a way to cancel further repeats on a false-positive gate-open; (3) turn the dogs-inside departure prompt into a real two-way ON/OFF toggle. Implementation: `notify_gate_opened`, `route_door_sustained_open_escalation`, and `route_door_alert_repeat_reminder` (`alerts_doors.yaml`) now run `camera.snapshot` immediately before every send (`camera.ipcam03_driveway` for `main_gate_sensor`, `camera.cam04_car_port_front` for `front_security_gate_sensor`, picked dynamically on the escalation path). New `input_boolean.gate_alert_snoozed` mutes `route_door_sustained_open_escalation`/`route_door_alert_repeat_reminder` for the rest of the current open cycle only — new automation `gate_alert_cancel_from_notification` sets it from a "Cancel Alert" tap (phone action `CANCEL_GATE_ALERT` or Telegram `/cancel_gate_alert`), new automation `gate_alert_snooze_reset` auto-clears it once `sensor.door_alert_context` returns to `normal`, so a dismissal can't suppress a later genuine event. Needed a new `actions`/`telegram_action` passthrough on `script.notify_security_event` (mirrors the `notify_system_event` pattern from BUG-A12) — while wiring that, discovered `dogs_inside_prompt` had **never** actually attached a button despite NOTIFICATIONS_CONTRACT.md claiming otherwise since 2026-06-14 (the script had no `actions:` mechanism at all) — the departure "🐕 Dogs home alone?" push now carries real "🐕 Yes, Inside" (`DOGS_INSIDE_ON`, existing handler) and "🚫 Not Inside" (`DOGS_INSIDE_OFF`, new handler `dogs_inside_off_from_notification`) buttons. See ALERTS_CONTRACT.md BUG-A13, SECURITY_CONTRACT.md BUG-S68, NOTIFICATIONS_CONTRACT.md script field docs. **No restart required — automations + script only, `Reload Automations` + `Reload Scripts` covers it.** Not yet live-verified: a real gate-open cycle and a real departure event, to confirm the Telegram `inline_keyboard` renders correctly now that it's templated (new technique for `telegram_bot.send_message` specifically) and that both button pairs work end-to-end. Known gap, not addressed: the 5/10/30/60 min repeat reminder only triggers off `group.tier1_perimeter` (main gate only) — front security gate never gets repeat nags, only the one-shot critical escalation.
- [x] **Grounds-rear stale image/camera-name bug + Event Timeline gap + daytime noise downgrade — shipped 2026-07-17.** User reported a pool/pond "Activity in grounds" push showing a 4-day-stale IPCam04-Pool-Bar image and a mismatched "Camera: Cam04-Car-Port-Front" field for a `cam12_back_pond, conf: low` event, plus the Overview dashboard's Security Status card not lining up (correct real-time camera name next to the same stale image). Root cause: `security_capture_best_snapshot`'s medium/high-confidence gate (same class as BUG-S47) never wrote `input_text.security_image_grounds_rear` or the two GLOBAL trackers (`security_last_motion_camera`/`security_last_motion_image`) for cam12/cam09 — both structurally can never reach medium/high confidence alone. Fixed by moving these writes into `security_capture_each_camera_motion` (fires unconditionally on every motion event). Also found and fixed the same gate blocking the security-control dashboard's Event Timeline (`input_text.security_event_session`) from ever showing low-confidence-only events. Separately, per user request, `grounds_low_confidence` (RUNG 7b — cam04/cam07/cam09/cam12 NVR/no-AI daytime false positives with nobody home) now downgrades a single uncorroborated daytime occurrence to `information` severity, escalating back to `warning` on a repeat within 15min, multi-camera corroboration, or nighttime (new `counter.security_grounds_low_confidence_count`). Overview dashboard's "Camera:" markdown line also reformatted to match the operations dashboard's friendly-name style (was showing raw `camera.cam12_back_pond` entity IDs). See SECURITY_CONTRACT.md BUG-S65/BUG-S66/IMPROVEMENT-S67. **No restart required — `Reload Automations` covers the automation changes; the new `counter:` helper needs a YAML reload-all (or restart if it doesn't appear) to register.**
- [x] **Security repeat reminders — base implementation shipped 2026-07-10.** `alert.security_alert`'s `notifiers: [STD_Alerts]` removed (was dead — same class as BUG-A11); a new `automation.security_alert_repeat_reminder` (`alerts_security.yaml`) now delivers the 5/15/30/60min reminders via `script.notify_security_event` directly, gated to fire only at those exact elapsed-minute marks so it structurally cannot duplicate the immediate first notification. Muted via the existing `input_boolean.security_alert_notify` toggle — does NOT yet hook the alert entity's native "Acknowledge" button (no template-readable ack state exposed by the `alert:` integration); that's a follow-up. **⚠️ Requires a full HA restart** (`alert:` entity `notifiers:` changed). See ALERTS_CONTRACT.md BUG-A10.
- [x] **Follow-up from BUG-A10 — resolved 2026-07-13 (BUG-NET08).** `alerts_temperature.yaml`'s routing automations had NO reload guard at all (worse than network's pre-2026-07-06 state) — fixed same session as the repeat-reminder work below, see that entry. "Acknowledge" button on `alert.security_alert` still has no effect on repeat delivery — genuinely still open, out of scope for the 2026-07-13 repeat-reminder pass (that pass targeted the 16 domains still on dead `STD_Alerts`; `security_alert`/`camera_health` already have their own working repeat mechanism from BUG-A10/A11).
- [x] **CORRECTED 2026-07-10, then actually fixed 2026-07-13:** the recurring `ServiceNotFound: notify.ryan_iphone16_mobile_app` / `ap_0223_1001` / `honor_10_dash_mobile_app` / `honor_x7_dash_mobile_app` / `telegram_bot_5527` errors are `notify.STD_Alerts` (`configuration.yaml`, `platform: group`) calling 5 dead bare `notify.<x>` services on every trigger AND every repeat, for all 16 domains still using `notifiers: [STD_Alerts]` (see NOTIFICATIONS_CONTRACT.md §7). The 2026-07-06 fix only ever covered the *initial* notification (via parallel one-shot routing automations); repeats kept silently failing — meaning an alert that stayed active for hours (e.g. a missing critical sensor) only ever notified once. **Fixed 2026-07-13**: added matching `for:`-duration repeat triggers (mirroring each domain's own dead `repeat:` schedule) to all 16 routing automations, reusing the existing message-building action code — no new automations needed except `alerts_doors.yaml` (door_alert has `skip_first: true`, so it had no one-shot routing automation to extend; added `route_door_alert_repeat_reminder`). `STD_Alerts` itself is still left broken-but-defined (unchanged, same as before) — only the working parallel path changed. See ALERTS_CONTRACT.md 2026-07-13 entry for full per-domain detail. `notifiers:` fields were NOT touched, so this is a pure `automation:` change — **no restart required, `Reload Automations` is sufficient.**
- [ ] **Telegram photo attachment unreachable (infra, not YAML) — old root-cause theory
      retracted 2026-08-21, still open.** `telegram_bot.send_photo` fails with "Failed to
      load URL: All connection attempts failed" for `https://ha.dunners.tech/...`. The
      previously-recorded root cause ("resolves to 10.10.1.5, nothing listening") is stale
      — live DNS lookup 2026-08-21 resolves correctly to Cloudflare, and the user confirmed
      local + public + VPN access all work today; do not keep building on that theory.
      **Needs a fresh live diagnostic** (exact HA log error text captured at the moment of
      a real failure) next time it happens — not re-attempted this session. Text/push
      notifications and Telegram message text + buttons are unaffected — only the inline
      photo in Telegram fails. See SECURITY_CONTRACT.md BUG-S61.
- [x] **Known-Problem Escalation feature (2026-07-10) — restart completed, live-verified same day,
      2 follow-up bugs found and fixed post-restart.** `.ha_run.lock` confirmed a real core restart at
      13:04. Verified via Supervisor API (`/core/api/states`, `/core/api/template`) that the registry
      sensor, `sensor.problem_alert_devices`, and the exclusion logic all work correctly against live
      state. Found and fixed two real bugs surfaced by the user during verification: (1) the
      "Active Alert Binary Sensors" card got stuck on "Configuration error" because its
      `filter.template` could render blank instead of literal `[]` when nothing matched — fixed
      everywhere in this feature; (2) `Total Acknowledged Alerts` and both "Acknowledged" list cards
      (Home + Alerts page) weren't excluding known-problem alerts, so a flagged-and-acknowledged
      alert (both toggles can be true independently) kept reappearing there. Fixed via `template.reload`
      (no further restart needed — no `alert:` entities touched this round). See ALERTS_CONTRACT.md
      Section 4B "Post-restart verification round". Also clarified (not a bug): "door alert missing
      from Active Alert Device Details" is pre-existing — `alerts_doors.yaml` labels its context
      sensor `domain: "Security"`, so it renders there instead.
- [x] **Known-Problem Escalation redesigned per-device (2026-07-10, same day) — shipped, live,
      user-verified working.** The original version was whole-domain-only; the user correctly
      pointed out `alert.critical_sensor_health` can report multiple unrelated hardware faults at
      once, so flagging the whole domain would hide a future genuinely-new fault. Redesigned: 22
      static `input_boolean.problem_device_*` helpers (one per member of `group.critical_sensors` /
      `group.network_devices` / `group.wan_services` / `group.device_power_monitored`, plus 2 for
      presence's static sub-types) replace the single `critical_sensor_health_marked_problem`
      toggle; `camera_health`/`security_alert` (NEW) stay domain-level since their alerts are
      structurally single-entry. New `sensor.device_problem_registry` (device→helper map) and
      `sensor.suppressed_alert_entities` (per-entity full-coverage check) added to
      `alerts_summary.yaml`. Also fixed a **pre-existing** bug found during design review:
      `alert.device_power_fault` could never be resolved by the aggregator's name-resolver (base
      strips to `device_power`, neither `alert.device_power` nor `alert.device_power_alert` exist)
      — device-power rows have silently carried a bogus `alert_entity` since that domain was
      written; fixed via an override map.
      **Real bug found post-deploy (3 restarts total this session to fully resolve):** the
      "Configuration error" the user kept seeing on "Acknowledged (N)" and "Known Problem
      Escalation (Per-Device)" survived TWO full restarts, ruling out the "stuck card from boot
      race" theory. Root cause, found by reading the vendored `auto-entities.js` source directly:
      this card version does NOT YAML-parse `filter.template` output — it naively
      whitespace/comma-splits the rendered string. The mapping-row format used in 5 templates
      (`- entity: X\n  secondary_info: Y`) got shredded into garbage tokens. Fixed by reverting all
      5 to bare-entity-id-per-line (the format already proven safe by the pre-existing
      Critical/Warning/Info cards) — 3rd restart loaded the fix, confirmed clean via Supervisor API
      re-render of every affected template against live state. **New standing rule added to
      CODING_STANDARDS.md** (Dashboard Card Standards) so this exact mistake isn't repeated.
      **Not fixed, flagged as follow-up:** `input_boolean.problem_device_*` helpers reset to `off`
      across restarts 2 and 3 despite being manually toggled `on` beforehand —
      `.storage/core.restore_state` shows all 22 helpers resetting in lockstep at each restart's
      startup moment, not preserving individual manual toggles. Recorder has no `input_boolean`
      exclusion, so root cause unconfirmed — needs its own investigation session, since it
      currently means known-problem flags don't reliably survive a restart. See ALERTS_CONTRACT.md
      Section 4C (supersedes 4B) for full detail, including the "Real bug found post-restart"
      writeup.
- [x] **Network dashboard rebuild (2026-07-10) — shipped and verified live.** The
      `network-control` and `network-debug` views on `.storage/lovelace.dashboard_operations`
      were rebuilt (WAN/LAN consolidation, new NAS/UPS debug sections); required a full HA
      restart per CODING_STANDARDS.md, which has since happened (HA 2026.7.1, core state
      RUNNING). Post-restart verification: both views confirmed intact in `.storage`
      (network-control 3 sections / network-debug 5 sections, both `max_columns: 3`), no
      lovelace/template errors in `ha core logs`, and every entity referenced by either view
      spot-checked live via the REST API. See NETWORK_CONTRACT.md Section 11.
- [x] **BUG-NET06 — root-cause fix actually applied 2026-07-17.** `sensor.network_device_down_alert_severity`
      was a trigger-based template sensor with no periodic re-evaluation, so a reload/restart-time
      race (2+ APs briefly `unavailable` during UniFi reconnect) could freeze it at a stale
      `critical` indefinitely. Added a `time_pattern: minutes: "/5"` trigger. Live-verified the bug
      reproducing in real time (a `template.reload` froze severity/context at `critical` again
      while `alert.network_alert` correctly stayed `idle`) immediately before applying the fix, then
      confirmed the new trigger self-corrects within one 5-min cycle. See NETWORK_CONTRACT.md.
- [x] **Network dashboard polish pass (2026-07-10, same day) — confirmed live 2026-08-21,
      multiple restarts completed since.**
      Post-restart visual review of network-control/network-debug found one real defect (fixed)
      and two non-issues (browser-cache related, no YAML change). Fixed: 5 oversized per-AP
      restart buttons on network-debug consolidated into one compact shared grid (they were
      standalone `button` cards outside a grid wrapper, so they took full section-grid sizing).
      Added a "Total Known Clients" line to the network-control LAN table. Confirmed no ASUS
      ROG router restart control exists in HA (device has zero button/switch entities — ping +
      stats-only, no `asuswrt` integration configured). See NETWORK_CONTRACT.md Section 11
      "Post-restart polish pass". **Do not open the dashboard's UI editor before restarting.**
- [x] **Geyser turn-ON command verification — added 2026-07-13, same day as the turn-off
      fix, at user's explicit request** ("add to turn on because if miss morning window
      then cold for showers"). Originally scoped out (see superseded note this file used
      to carry, now removed) on the reasoning that turn-on self-heals via midday-forced-
      minimum/20:00-check/evening fallbacks re-attempting later the same day — true for
      midday/evening, not true for morning, which has no same-day backstop. Added
      `script.geyser_verified_turn_on` (exact mirror of `geyser_verified_turn_off`); all 7
      `switch.turn_on` call sites now route through it. See POWER_CONTRACT.md Issue 21
      "UPDATE 2026-07-13" for full detail.

*2026-07-17 — `arrival_detected` deadlock silently killed all night-arrival lighting for ~4 months (BUG-P20/BUG-L17); front security light added to Quiet Mode arrival with 15-min bedtime cap:*
*PRESENCE/LIGHTING — User reported entrance downlights + dining room not turning on for a 9pm+ arrival on 2026-07-14. Both the pedestrian-entrance fix (BUG-P17, 2026-07-08) and the dining-room scene gap (BUG-L15, 2026-06-28) were already closed, so — rather than trust doc status — investigated the recorder database directly (`home-assistant_v2.db`, 39GB, 90-day retention). Confirmed live via Supervisor API: `input_boolean.arrival_detected` had `last_changed: 2026-07-16T11:01:35Z` with zero `off` rows in the recorder going back to 2026-06-14 (earliest queried); `automation.presence_clear_arrival_flag.last_triggered == null` (never fired, ever); `automation.arrival_night_lighting.last_triggered == 2026-03-03` — over 4 months stale.*
*ROOT CAUSE — `presence_clear_arrival_flag`'s only trigger is `from: "off", to: "on", for: "00:05:00"` — it can only re-arm on a genuine off→on edge. Because the boolean was apparently already `on` when this automation was first created (2026-05-17, same session as BUG-P06), it never got that edge and could never clear itself. Every later `turn_on` call (from `house_entry_event`, `presence_boundary_resolver`, `security_automations.yaml`) was then a no-op on an already-`on` boolean — no `state_changed` event, so `automation.arrival_night_lighting`'s own `from: off, to: on` trigger never re-fired either. A self-sustaining deadlock, invisible to `ha core check` (valid YAML, silent at runtime) and not something BUG-P17's fix could have surfaced, since BUG-P17 only added a new caller of `turn_on` — it never touched the clear side.*
*FIX — Added a `time_pattern` (`minutes: "/5"`) trigger to `presence_clear_arrival_flag` (`presence_boundary.yaml`) plus a `condition: state ... for: "00:05:00"` guard, so it force-clears the boolean on a periodic check independent of edge timing — same proven pattern as the `staff_on_site_override` stuck-boolean fix (BUG-P14). Manually cleared the live stuck value via the Supervisor API immediately rather than waiting for the new backstop. `ha core check` passed; reloaded via `automation.reload` (no restart needed — automation-only change). Verified live: `input_boolean.arrival_detected` → `off`, `automation.presence_clear_arrival_flag` picked up the new trigger.*
*FEATURE (user request, same session) — Quiet Mode arrival scenario (`lighting_arrival_night.yaml`, fires when `binary_sensor.quiet_arrival_mode` is on — i.e. arriving after the kids are already in bed) now also turns on `switch.front_house_security_light`, auto-off 15 minutes later if `input_boolean.bedtime_mode` is on — mirrors Scenario 2's existing 5-min patio/front-security auto-off pattern (BUG-L15), just a longer window since quiet mode arrivals are later/darker by definition. Deliberately left as a separate scenario block rather than merged with Scenario 2 — the two fire under different conditions (`quiet_arrival_mode` vs `anyone_connected_home`+not-quiet) and previously had different light lists; keeping them separate matches the existing design (see BUG-P17 session note where quiet mode was deliberately kept minimal — this reverses that specific decision for front security light only, not patio).*
*DOCS: PRESENCE_CONTRACT.md BUG-P20 added (Section 10 + Section 13 summary, now 15 closed issues) + Section 5 Arrival Persistence note. LIGHTING_CONTRACT.md BUG-L17 added (cross-referencing BUG-P20), Section 4 Night Arrival scenario table updated (Quiet Mode + front_house_security_light + 15min cap; Someone Home clarified as bedtime_mode-gated not unconditional), Section 8 cross-domain table entry updated.*

*2026-07-13 (alerts session — reload-glitch debounce + STD_Alerts repeat-reminder fix,
BUG-NET08) — Investigated a live incident (`switch.water_pressure_pump` unavailable since
17:41 UTC after a restart, tripping `alert.critical_sensor_health`) and a user-reported false
"Network Alert — Device Down" push (4 APs) plus a false "Bar occupied" push, both fired
during an unrelated `template:` reload (adding the ZenWiFi sensor, see session above).
**Root cause (BUG-NET08):** the existing `from: "off"` reload guard (added 2026-07-06,
ALERTS_CONTRACT.md) only protects against the unknown/unavailable transient a `template:`
reload causes. It does NOT protect against a *second*, previously-flagged-but-unaudited
failure mode: a reload can make a chain of dependent template sensors recompute out of
order, so a downstream sensor briefly reads a not-yet-recomputed upstream one and computes a
valid-looking-but-wrong value — never passing through unknown/unavailable at all. Confirmed
live for `binary_sensor.network_device_down_alert_active` (→ `group.network_devices` → AP
ping sensors) and `binary_sensor.bar_occupied` (→ `sensor.bar_presence_confidence` →
`binary_sensor.bar_occupied_raw`/`bar_motion_raw`) — same dependency-chain shape in both.
**Fix:** added `for: "00:00:20"` to every reload-vulnerable `from: "off", to: "on"` trigger
across `alerts_network.yaml` (4 automations) and `lighting_bar_presence.yaml` (1 trigger) — a
sub-minute recompute glitch can't sustain 20s, a real event still notifies almost instantly.
Audited `alerts_temperature.yaml` per the still-open BUG-A10 follow-up item (above) and found
it had NO guard at all (worse than network's pre-fix state) — added `from: "off"` + `for:
20s` + `not_from: [unknown, unavailable]` to all 4 routing automations (WAN/LAN/Device/
Storage Temp).
**Bundled in the same pass — STD_Alerts repeat-reminder gap (see OPEN TODO entry above for
the diagnosis):** added repeat-reminder `for:`-duration triggers, matching each domain's own
dead `repeat:` schedule, to all 16 alert entities still routed through the broken
`notify.STD_Alerts` group: `critical_sensor_health`, `network_alert` (4 causes: device_down/
wan_down/wan_degraded/device_restart), `device_power_fault`, `presence_alert`,
`dash_battery_alert`, `garden_alert`, `media_alert`, `power_alert`, `door_alert`, `water_alert`,
`water_borehole_fault`, `water_borehole_critical_fault`, plus the 4 temperature domains above.
All reuse the existing message-building action code (severity read fresh at execution time),
so no logic was duplicated. **Known-problem suppression (explicit user request):** for the 4
domains with a per-device `sensor.device_problem_registry` entry (`critical_sensor_health`,
`network_device_down`, `wan_down`, `device_power_fault`, `presence_alert`'s unknown-device
case) added a `condition:` that skips the notify action — both initial and every repeat — if
every currently-bad member is marked "known problem" via `input_boolean.problem_device_*`.
Domains without a registry entry (batteries/garden/media/power/doors/water/temperature) got
repeat delivery only, no suppression (there's no "mark as known problem" toggle for them to
check). `alerts_doors.yaml`'s `door_alert` has `skip_first: true` and no pre-existing one-shot
routing automation to extend, so added a new `route_door_alert_repeat_reminder` automation
instead, reusing `door_alert`'s own severity-branched message template.
**No `alert:`/`notifiers:` blocks were touched anywhere** — every change is additive inside
existing `automation:` sections, so this is a pure automation reload, **no HA restart
required**. `ha core check --raw-json` → `{"result":"ok"}`. Not yet reloaded/verified live as
of this entry — see ALERTS_CONTRACT.md and NETWORK_CONTRACT.md for full per-file change list.
Also still live and unresolved as of this entry: `switch.water_pressure_pump` +
`switch.sonoff_pond_fountain_pump` + `switch.sonoff_10021bb87e` (BASICR2_extractor_network)
have been `unavailable` since the 17:41 UTC restart — 3 Sonoff devices that didn't reconnect
while every other Sonoff device on the same integration did; needs a physical/on-site check,
not a config fix.*

*2026-07-13 (network topology correction session) — User flagged that the dashboards (and
this file) had the WAN router wrong: the **ASUS ZenWiFi XD6 (192.168.1.3)** is the actual
internet-facing WAN router, not the ASUS ROG router (192.168.1.1), which exists solely to
provide a dual-LAG bonded LAN connection for the Synology NAS. Corrected `network-control`
(added a WAN Router status line for ZenWiFi at the top of the WAN section; LAN table now
leads with a "ZenWiFi (WAN Router)" row followed by a new "ASUS ROG (NAS Dual-LAG)" row,
previously absent from this view entirely) and `network-debug` (new WAN-section block for
ZenWiFi; ASUS ROG heading relabeled from "Secondary LAN" to "NAS Dual-LAG only — not WAN").
Fixed this file's "Network" hardware line (was `WAN Router: ASUS ROG`). Updated
NETWORK_CONTRACT.md Section 1/3/8/11 with the corrected topology, the new router entity
reference, and IMP-NET04 (the ZenWiFi still isn't in `group.network_devices` or any alert
group — dashboard visibility only, alerting not yet wired up). Both `.storage/lovelace.*`
files backed up before editing and validated as parseable JSON with unchanged section
counts. **Restarted 2026-07-13 19:40:57** — confirmed via `ha core logs`, no lovelace or
template errors; both `.storage` files still contain the new content post-restart (nothing
reverted by a stale-editor autosave). **Also fixed, found while reviewing the live
dashboard post-edit (BUG-NET07):** `sensor.wan_health_score` and the three
`sensor.wan_*_packet_loss` template sensors had no `state_class`, so their `network-debug`
"WAN Health Score (7d)"/"WAN Packet Loss (7d)" `statistics-graph` cards showed "No
statistics found" (recorder only keeps long-term statistics for sensors with a
`state_class`). Added `state_class: measurement` to all four in
`packages/network/network_helpers.yaml`; `ha core check` passed. Template-sensor change
only — applied via `template.reload` (called directly through the Supervisor API, after the
restart above), confirmed live: all four sensors now show `state_class: measurement` in
their attributes.

**Same day, follow-up — IMP-NET04 closed.** User asked to add ZenWiFi to the correct alert
groups. Added `binary_sensor.zenwifi_xd6_connected` (new template binary_sensor,
`network_helpers.yaml` — no discrete connected/disconnected state exists for this device,
so `sensor.zenwifi_xd6_cpu_usage` availability is used as the online/offline proxy) to
`group.network_devices` (`alerts_network.yaml`), and `sensor.zenwifi_xd6_uptime` to
`group.network_device_uptimes` + `group.network_device_restart_times`. `ha core check`
passed; applied live via `template.reload` then `group.reload` (order matters — the group
references the new template entity), no restart needed. Verified via API:
`binary_sensor.zenwifi_xd6_connected` reads `on`, `group.network_devices.entity_id` lists
it, both uptime groups list `sensor.zenwifi_xd6_uptime`. NETWORK_CONTRACT.md Section 3/8
updated (IMP-NET04 marked fixed, group member table updated).*

*2026-07-13 (notifications session) — Onboarded Vicky's phone (`notify.vicky_iphone13_mobile_app`
entity target / `notify.mobile_app_iphone13promax_vicky` legacy service, confirmed live via
Supervisor API `GET /api/services`) into 3 of the 6 domain notification scripts, per explicit
user request to avoid spamming her with things she doesn't understand: **Security**
(`notify_security_events.yaml`), **Power** (`notify_power_event.yaml`), and **Water**
(`notify_water_events.yaml`, added later same session on follow-up request). Added her as a 5th
per-device target in the **warning and critical branches only** in all three scripts —
deliberately left out of the information branch (and, for water specifically, out of its
unique quiet-hours info→warning escalation branch too), so she gets always-audible
warning/critical pushes but nothing at the info tier. Matches the existing hardcoded-per-device
pattern already used for the other 4 targets in each script (this repo has no per-person
notification-preference infrastructure — user explicitly chose hardcoding over building a new
toggle system, see NOTIFICATIONS_CONTRACT.md). Presence/System/Lighting scripts untouched — she
is NOT onboarded to those domains. Deployed live: `ha core check` passed after each edit,
Scripts reloaded via Supervisor API both times, live-fired a real test warning through
`script.notify_security_event` — confirmed via logbook that `notify.vicky_iphone13_mobile_app`'s
state updated immediately after (push accepted, no errors in logbook for that context). Power
and Water additions used the identical, already-proven call shape (not independently
live-fired). See NOTIFICATIONS_CONTRACT.md Section 3A "Per-Person Onboarding" for the pattern if
another family member or domain is added later.*

*2026-07-13 (power session — BUG-PWR-GEYSER01) — Investigated user report "geyser was still
on by 11pm" Saturday 2026-07-11. No `home-assistant.log` was available for the window
(current log had rotated out), so root cause was found by querying the recorder DB
(`home-assistant_v2.db`) directly: `geyser_turn_off`'s winter evening hard-off branch fired
correctly at 21:00:00 and issued `switch.turn_off` — confirmed via the events table — but the
physical Tuya Cloud switch never responded. It stayed reported "on" through the 21:30/22:00
default-branch checks, and its paired power sensor (`sensor.geyser_heat_pump_power`) was
frozen with zero updates for ~3 hours (18:56→22:10) with no `unavailable` transition to flag
the failure. The geyser kept physically heating (confirmed by a real reheat cycle 22:10–22:40
climbing to ~1090W) until manually forced off via the dashboard at 23:14:51. Not an
automation logic bug — the automation did exactly what it should; the Tuya Cloud command was
silently dropped downstream with no error surfaced anywhere in HA. **Fix:** added
`script.geyser_verified_turn_off` to `geyser_automations.yaml` (E5 in the file's own History
block) — issues the command, waits 5 min for confirmed off, retries once, waits another
5 min, then logs + fires a **critical** alert if the switch still hasn't responded. All 10
`switch.turn_off` call sites in the file now route through it (AM battery protection, both
morning hard-offs, midday, all 4 evening/sports hard-offs, orchestrator emergency, manual-run
completion). Deployed live: `ha core check` passed, Automations + Scripts reloaded via the
Supervisor API, confirmed `script.geyser_verified_turn_off` live with correct `mode: queued`
/ `max: 5` attributes. Did not live-fire a full retry-path test (would have toggled the real
geyser mid-afternoon) — logic is a standard `wait_for_trigger`/timeout/retry pattern, not
independently verified under an actual Tuya failure. Turn-on verification deliberately scoped
out — see OPEN TODO above. **Also fixed, found while investigating:** CLAUDE.md and
INFRA_CONTRACT.md both misattributed every Tuya device in the house (geyser heat pump switch,
pool pump, pond filter pump, water tank level sensor cluster) to the `localtuya` integration.
Confirmed via `.storage/core.entity_registry` + `.storage/core.config_entries` that all of
them are actually on the cloud `tuya` integration, and `localtuya` (installed via HACS) has
zero registered entities — corrected both docs. Full incident writeup:
POWER_CONTRACT.md Issue 21 / BUG-PWR-GEYSER01.*

*2026-07-13 (power session, same day — BUG-PWR-GEYSER02) — User reported geyser cold Monday
evening ("thought we fixed this... turns on earlier on monday/thursday night after maid has
been here"), manually turned it on. Recorder DB showed the manual `switch.turn_on` actually
fired at 18:07:43 (user first said "8pm", corrected to "6pm" — matches; also corroborated by
a physical timer photo showing 18:13 on its own clock). Today's midday run was strong (5.04
kWh delta, confirmed at temperature by the 15:00 hard-off) — the tank had simply cooled
during the normal ~3h winter standby gap before the already-scheduled 18:30 fallback; the
manual toggle landed ~20 min ahead of what the automation was already about to do. **This
specific incident was not a bug** — but investigating it surfaced a real one matching the
user's stated expectation: `geyser_turn_on` Branch 3's (evening early start) maid-day
threshold boost checked `now().weekday() == 3` (Thursday only), inconsistent with this same
file's own maid-day definition used everywhere else (`weekday() in [0, 3]`, Mon+Thu — see the
morning-extension logic). Duplicated Thursday-only in 4 places: 3 in
`geyser_automations.yaml` (Branch 3 condition + logbook + notify messages) and 1 in
`power_state.yaml`'s `sensor.geyser_daily_status` (dashboard-facing `midday_adequacy` text).
**Fix:** all 4 corrected to Mon+Thu; user-facing text now says "maid day high-usage" instead
of "Thursday high-usage day". Left `input_number.geyser_thursday_high_usage_extra_kwh`'s
entity ID unchanged (avoid a dashboard-binding-risking rename) but corrected its misleading
friendly name to "Geyser Maid-Day High-Usage Extra Energy" with an explanatory comment. Note:
today's actual midday delta (5.04 kWh) clears even the corrected higher threshold (4.5 kWh),
so this bug wouldn't have changed tonight's outcome either way — it'll matter on a future
Monday with more marginal midday solar. Deployed live: YAML validated, `ha core check`
passed, Automations + Template Entities + Helpers reloaded via Supervisor API, confirmed live
via REST API (`sensor.geyser_daily_status.midday_adequacy` now reads correctly). Full
incident writeup: POWER_CONTRACT.md Issue 22 / BUG-PWR-GEYSER02.*

*2026-07-10 (alerts dashboard session) — Added "Known-Problem Escalation": a user-triggered
toggle that removes a recurring/hardware-fault alert (e.g. `alert.critical_sensor_health` for the
stuck-`unavail` `switch.water_pressure_pump`) from the Global Alert Summary, the alert totals,
and the "Active Alert Binary Sensors" card on both the Home and Alerts dashboards — without
touching the underlying `alert:` entity's own on/off/repeat lifecycle. The flagged alert shows
instead in a new "Known Problems" section on the Alerts page. Redesigned same day from a
whole-domain toggle to per-device (22 static `input_boolean.problem_device_*` helpers) after the
user pointed out `alert.critical_sensor_health` can report multiple unrelated hardware faults at
once; `camera_health`/`security_alert` stay domain-level since those alerts are structurally
single-entry. Also fixed a pre-existing bug: `alert.device_power_fault` could never be resolved by
the aggregator's name-resolver. Full detail, including a real "Configuration error" root-cause
bug found post-deploy (this repo's vendored `auto-entities.js` doesn't YAML-parse
`filter.template` output — it whitespace-splits it, so mapping-row templates get shredded into
garbage tokens; fixed by reverting to bare-entity-id-per-line, new rule added to
CODING_STANDARDS.md) and camera_health/camera_health_context display-name fixes: see
ALERTS_CONTRACT.md Section 4C. Required 3 full HA restarts total this session; all verified live
via Supervisor API before and after each. Not fixed, flagged as follow-up:
`input_boolean.problem_device_*` helpers aren't reliably persisting their `on` state across
restarts — needs its own investigation session.*

*2026-07-10 (doc-audit + fix session) — Read PROJECT_STATE, CODING_STANDARDS, and all 11
domain contracts; verified every "open" bug claim against live YAML instead of trusting the
docs. Found ~14 bugs marked open across ALERTS/LIGHTING/NOTIFICATIONS/PRESENCE/INFRA/SECURITY
contracts that were already fixed in code — closed all of them out with dated Status notes in
their respective contracts (see each file's changelog). Also fixed the bugs that were still
genuinely live:*

*PRESENCE — BUG-P08: `presence_validation.yaml`'s unknown-AP sensors read
`device_tracker.<name>_iphone` (no `_tracker` suffix), which doesn't exist in
`.storage/core.entity_registry` at all — `state_attr()` on a nonexistent entity always
returns `None`, so both sensors were permanently stuck at 0 since creation. Cross-checked
`.storage/person` (People integration) to confirm the canonical entity is the
`_tracker`-suffixed UniFi tracker (already used correctly in `presence_core.yaml`); aligned
`presence_validation.yaml` to match. BUG-P09: departure automation's `logbook.log` said
"House entry event recorded" (copy-paste from the entry automation) — fixed to "departure".*

*ALERTS — BUG-A10 follow-up: `alert.security_alert` had the same dead-`notifiers:` problem
BUG-A11 already fixed for `camera_health` (`STD_Alerts` broken since 2026-06-28). Removed
`notifiers: [STD_Alerts]`; added `automation.security_alert_repeat_reminder` as a base
interval-repeat implementation — see OPEN TODO above for details and the follow-up gap
(Acknowledge button not wired up).*

*INFRA — BUG-CORE02: removed dead `{% set errors = 1 %}` from `sensor.ha_stability_score`
(`core_helpers.yaml`); also added `temp > 85` (critical) / `temp > 70` (degraded) thresholds
to that sensor, since it previously ignored CPU temperature entirely despite driving the
"Home Assistant Health" dashboard card's color — a genuinely overheating Pi with normal
CPU/load would have shown green. BUG-CORE01: the sensors this entry described no longer
exist (already replaced by the lightweight `sensor.ha_event_rate_1m` `statistics`-platform
sensor in `packages/sensors/filter.yaml`); added an inline comment there explaining why it
must stay cheap (Raspberry Pi performance) instead of reverting to a full-registry-scan
pattern. BUG-WEA02: fixed `# 1hr` → `# 4hr` comment on the 14400s threshold
(`weather_core.yaml`). BUG-WEA01 follow-up gap: `weather_api_recovery`'s
`not_from: [unknown, unavailable]` guard was blocking recovery on the exact
template-reload transition it needed to catch — removed the guard (recovery only needs
`to: "healthy"`; action is idempotent, condition already checks the limited-mode boolean).*

*SECURITY — Issue 7: added `from: "off"` to `security_capture_each_camera_motion` and
`security_track_movement_path` (both had bare `to: "on"`, could fire on restart);
`security_event_start` has no on/off shape (triggers off a camera-name-valued sensor) so
added `not_from: ["unknown", "unavailable"]` instead. Also closed Issue 8 in the same file as
stale — its "Deferred to S2/S3" trust-model-disabled claim was already resolved by the S2/S3
classifier rebuild (verified: zero `# disable for testing` matches repo-wide,
`security_automations.yaml` actively checks `staff_on_site`/`guest_mode`/`low_trust_present`).*

*DASHBOARD — Home page's existing "Home Assistant Health" mushroom card
(`.storage/lovelace.dashboard_overview`, Home view) showed CPU/Load/RAM only; extended its
`secondary` template to add CPU temperature and uptime (both already computed by
`sensor.ha_system_monitor_*` / `sensor.ha_uptime_hours`, just not surfaced). Edited via direct
JSON patch with a byte-exact round-trip verification before/after (single-field diff
confirmed, nothing else in the file touched) — same file another in-progress session had
already made unrelated edits to today; confirmed via diff that those were preserved.
**⚠️ Requires a full HA restart** (`.storage/lovelace` changed) — do not open the Home
dashboard's UI editor before restarting.*

*2026-07-10 (network dashboard session) — Consolidated the Network domain into two rebuilt views
on `dashboard_operations`: `network-control` (at-a-glance status + primary controls — WAN health
banner, bandwidth/loss/jitter tiles, consolidated LAN device markdown table, restart-button grid,
NAS/UPS protection) and `network-debug` (deep-dive — WAN quality history, per-device UniFi
Gateway/Switch/ASUS/5×AP/ZenWiFi breakdowns, plus brand-new NAS (Synology Guardians) and UPS
(EcoFlow River Pro) diagnostic sections that weren't on any dashboard before). Both now use the
`sections` layout at `max_columns: 3` (tablet-safe), matching the pattern already used by
power-control/security-control/water-control. Added breadcrumb chip-card navigation between the
two views (mirrors the existing Home-page pattern); Home page's "Network Health" card and the
Alerts summary card already pointed to the correct URLs (`/dashboard-operations/network-control`,
`/dashboard-system/alerts`) — no change needed there. Confirmed via the HA REST API
(`/api/states`, `/api/history/period`) that AP/ASUS CPU, memory, and ping-RTT sensors have zero
recorder history — excluded by the `sensor.ap_*_cpu_*` / `sensor.asus_*_cpu_*` / `sensor.*_ping_*`
recorder globs in `configuration.yaml` — so those are shown as gauges/badges (current state only)
rather than history graphs; WAN quality, NAS, and UPS metrics do have recorder history and got
trend graphs instead. Discovered a live stale-state bug in the alert severity pipeline while
validating dashboard data — see BUG-NET06 above and NETWORK_CONTRACT.md. File backed up first
(`lovelace.dashboard_operations.bak.20260710_125027`) per established `.storage` edit precedent;
JSON validity and all ~113 referenced entity IDs confirmed live via the API before writing.
**Restarted and verified 2026-07-10** — HA 2026.7.1, core state RUNNING, both views confirmed
intact post-restart (network-control 3 sections / network-debug 5 sections, `max_columns: 3`),
no lovelace/template errors in `ha core logs`, all referenced entities spot-checked live. The
restart also force-recomputed BUG-NET06's stale severity sensor back to a correct `none`/`normal`
reading — confirmed self-cleared for now, root cause (no periodic re-trigger) still open.*

*2026-07-10 (network dashboard polish pass, same day) — User screenshotted both dashboards
post-restart and flagged three things. Fixed one real defect: `network-debug`'s 5 per-AP
restart buttons were each a standalone `button`-type card outside any grid wrapper, so each
rendered at full section-grid size instead of sharing a row — consolidated into one shared
`columns: 4` grid (`lan_wireless_restart_grid`), same pattern as the already-correct
`network-control` restart grid. Investigated the "where's the ASUS restart button" question:
checked the device registry, confirmed the ASUS ROG router (GT-AX11000, 192.168.1.1) has zero
button/switch entities in HA (monitored via `ping` + a stats-only source, no `asuswrt`
integration) — there's genuinely nothing to wire up, not a dashboard gap. Diagnosed the
"Failed to fetch dynamically imported module" error and garbled entity-row icons the user saw
as a stale frontend JS-chunk cache from the HA version bump (identical symptom across unrelated
cards on both dashboards) — a hard refresh fixes it, not a YAML issue, no dashboard change made
for it. Also added a "Total Known Clients" line to the network-control LAN table (sums the
sensors that have live client counts; explicitly notes which are excluded and why). Re-patched
`.storage/lovelace.dashboard_operations` (backed up first to
`lovelace.dashboard_operations.bak.20260710_133824`). **Not yet restarted/verified live as of
this entry** — see OPEN TODO above.*

*2026-07-10 (log-review session) — General `ha core logs` review (not tied to a specific
bug report), triggered by user request to "check through log issues and fix." Found and
fixed 5 real bugs across presence/security/power, plus corrected one stale OPEN TODO
attribution (see above). All changes validated via `ha core check`.*

*PRESENCE — `presence_marker_reset` (`presence_boundary.yaml`) was missing `mode:
restart`, dropping the 10s auto-clear of `last_arrival_person`/`last_departure_person`
when two people arrived/departed within the same 10s window ("Already running" in logs).
Fixed to match its sibling `presence_clear_arrival_flag`'s established mode. See
PRESENCE_CONTRACT.md BUG-P18.*

*SECURITY — `input_text.security_last_path` (`security_helpers.yaml`) was missing `max:
255`, silently rejecting the 5-zone movement-path writes from `security_track_movement_
path` once the joined zone names exceeded HA's default 100-char limit — this contract
already documented `max: 255` for this field, the YAML just never matched it. See
SECURITY_CONTRACT.md BUG-S64.*

*POWER (3 fixes, see POWER_CONTRACT.md Issue 20 for full detail):*
*(a) `geyser_turn_on` (`geyser_automations.yaml`) had `mode: single` with several
same-second triggers (04:00 winter-weekday/Saturday; critically, 14:00 `midday` vs
`midday_force_check`) — a losing trigger was silently dropped via "Already running,"
observed live at 04:00:00.398 on 2026-07-10. Changed to `mode: queued`, matching existing
precedent elsewhere in the same file.*
*(b) `sensor.inverter_production_7d_mean`/`_30d_mean` (`power_statistics.yaml`) paired
`device_class: energy` with `state_class: measurement` — an invalid combination per HA's
device class registry (energy requires total/total_increasing/None), since these are
derived daily averages not cumulative totals. Removed `device_class: energy`; confirmed
via `.storage/energy` neither sensor feeds the Energy dashboard.*
*(c) `sensor.house_power` (`power_templates.yaml`) passed `sensor.inverter_load_power`'s
raw state straight through with no numeric guard, causing a `template.validators` ERROR
whenever the solarman source briefly went `unavailable`. Added `| float(0)`, matching the
sibling `Battery SOC` sensor's existing pattern in the same file.*

*Investigated but NOT changed (already fine or out of scope for YAML):* the prepaid
tariff-rate dict lookup warning in `prepaid_strategy.yaml` was already fixed in a prior
session (`.get(this.state, 0)` already in place — the log line predated that fix).
hikvision_next malformed-event warnings, Sonoff cloud 504s, EcoFlow MQTT keepalive drops,
solarman `total_increasing` noise, and the daily `gitupdate.sh`→`sync_docs.sh` chain's one
observed `return code: 1` at 05:00:05 (re-tested manually post-session — clone/push both
succeeded, looks like a transient network blip, not a reproducible bug) are all
infra/third-party-integration-level and not expressible as a package YAML fix — consistent
with this repo's existing pattern for this class of issue.*

*2026-07-09c (security S17d) — Fixed three related wrong-camera-image bugs in
`security_automations.yaml`'s router (BUG-S62, SECURITY_CONTRACT.md): (1) RUNG 5
visitor notification looked up its image via `zone_label` — a general-purpose
zone ladder that checks inside zones first — instead of reading the
perimeter-front image slot directly, so a concurrent inside-camera event could
substitute the wrong image; now reads `input_text.security_image_perimeter_front`
directly, message text also corrected to drop the now-redundant `{{ zone }}`
reference. (2) RUNG 5c gate_activity image read the shared
`security_image_grounds_front` slot (written by ipcam03 driveway + cam04
carport + cam07 kitchen), so a concurrent cam07 event could show a kitchen
image for a driveway event; now prefers the ipcam03-exclusive
`ipcam03_driveway_history` slot, same pattern BUG-S48 already applied to
Arrival Stage 1. (3) Departure notification had the identical gap the arrival
branch immediately above it in the same file had already fixed for BUG-S48;
same `ipcam03_driveway_history`-preferred fix applied. No new entities.*

*2026-07-09d — Diagnosed and fixed three alert families firing false positives on
every HA restart/reload (user-reported via notification screenshot): prepaid
critical/warning alerts, security "Perimeter activity", network "Device Down"/"WAN
Down". Root cause common to all three: post-restart/reload, an upstream
integration or template sensor briefly reads `unavailable` or a default value
before it has reconnected/recomputed, and downstream logic treats that
valid-looking-but-wrong reading as real. `not_from`/`not_to` unknown/unavailable
guards can't catch this class of bug because the bad value is never literally
`unknown` — it's a plausible number or string computed from defaults.*

*PREPAID (packages/power/prepaid_core.yaml, prepaid_strategy.yaml): closes the
"NOT fixed" half of Issue 18 (POWER_CONTRACT.md). Added `for: "00:01:00"` to the
5 affected automations' triggers (`prepaid_low_units_warning`,
`prepaid_critical_units_alert`, `prepaid_critical_night_protection_alert`,
`prepaid_strategic_top_up_suggestion`, `prepaid_buy_decision_notify`) to bridge a
template reload, plus a `input_boolean.system_startup == off` condition on each
to bridge the ~1-2min post-restart window while `sensor.grid_energy_import_total`
(solarman) reconnects. See POWER_CONTRACT.md Issue 19.*

*SECURITY (packages/security/security_logic.yaml): RUNG 6 (`perimeter_threat`)
never excluded `guest_mode`, unlike RUNG 7/8 (intruder/grounds_low_confidence).
`suppress_security_after_restart` (presence_boundary.yaml) already forces
`guest_mode` ON for 2min after every HA start specifically to cover this
race — RUNG 6 just wasn't wired to check it. Added `and not guest`. No new
mechanism, reused existing infra. See SECURITY_CONTRACT.md BUG-S63.*

*NETWORK (packages/alerts/alerts_network.yaml): `group.network_devices` /
`group.wan_services` are `all: true` — a member reading `unavailable` while
UniFi/ping integrations reconnect post-restart makes the whole group read `off`,
same as a real outage; if reconnect takes longer than the 250s anti-flap window,
`network_device_down_alert_active`/`wan_down_alert_active` legitimately (per their
own formula) flip on. Added `input_boolean.system_startup == off` to both.
Separately, `sensor.network_devices_down_list`/`wan_services_down_list` were
missed by BUG-NET04's 2026-04-21 fix (which patched the severity sensor and
context devices list but not these) — still filtered literal `state == 'off'`
only, so an `unavailable` member wasn't listed by name; extended selectattr to
include `unavailable`. See NETWORK_CONTRACT.md BUG-NET05.*

*Note on `system_startup` (core/core_helpers.yaml): actual duration is 2 minutes
(`delay: "00:02:00"`), not 60 seconds as PRESENCE_CONTRACT.md §"Startup Guard"
previously stated — pre-existing doc drift, not introduced this session.
Corrected in PRESENCE_CONTRACT.md same session since both new fixes above reuse
this exact boolean.*

*2026-07-09b — City Power inclining block tariff modeled; live per-purchase block
warnings added; prepaid spend/block-kWh history corrected and extended:*

*POWER — User supplied the 5 remaining "gap" purchase receipts (May24'25, Aug2'25,
Apr16&23'26, Jun17'26) that were previously estimated, not real — every one of the
75 purchases in the Dec 2024-Jun 2026 dataset now has a real receipt, no estimates
left. One real correction found: Aug 2025's Aug2 receipt showed a R70 fixed charge
not previously known, shifting that month's split from R130/R927 (fixed/energy) to
the correct R200/R857 (total R1057 unchanged). Historical statistics for
sensor.prepaid_actual_spend_this_month/prepaid_month_energy_spend/
prepaid_month_fixed_charge were re-imported with the correction (same
pyscript stop/edit/start pattern as 2026-07-09a).*

*User located and provided City Power Johannesburg's official FY2025/26 tariff
bulletin (joburg.org.za Consolidated-Tariffs-FY20252026.FINAL.pdf). Residential
Prepaid High: Block 1 0-350kWh @ R2.6645/kWh, Block 2 350-500kWh @ R3.0564/kWh,
Block 3 >500kWh @ R3.4826/kWh, Fixed R200/month (R70 service + R130 capacity) —
the R200 figure is an exact match to what this dataset already showed empirically,
good independent confirmation. Block thresholds unchanged from FY2024/25 (only
rates increased), so block ASSIGNMENT is valid across the whole dataset; analysis
found 50/74 purchases stayed entirely in Block 1, 16 touched Block 2, 8 reached
Block 3 — and at the month level, 12/19 months exceeded 350kWh and 5/19 exceeded
500kWh (May'25, Jun'25, Jan'25, Feb'25, Jun'26).*

*NEW FEATURE — tariff block awareness is now live, not just historical analysis:*
*• 5 new input_numbers (prepaid_helpers.yaml): prepaid_tariff_block1_threshold/
_block2_threshold/_block1_rate/_block2_rate/_block3_rate — editable constants
(not hardcoded), since these change every fiscal year.*
*• script.update_prepaid_units (prepaid_core.yaml) now computes, per real
purchase, how many kWh landed in each block (given month-to-date total from
input_number.prepaid_month_units BEFORE this purchase) — stored in new
input_number.prepaid_last_topup_block1_kwh/_block2_kwh/_block3_kwh, and
accumulated into input_number.prepaid_month_block1_kwh/_block2_kwh/_block3_kwh
(reset by prepaid_month_counters_reset, same cadence as prepaid_month_units).
The "Prepaid Top-Up Recorded" notification now lists the block breakdown and
escalates severity to warning if any kWh landed above Block 1.*
*• sensor.prepaid_current_tariff_block (prepaid_strategy.yaml) — live status
(1/2/3) + attributes kwh_this_month/kwh_to_next_block/current_rate. The
proactive "Smart Top-Up Decision" notification (prepaid_buy_decision_notify)
now includes this context so a suggested top-up comes with tariff awareness,
not just a Rand amount.*
*• sensor.prepaid_month_block1_kwh/_block2_kwh/_block3_kwh (prepaid_core.yaml)
— graphable wrappers, same state_class:total pattern as prepaid_month_energy_spend.
New "Monthly kWh by Tariff Block" plotly-graph card added to
lovelace.dashboard_operations (view 11 section 2, right after "Monthly Prepaid
Spend") — separate graph, not merged into the existing Energy/Fixed Rand-spend
bar, since it's a different axis (kWh consumption tier, not Rand category).*

*BUG CAUGHT MID-SESSION — modern `template:` platform does NOT support
`object_id` (legacy-only key); using it silently drops the whole sensor
definition with no visible error except in core logs ("'object_id' is an
invalid option for 'template'"). Root cause of a naming collision: sensor name
"Prepaid Month Block 1 kWh" (space before digit) slugifies to
`sensor.prepaid_month_block_1_kwh` (extra underscore), not the
`sensor.prepaid_month_block1_kwh` used everywhere else (dashboard card, import
script) — caused historical statistics to import into a real statistic_id with
no matching live entity. Fixed by renaming to "Prepaid Month Block1 kWh" (no
space before digit) instead of trying to force entity_id via object_id. Also
had to manually clear 3 stale entries from `.storage/core.entity_registry`
(entity registry entity_id assignment is sticky — a template.reload alone does
NOT rename an already-registered entity even after the source config's implied
entity_id changes; needs the stale registry row removed, or a full restart with
the registry pre-cleared, before it re-registers under the new id).*

*Historical backfill extended to all 6 sensors (3 corrected, 3 new) via one more
pyscript stop/edit/start cycle (same 2026-07-09a pattern) — verified live via
recorder.get_statistics, all 19 months intact on the correct entity_ids after
the object_id fix + entity registry cleanup + final restart.*

*2026-07-09a — Prepaid spend history backfilled Dec 2024 → Jun 2026 (real data, both accounts); dashboard restart + template reload completed:*

*POWER — Closed both 2026-07-08 OPEN TODO items. User supplied real per-purchase detail across many sessions of screenshots (City Power "Token information" popups and "Transaction Details" screens from the Discovery app) plus a personal Tymebank spreadsheet with exact Amount/VAT/Fixed/kWh breakdowns per purchase — reconciled into 70 real data points, Dec 2024 through Jun 2026, covering two parallel purchase streams (Discovery-app and Tymebank, no overlapping purchase dates found). Monthly totals: Total, Fixed (service charge), Energy (Total−Fixed) — R0 fixed-charge default used only for the handful of purchases where no service-charge line appeared on the receipt (R210/month blanket assumption from earlier in the session was retired once real per-purchase data made it unnecessary). R/kWh across the full dataset: R2.72–R3.98, consistent with expectation.*

*IMPORTANT DISCOVERY — `recorder.import_statistics` is NOT a callable HA service in this install (confirmed live: `recorder` domain only exposes `purge`/`purge_entities`/`enable`/`disable`/`get_statistics`; the import function is Python-internal only, requires direct `hass` object access). Backfilled via a temporary one-off `pyscript` script (`backfill_prepaid_statistics.py`, deleted after use) that called `homeassistant.components.recorder.statistics.async_import_statistics` directly. This required temporarily flipping the pyscript integration's `allow_all_imports` and `hass_is_global` config options to `true` (`.storage/core.config_entries`) — **both correctly reverted to `false`/`false` after the import ran**, confirmed post-revert. Learned mid-task: editing `.storage/*` files while HA is still running does NOT persist — HA holds its own in-memory copy and flushes it back over any manual edit during a restart's shutdown phase. The edit only sticks if made while HA is fully stopped (`ha core stop` → edit → `ha core start`, not `ha core restart`). Used this stop/edit/start pattern for both the temporary grant and the revert (two full stop/start cycles this session).*

*Backfilled `sensor.prepaid_actual_spend_this_month`, `sensor.prepaid_month_energy_spend`, `sensor.prepaid_month_fixed_charge` — one long-term-statistics point per month (2024-12 through 2026-06), `state` = that month's real total, `sum` = running cumulative (Jun 2026 cumulative: R27,243 combined = R3,516.50 fixed + R23,726.50 energy — energy+fixed cross-check matches combined exactly). Verified live via `recorder.get_statistics` after the import and again after both restarts to confirm the data survived. Same restart also activated the two new template sensors and the "Monthly Prepaid Spend" stacked-bar dashboard card from 2026-07-08 (both previously pending a restart) — confirmed live afterward (`sensor.prepaid_month_energy_spend`/`_fixed_charge` reporting real July-so-far values).*

*Known small gaps, accepted as-is (immaterial, R0-fixed-default per purchase): 2025-05-24 (R112), 2025-08-02 (R140), 2026-04-16 & 2026-04-23 (R500 each), 2026-06-17 (R154) — per-purchase Fixed/Energy split for these few purchases defaults to Fixed=R0, but the month's real Total is correct in all cases (matches bank/app statement totals exactly). Dec 2024 may extend earlier than the 13th — unconfirmed, not chased further.*

*2026-07-08 — Prepaid dashboard: added Monthly Spend graph alongside Monthly Usage; investigated whether historical (pre-2026-07-03) top-ups are reconstructable:*
*POWER/DASHBOARD — User asked for a "spend per month" graph matching the existing "Monthly Grid Usage" one. Before building, user pushed back on the assumption that no history exists, asking "each top up is recorded in the prepaid process is it not?" — investigated `script.update_prepaid_units` (`prepaid_core.yaml:857-989`) directly rather than re-assert the earlier claim. Finding: yes, every top-up is recorded, but only as (a) a Logbook text entry and (b) increments to a handful of running-total `input_number`s (lifetime + this-month/this-year, the latter added 2026-07-03) — there is no structured per-purchase ledger (no date+amount row) anywhere. Critically, `configuration.yaml:170` sets `recorder: purge_keep_days: 90` — the raw state-history of those counters (the only place a per-purchase timestamp+delta could theoretically be reconstructed from) is hard-deleted after 90 days, not just unqueried, and HA's long-term statistics (which never purge) only apply to `sensor`-domain entities with `state_class` — `sensor.prepaid_actual_spend_this_month` didn't exist before 2026-07-03. Net: top-ups older than ~90 days are already gone from the database entirely; nothing before 2026-07-03 is recoverable from HA at all. User accepted starting the graph from now.*
*FIX — Added a new `custom:plotly-graph` card "Monthly Prepaid Spend" to `lovelace.dashboard_operations` (view 11, section 2, right after the existing "Monthly Grid Usage" card), same `statistic: state, period: month` pattern, sourcing `sensor.prepaid_actual_spend_this_month`. File backed up first (`lovelace.dashboard_operations.bak.20260708_232914`) per the established `.storage` edit precedent; JSON validity confirmed after edit.*
*⚠️ Requires a full HA restart to take effect (`.storage/lovelace` changes are not picked up by a browser hard refresh — confirmed unreliable twice already, see CODING_STANDARDS.md Reload vs Restart Rules) — not yet restarted this session. Also: do not open this dashboard's UI editor before restarting, or its autosave will silently overwrite this direct edit.*

*2026-07-08 — Priority-fix batch: water reporting/race bugs, prepaid rolling-vs-calendar-month split, security Sprint 3 doc-drift closed; two more doc-drift-only findings along the way:*
*WATER (Issue 1) — `sensor.water_refill_avg_flow_this_week`/`_last_week` always read 0; both referenced `sensor.water_refill_flow_rate`, which was never defined, and `_last_week` also referenced a `mean_7d` statistics attribute that never existed on any platform (adding the missing sensor alone would only have fixed "this week"). FIX: added `sensor.water_refill_flow_rate` (`platform: statistics`, source `sensor.water_refill_cycle_avg_flow_rate`, `state_characteristic: mean`, `max_age: days: 7`, rolling not calendar) to `water_reporting.yaml`. For "last week", added `input_number.water_refill_avg_flow_last_week` (`water_helpers.yaml`) + new automation `water_snapshot_weekly_avg_flow` (`water_maintenance_automations.yaml`, Monday 00:00 — same cadence as the pre-existing `water_reset_weekly_fault_counter`) that freezes the current rolling value before the new week starts. `sensor.water_refill_avg_flow_last_week` now reads that input_number.*
*WATER (Issue 3) — `input_number.water_refill_start_depth` was double-written: `water_tank_refill_control.yaml` wrote the VALIDATED depth just before turning the pump on, in all 6 of its refill branches (Safety/Critical/Emergency/Demand-target/Force-override/Predictive-fill), then `water_capture_refill_start` (`water_refill_capture.yaml`) fired ~10s later and overwrote it with the RAW depth — the validated write was pure waste and directly violated `a_water_lifecycle_contract.yaml`'s own explicit invariant ("Depth values: RAW sensor (Tuya). Validation: analytics-only (never capture)."). FIX: removed the write from all 6 control branches (each replaced with a one-line ownership comment), leaving `water_capture_refill_start` as sole writer — mirrors that same file's pre-existing "let capture control this" pattern already used for `water_refill_cycle_active`. `input_datetime.water_refill_start`'s parallel dual-write was left untouched (not part of the reported bug, and not contract-violating).*
*WATER (Issue 8) / NOTIFICATIONS (BUG-N02) — both contracts claimed `water_notifications.yaml` referenced a wrong counter name (`counter.water_borehole_faults_week` instead of `_this_week`). Re-verified live: the code already reads the correct `_this_week` name — grep-confirmed zero occurrences of the wrong name anywhere in `packages/`. Doc-only fix (third doc-drift finding closed this week, after BUG-P07/SECURITY-ISSUE-3 on 2026-07-08 earlier today) — the NOTIFICATIONS_CONTRACT.md entry also had the wrong file name (`notify_water_events.yaml` instead of the actual `water_notifications.yaml`), corrected too.*
*POWER — `sensor.prepaid_monthly_usage_true` was labeled "Rolling 30-Day Grid Usage" but its logic actually mirrors the last COMPLETED calendar month (or month-to-date) — flagged mislabeled 2026-07-03, still open. User's fix instruction: keep a rolling concept, but make sure calendar-month is a properly separated, correctly-labeled thing rather than one sensor conflating both. FIX: renamed the existing sensor's friendly name only to "Prepaid Calendar Month Usage" (entity_id/unique_id `prepaid_monthly_usage_true` left untouched — renaming those would have orphaned the entity in the registry and broken any dashboard binding, since `platform: template` unique_ids are namespaced per-integration and swapping platform types creates a new entity even with the same unique_id string). Added a genuinely separate NEW sensor, `sensor.prepaid_rolling_30day_usage` (`platform: statistics`, source `sensor.grid_energy_import_total` — the underlying `total_increasing` template sensor already used by the `prepaid_import_*` utility meters — `state_characteristic: change`, `max_age: days: 30`), which is the actual sliding-window sensor that was missing.*
*SECURITY — Sprint 3 checklist item "change triggers on grounds/rear/house automations to use zone binary_sensors instead of sensor.security_correlation" was still unchecked, read by a research pass as an open bug. Investigated live: `security_grounds_motion`/`security_rear_grounds_motion`/`security_house_motion` don't exist anywhere in `security_automations.yaml` or the legacy `automations.yaml` — they were deleted outright in the 2026-05-17 S3 rebuild (already documented as done in Section 6 ISSUE 2, "✅ RESOLVED 2026-05-17 (S3)") and replaced by `security_event_router` reading `sensor.security_event_classification`, which already sources the dedicated zone binary_sensors (`security_perimeter_front_motion`/`_perimeter_rear_motion`/`_grounds_motion`), not the old correlation sensor. The underlying goal was achieved via a more thorough rebuild than the checklist's originally-planned in-place trigger swap — the checkbox itself was just never ticked. Doc-only fix, no code change (fourth doc-drift-only finding today).*
*TESTING — `ha core check` clean after all water/power code changes.*
*DOCS: WATER_CONTRACT.md Issues 1/3/8 marked fixed/closed (+ entity tables, Sprint 3 checklist). NOTIFICATIONS_CONTRACT.md BUG-N02 closed (+ file-name correction). POWER_CONTRACT.md prepaid sensor table entry rewritten (mislabel closed, new rolling sensor documented). SECURITY_CONTRACT.md Sprint 3 checklist Issue 2 line closed.*

*2026-07-08 — Pedestrian front-entrance arrivals never turned on entrance/dining lights at night (BUG-P17):*
*PRESENCE/LIGHTING — User reported that returning home after 9pm via the front security gate + front door (walking in) never turned on `entrance_down_lights` or `dining_room_light`, even though driving in through the driveway gate works fine. Investigated live rather than assuming a lighting-side bug (LIGHTING_CONTRACT.md's own BUG-L15/L12/L13 arrival-scenario bugs were already closed). Root cause: `binary_sensor.main_gate_sensor` (area `main_gate`, driveway/vehicle gate) and `binary_sensor.front_security_gate_sensor`/`binary_sensor.front_door_sensor` (both area `entrance`, the pedestrian gate+door) are different physical hardware — confirmed via `.storage/core.device_registry`. Every path that sets `input_boolean.arrival_detected` (the sole trigger for `automation.lighting_arrival_night`) was keyed off `main_gate_sensor` or a vehicle-approach camera (`presence_boundary_resolver`; `security_gate_vehicle_stage1`). `house_entry_event` (`presence_boundary.yaml`) is the one automation that actually sees the gate→door sequence, but it only toggled its own boolean + sent a notification — it never touched `arrival_detected` and had no other consumers, so a walking arrival through the front entrance never reached `lighting_arrival_night` at all, regardless of quiet-mode/bedtime state.*
*FIX — `house_entry_event`'s action block now also turns on `input_boolean.arrival_detected` (`presence_boundary.yaml`), reusing the existing `presence_clear_arrival_flag` 5-min auto-clear. No new entities; `ha core check` passed. Reload via Developer Tools → Reload Automations (no restart needed — automation-action-only change).*
*DESIGN DECISION (same session) — user separately asked whether front+back security lights and patio should turn on before bedtime (only back security after) for this same arrival. Confirmed this is already exactly what `lighting_arrival_night`'s Scenario 2 ("Someone Home") does (BUG-L15, closed 2026-06-28); user explicitly chose to leave quiet mode (kids already asleep) minimal as-is rather than adding front security/patio there too — no lighting-side change made.*
*DOCS: PRESENCE_CONTRACT.md BUG-P17 added (Section 10 + Section 13 summary table); Section 5 "Arrival Persistence"/"Door Correlation" updated to drop the stale "arrival_detected never set by boundary resolver" note (superseded by BUG-P06, already fixed 2026-05-17) and document the new house_entry_event wiring. LIGHTING_CONTRACT.md Section 8 cross-domain table: added `input_boolean.arrival_detected` row.*

*2026-07-07 — HA restart performed (closes BUG-A11) + garden TURN_OFF_POND_PUMP action button restored (BUG-A12):*
*INFRA — Triggered the full HA core restart that BUG-A11's `alert.camera_health` notifier fix (2026-07-06) had been waiting on since it can't be reloaded piecemeal. Ran via `ha core restart`; confirmed live afterward via `.ha_run.lock` timestamp change and `ha core info` (2026.7.1, healthy) — `alert.camera_health` loads cleanly with no `notifiers:` key, no more `ServiceNotFound` on repeat reminders.*
*NOTIFICATIONS/ALERTS — Same session, closed the other 2026-07-06 OPEN TODO item: `alerts_garden.yaml`'s `TURN_OFF_POND_PUMP` mobile action button had delivery restored by BUG-A10 but still couldn't be tapped, because `script.notify_system_event` had no way to attach action buttons to the push. Added an optional `actions` field to the script (passthrough on warning/critical branches only — info uses `notify.send_message`, which structurally rejects any extra `data` key, same bug class as BUG-N13/N14) — uses HA's `omit` Jinja global so callers that don't pass `actions` see zero behaviour change. `alerts_garden.yaml`'s `route_garden_alert` automation now passes the button; the existing tap handler (`automation.garden_alert_ack_turn_off_pond_pump`) needed no changes.*
*TESTING — `ha core check` clean. Post-restart, live-fired `script.notify_system_event` via direct Supervisor API call twice: once with `actions` set (pond-pump case), once without (backward-compat check representing every other existing caller, e.g. temperature domain) — both completed cleanly (script state on→off, nothing in the HA error log), confirming the new template logic doesn't break existing callers. Both test calls sent real pushes to all 4 devices; user asked to visually confirm the button renders and works on next tap.*
*SESSION-CONTINUITY NOTE — the code for this fix (`notify_system_event.yaml` + `alerts_garden.yaml`) was actually written and validated in a 2026-07-07 session whose terminal disconnected before `./gitupdate.sh` ran; it reached disk correctly but was picked up by the unattended 05:00 daily-backup cron the next morning (commit `ffd92d3`, generic message) instead of a clean session-close commit. No content was lost — confirmed via `git reflog`/`git stash list` (both clean) — but the descriptive commit message and this doc update were owed and are being closed out now.*
*DOCS: ALERTS_CONTRACT.md BUG-A11 marked restart-complete, BUG-A12 added (+ Section 9/10 tables updated). NOTIFICATIONS_CONTRACT.md §5 "Extended Fields" section added for notify_system_event.yaml. PROJECT_STATE.md OPEN TODO items 1 and 2 cleared (both now closed).*

*2026-07-06 — Weather stale-alert false positive fixed (BUG-WEA01 corrected):*
*INFRA — User asked why the weather integration keeps firing recurring "stale data" alerts. Investigated live rather than trusting INFRA_CONTRACT.md's existing BUG-WEA01 entry, which claimed `weather_api_stale_alert` had no action block and was a no-op — that claim was itself stale; the automation has always called `script.notify_system_event`. Root cause: `sensor.weather_last_update_age` (weather_core.yaml) computed staleness from `weather.openweathermap.last_changed`, which only updates when the entity's state STRING (the condition, e.g. "sunny") changes — not on every poll. Confirmed live: state=sunny, last_changed=05:00 (9h stale by that measure), but last_updated=13:42 (17 min old, i.e. actively polling fine) — the condition just hadn't changed all day (typical Johannesburg winter clear spell). 3-day history of sensor.weather_api_health showed a repeating healthy→delayed→stale→reset cycle, confirming this fires falsely roughly once or twice a day, not a real API/connectivity problem.*
*FIX: weather_core.yaml's Weather Last Update Age sensor switched from `entity.last_changed` to `entity.last_updated`. Reloaded live via `template.reload` — sensor.weather_api_health confirmed back to "healthy" immediately (age dropped from 32288s to 1152s). Also cleared `input_boolean.openweathermap_api_limited`, stuck `on` since 2026-07-04 as a downstream side effect (the matching recovery automation's `not_from: [unknown, unavailable]` guard blocked it from firing during the reload transition — separate minor gap, not fixed).*
*DOCS: INFRA_CONTRACT.md BUG-WEA01 corrected in place (old "no action" claim marked wrong, real bug + fix documented).*

*2026-07-06 — Geyser morning extension reworked: holiday_mode + manual override wired in, "until it stops heating" mechanism (replaces fixed 60-min timer), tiered safety caps, "more than one home" quantifier; PLUS unrelated presence bug found + fixed (BUG-P15/BUG-L16):*
*POWER — User reported wife's bath was cold after 10am on a holiday morning — no wake-up/school run means showers happen later, more like a lazy weekend, sometimes into early afternoon. The 2026-06-21 morning-extend mechanism (geyser_automations.yaml Branch 1/1b) only triggered on winter + cold ambient + poor solar + whole family home, and extended by a fixed `geyser_morning_extend_minutes` (60 min default) regardless of whether the geyser was actually still heating by then — a guess, not a measurement.*
*FIX 1 (holiday + manual override triggers) — Branch 1's extend condition now also fires when `input_boolean.holiday_mode` is on (existing shared boolean, already wired into `security_logic.yaml` threat escalation and `lighting_bedtime.yaml` bedtime scheduling — reused, not duplicated, per user's note that it "can also be used for other things like lights and bedtimes"), OR when new `input_boolean.geyser_morning_extend_override` is manually turned on — a same-day, geyser-only equivalent of holiday_mode for when plans change without wanting the security/bedtime side-effects. Override auto-clears at 00:01 (same convention as other `*_override`-suffixed helpers).*
*FIX 2 (measure, don't guess) — Branch 1 now also requires `binary_sensor.geyser_at_temperature == 'off'` (still actually drawing heating power) before agreeing to extend. Branch 1b no longer waits for a fixed timer — it fires directly off `geyser_at_temperature` transitioning to `on` (power draw < 50W, heat pump actually finished), i.e. "run until it stops heating."*
*FIX 3 (more than one person, not "everyone") — the cold/poor-solar trigger's presence check changed from `binary_sensor.all_family_home` (universal quantifier — all 4 of ryan/vicky/luke/tayla) to an inline count of family members in a home AP zone, requiring **more than one** — a single person home alone doesn't need the extra morning capacity the way 2+ people showering does.*
*FIX 4 (tiered safety caps) — three fallback cutoffs (only reached if the at-temperature sensor never trips, e.g. an element fault — fires a **warning** notification, not information): **09:00** normal cold/poor-solar day (`geyser_morning_extend_max_hour`, lowered from an initial 11:00 draft — covers a later gym-then-shower slot or longer winter shower without running needlessly long, and doesn't waste grid since the dynamic mechanism already turns off once actually hot); **10:00** on a maid day, Mon/Thu (`geyser_morning_extend_maidday_hour`, new — extra household hot-water demand those days); **13:00** on holiday_mode or manual override (`geyser_holiday_extend_max_hour`, unchanged).*
*`ha core check` passed; `input_number`/`input_boolean`/`automation` reloaded live via Supervisor API — all new/changed entities confirmed live (had to also push the live value of `geyser_morning_extend_max_hour` from its old 11.0 to the new 9.0 default via `input_number.set_value`, since a domain reload doesn't reset an existing entity's current value to a changed `initial:`).*
*KNOWN FOLLOW-UP (not done this session): `.storage/lovelace.dashboard_operations` still has a "Morning Extend Min" row bound to the now-removed `input_number.geyser_morning_extend_minutes` — will show Unavailable until manually swapped for the new cap entities via the dashboard UI editor (per 2026-07-03 precedent, `.storage` dashboards are cached in memory and risky to hand-edit while HA is running).*
*PRESENCE/LIGHTING — While rewriting the "everyone home" check, found `presence_confidence.yaml` has two top-level `template:` keys — same bug class as the `power_statistics.yaml` incident (2026-06-23). Confirmed live: Office/Garage/Bedrooms/Living Areas/Bar presence-confidence % sensors AND their Occupied binary sensors (10 entities, all in the first `template:` block, silently dropped the moment a second `template:` key was added for BUG-P13 on 2026-05-17) were sitting on stale `unavailable`/`restored` state — likely since that date, ~7 weeks. This directly explains the user's separately-raised question about office/garage lights not turning off: `lighting_office_presence.yaml`/`lighting_garage.yaml` trigger their turn-off action on these entities going `from: "on" to: "off"`, which an `unavailable` entity can never do. Also affected (same root cause, not just office/garage): `bar_occupied` (patio delay gate, bar_bedtime_cutoff), `bedrooms_occupied` (bar quiet-mode check), `living_areas_occupied` (morning wake trigger). FIX: merged the two `template:` keys into one (no logic change) — confirmed live, all 10 entities now report real values. See PRESENCE_CONTRACT.md BUG-P15, LIGHTING_CONTRACT.md BUG-L16 for full detail.*
*DOCS: POWER_CONTRACT.md Morning extension section rewritten. PRESENCE_CONTRACT.md BUG-P15 added (+ summary table). LIGHTING_CONTRACT.md BUG-L16 added.*

*DASHBOARD — Same session, follow-up: fixed the `.storage/lovelace.dashboard_operations` "Entity not found" row left by the `geyser_morning_extend_minutes` removal above. Backed up the file first (`lovelace.dashboard_operations.bak.20260706_153712`). In the Appliance Control view's Geyser section: replaced the dead entity row with `geyser_morning_extend_max_hour`/`geyser_morning_extend_maidday_hour`/`geyser_holiday_extend_max_hour`/`geyser_morning_extend_override` (Geyser Controls entities card); per user's chosen layout, moved the "Manual Run" mini-card out of the top 2×2 status grid into its own thin full-width card below, and put a new "Today's Cycles" markdown card (Morning/Midday/Evening kWh + the existing `midday_adequacy` outlook text, styled like the Pool Pump status cards) in Manual Run's old grid slot; removed the now-redundant plain Morning/Midday/Evening kWh + Evening Outlook attribute rows from the Geyser Status entities card since that's now covered by the new card. JSON validated after each edit.*

*CODING_STANDARDS CORRECTION — User confirmed a browser hard refresh did NOT pick up this dashboard change and a full HA restart was needed — the second time this has been observed (first was 2026-07-03). `CODING_STANDARDS.md`'s "Reload vs Restart Rules" table had this backwards: the `.storage/lovelace` row sat under the "⚠️ FULL RESTART REQUIRED" heading but its own text said "Browser hard refresh only... not HA restart" — direct contradiction, and now confirmed wrong by observed behavior twice. Fixed: row and the "Standard closing block" now both say `.storage/lovelace` changes require a full HA restart, hard refresh is not reliable.*

*2026-07-06 — Notification severity/sound classification overhaul across ALL domains (S17c follow-up), 13 dead alert-delivery domains fixed, two more silent critical-alert bugs found and fixed:*
*NOTIFICATIONS — User asked for a full run-through classifying every alert category into critical (alarm channel, DND bypass — visitors/intruders/perimeter, power outage escalated on low SOC+low solar, prepaid escalated very-low), warning (distinct audible sound, one tier below critical — family arrivals/departures, low prepaid, escalated info), and info (routine/silent — geyser run times etc). Two research passes across every notify script, the security classifier, and all 13 `packages/alerts/*.yaml` files found this was not a pure sound-mapping exercise — see below.*
*BUG FOUND — universal warning-tier sound missing everywhere except security: only `notify_security_events.yaml` had the S17c warning-tier treatment (iOS `time-sensitive` + Android `channel: security_warning`). Power/water/presence/system/lighting collapsed info+warning into identical delivery. FIX: converted the warning branch in `notify_power_event.yaml`, `notify_water_events.yaml`, `notify_presence_events.yaml`, `notify_system_event.yaml`, `notify_light_events.yaml` from plain `notify.send_message` to the legacy per-device `notify.mobile_app_*` pattern with the same iOS time-sensitive/Android `security_warning` channel — one shared warning sound reused across every domain (user's choice — no new on-device channel setup required).*
*BUG FOUND (live, previously silent) — `notify_power_event.yaml` and `notify_presence_events.yaml` critical branches were STILL calling `notify.send_message` with a nested `data: {push, channel, ttl, priority}` block — the exact "extra keys not allowed @ data['data']" bug fixed everywhere else on 2026-07-01, but these two scripts were missed at the time. This meant `power_battery_soc_critical_alert` (power_automations.yaml) and both prepaid-critical automations (prepaid_core.yaml, prepaid_strategy.yaml) — i.e. exactly the "power outage escalate to critical" and "prepaid very low" cases the user called out — had been silently failing to reach any phone, masked by `continue_on_error: true`. FIX: converted both critical branches to the same legacy per-device `notify.mobile_app_*` pattern already proven in `notify_security_events.yaml`.*
*BUG FOUND (live, previously silent, discovered while testing the fix above) — `notify_power_event.yaml`'s Telegram mirror for critical severity called `notify.send_message` targeting `notify.telegram_bot_5527` with `inline_keyboard` at top level of `data:` — this is the exact same rejection as the security Telegram crash already fixed in S17c (`extra keys not allowed @ data['inline_keyboard']`), just never applied to power. Live-reproduced via Developer Tools while testing the critical-severity fix above; confirmed it was firing on every real prepaid-critical alert (`Prepaid Critical Night Protection`, `Prepaid Buy Decision Notification` — both seen erroring live in `ha core logs`). FIX: switched to `telegram_bot.send_message` directly, same as security's S17c fix. The mobile push itself was unaffected (happens before this step) — only the Telegram Acknowledge button/message was silently missing.*
*ARRIVALS/DEPARTURES — promoted from `severity: "information"` to `"warning"` at all 4 call sites in `security_automations.yaml` (Arrival Stage 1/2, Departure Stage 1/2) per user's explicit request — always audible, no quiet-hours exception. The "Dogs home alone?" departure prompt stays `critical`, unchanged.*
*BROKEN PIPELINE FOUND AND FIXED — `notify.STD_Alerts`, the notify group used by nearly every `alert:` entity in `packages/alerts/*.yaml`, has been a broken `platform: group` since 2026-06-28 (bare `notify.*` services no longer exist post mobile_app migration) — see NOTIFICATIONS_CONTRACT.md §7, previously known but only partially mapped. Full audit this session found 13 domains with literally zero push delivery: network (4 subtypes), device power, media, system health, water tank-low, water borehole faults (tiers 2/3), presence anomaly, garden (which also meant its `TURN_OFF_POND_PUMP` mobile action button was unreachable — the action event only fires from inside a push that never arrived), dash battery, camera-health repeat reminders, door sustained-open escalation, and — highest priority — power grid-offline/battery-low warning tiers. FIX: added one routing automation per domain (in the same `packages/alerts/<domain>.yaml` file), mirroring the already-proven temperature-domain pattern (`alerts_temperature.yaml` Route WAN/LAN/Device/Storage Temp Alert) — trigger on the underlying binary_sensor/severity-sensor, call the working `script.notify_*_event` directly instead of relying on the dead `alert:` notifier. `alert.camera_health`'s `notifiers: [STD_Warning]` was removed outright (that group doesn't exist at all, unlike STD_Alerts which is merely broken-but-defined — every repeat was throwing a hard `ServiceNotFound`); this is the one `alert:` entity edited this session and needs a full HA restart (not yet done). Deliberately NOT fixed: `alert.security_alert`'s repeat reminders — the initial hit already works via the event router, and a naive on-transition trigger would reintroduce the exact BUG-A04 duplicate-delivery bug; real interval-based repeat logic is a separate follow-up.*
*LIVE INCIDENT DURING THIS SESSION (self-caused, fixed same session) — reloading `template:` to activate the above fixes caused a wave of false CRITICAL push notifications (user screenshot: spurious "Critical Sensor Health", "Network Alert — Device Down", "Water Tank Low — 0.0% Unavailable" etc.) because the 13 new routing automations triggered on bare `to: "on"`/`to: "critical"` with no guard against the transient unknown/unavailable state every template entity passes through while `template:` reloads — exactly the BUG-N10 reload-vulnerability class already fixed elsewhere in this codebase, which the new automations reintroduced. FIX (immediate, same session): added `from: "off"` to every binary_sensor trigger and `not_from: ["unknown","unavailable"]` to every severity-sensor escalation trigger across all 13 new automations, following the established Rule 7 pattern. Re-tested via direct script calls (not a second template reload) — clean. Note: the pre-existing temperature-domain "Route WAN/LAN/Device/Storage Temp Alert" automations (not touched this session) have the same missing-guard structure and were not audited/fixed here — flagged as a related latent risk, not in scope.*
*TESTING — `ha core check` clean throughout. Reloaded `automation`/`script`/`template` via Supervisor API (SUPERVISOR_TOKEN). Live-tested via Developer Tools-equivalent direct script calls: critical + warning severity for power/water/system/lighting/presence, and a security event with an image attachment (regression check — security's script was not touched this session; confirmed the only error was the pre-existing, already-documented BUG-S61 `ha.dunners.tech` photo-attachment infra issue, not a new regression). Could not flip real domain sensors (network/doors/etc.) to test the 13 new routing automations fully end-to-end without risking further false pushes — verified via config check + confirming each automation is registered/enabled + calling the script each automation invokes directly with representative severity. User asked to visually confirm final phone receipt.*
*Also observed (not fixed, no YAML cause found): intermittent `ServiceNotFound: notify.<device>` errors in core logs on some `notify.send_message` multi-entity calls, unrelated to any code in this repo (zero grep hits) — see OPEN TODO.*
*DOCS: NOTIFICATIONS_CONTRACT.md, ALERTS_CONTRACT.md, SECURITY_CONTRACT.md, POWER_CONTRACT.md, PRESENCE_CONTRACT.md all updated this session.*

*2026-07-03 — Notification pipeline: image attachments restored for all severities + Telegram inline_keyboard crash fixed + warning gets distinct sound/channel (S17c):*
*SECURITY — Following S17b, live-tested the "structurally impossible" claim from the 2026-07-01 outage notes rather than accepting it at face value. Confirmed: `notify.send_message` (used for info/warning) genuinely rejects ANY `data:` sub-field on this HA version (2026.6.4) — reproduced the exact `extra keys not allowed @ data['data']` error live, and confirmed it aborts the entire script (nothing downstream runs, not even the Telegram mirror). Fix: migrated info/warning off `notify.send_message` onto the same 4 legacy `notify.mobile_app_<device>` action calls the CRITICAL branch already used successfully — that path does accept `data.image`. Live-tested all three severities end-to-end after the change: all 4 devices fire cleanly with real image attachments for info, warning, and critical.*
*SECURITY — While testing, found a second, unrelated, more serious bug: the Telegram mirror for CRITICAL severity was crashing with `extra keys not allowed @ data['inline_keyboard']` (same root cause — `notify.send_message` targeting the telegram_bot notify entity doesn't pass through inline_keyboard either) — meaning the Telegram "Acknowledge" button, and therefore the entire Telegram copy of every critical security alert, had been silently failing for an unknown period (likely since the 2026-07-01 rewrite; predates tonight). Phone push notifications were unaffected since the crash happened in a later step. Fix: switched to the native `telegram_bot.send_message` action (same integration `telegram_bot.send_photo` two steps later already used successfully) instead of the generic notify wrapper — `inline_keyboard`/`parse_mode` are native fields there. Live-tested: Telegram message + Acknowledge button now arrive correctly.*
*SECURITY — Separately found (via the same live testing, NOT a code bug): `telegram_bot.send_photo` fails with "Failed to load URL: All connection attempts failed" for `https://ha.dunners.tech/...`. Root cause identified: `ha.dunners.tech` resolves internally to `10.10.1.5`, but nothing is listening on port 443 there right now (connection refused, not timeout) — a local DNS/reverse-proxy infrastructure issue, not a YAML bug. This was very likely masked until tonight because the inline_keyboard crash (above) was aborting the script before execution ever reached the send_photo step. Flagged to user for infra-side investigation (reverse proxy container status / LAN IP correctness) — not fixed, outside YAML scope.*
*SECURITY — Added distinct warning-severity treatment: iOS (ryan_iphone16promax) gets `push.interruption-level: time-sensitive` (breaks through Focus modes, one tier below critical's full DND-bypassing alarm channel); Android devices (ap_0223_1001, honor10/honorx7) get a new `channel: security_warning` notification channel, which needs a one-time on-device sound assignment (Settings > Apps > Home Assistant > Notifications > security_warning) since Android doesn't accept a sound file over the wire the way iOS's `push.sound` can.*
*DOCS: SECURITY_CONTRACT.md Section 6 (BUG-S60/S61) updated.*

*2026-07-03 — Prepaid purchase backfill (2 missed July 1 top-ups) + monthly/annual spend tracking added + dashboard cumulative-vs-monthly bug fixed:*
*POWER — User's phone showed 5 City Power purchases (Jun 18/23/29, and TWO on Jul 1) but `input_number.prepaid_total_units`/`prepaid_total_spent` hadn't moved since Jun 29. Confirmed via HA history (recorder) that Jun 17/18/23/29 were all correctly entered via `script.update_prepaid_units` on their actual dates — only both Jul 1 purchases (77.40 kWh/R290+R210 fixed-charge, voucher had a meter network timeout on original entry; and 149.70 kWh/R500) were never applied. User re-keyed the timed-out voucher into the physical meter and read fresh values (1.8.0=120267.04, C.51.0=192.08). Verified via meter math before writing anything: balance +140.03 kWh net, consumption 87.07 kWh (lifetime import delta) ⇒ 227.10 kWh actually purchased = exactly 77.40+149.70 to the cent — both vouchers confirmed landed on the meter, safe to backfill without double-counting.*
*FIX (reconciliation): meter fields set to fresh reading → `script.prepaid_realign_offset` run (drift 2.78→0 kWh) → `input_number.initial_prepaid_grid_import` manually reset to live `sensor.grid_energy_import_total` (NOT via `update_prepaid_units`, since that script also bumps `prepaid_units` balance — which the fresh meter reading already reflects; running it would have double-counted the 227.10 kWh into the balance a second time). Both purchases' units/cost/fixed-charge added directly to the lifetime running totals instead. Logged a Logbook note stating the true 2026-07-01 date (HA can't backdate the Logbook timestamp itself via the API — it shows 2026-07-03 — but the cumulative totals are date-agnostic so no financial figure is wrong).*
*BUG FOUND (live, blocking): `input_number.prepaid_fixed_cost_paid` had `max: 1000` and was already at 850 from unrelated prior accumulation — the R210 add-on was rejected (HTTP 400). Confirms these "lifetime" counters were never designed with real headroom. Raised `max` on `prepaid_total_spent`/`prepaid_total_units`/`prepaid_fixed_cost_paid`/`prepaid_total_discount` (prepaid_helpers.yaml) to 200000/200000/50000/50000.*
*CONFIRMED BUG (user's original question) — "monthly prepaid spend looks cumulative": `sensor.prepaid_spend_this_month` (prepaid_core.yaml) was never a sum of real purchases — it's `prepaid_import_monthly (kWh used this month) × prepaid_true_cost_per_kwh (LIFETIME blended rate)`, an estimate. It showed R238 while real July spend was R790 (two Jul 1 purchases). Root structural cause: `prepaid_total_spent`/`prepaid_total_units`/`prepaid_fixed_cost_paid`/`prepaid_total_discount` are labeled "This Month"/"This Cycle" in the UI but no automation has EVER reset them — genuinely lifetime since inception (confirmed via full history, continuous accumulation since at least 2026-06-15, zero resets).*
*FIX (new real-purchase tracking, prepaid_helpers.yaml + prepaid_core.yaml): added `input_number.prepaid_month_units/_spent/_fixed_paid/_discount` (reset to 0 on the 1st) and `input_number.prepaid_year_units/_spent/_fixed_paid/_discount` (reset to 0 only when the 1st is also January) per user's follow-up request to roll monthly figures into an annual trend for graphing. New automation `prepaid_month_counters_reset` (00:00:10, day==1, nested `if` for the Jan-only year reset). `script.update_prepaid_units` now increments both new counter sets alongside the existing lifetime ones on every purchase. New sensors: `sensor.prepaid_actual_spend_this_month` and `sensor.prepaid_actual_spend_this_year` (= month/year spent + fixed_paid) — the real "what did I actually spend" figures. Existing `sensor.prepaid_spend_this_month` renamed to "Prepaid Spend This Month (Estimate)" in-place (entity_id unchanged, unique_id preserved) with a doc comment clarifying it's an estimate, not real receipts — kept for its mid-month-projection use case. Month/year counters seeded with the two backfilled Jul 1 purchases (227.10 kWh / R790 / R210 fixed) since they predate this feature and bypassed the script. NOTE: annual figure is only accurate from 2026-07-03 onward — full-year reconstruction from Jan 2026 isn't possible (recorder `purge_keep_days: 90`, ~Apr 2026 is the oldest available history); offered to partially backfill from April if wanted, not done this session.*
*DASHBOARD BUG (found from user's screenshot) — `.storage/lovelace.dashboard_operations` "Monthly Prepaid Spend" tile was bound to `input_number.prepaid_total_spent` (lifetime, showing 8617 R) — the exact bug the user suspected. Also found: "Prepaid Usage & Cost" graph's "Energy Spend" + "Fixed Cost (Accum)" traces and "Solar Savings vs Cost" graph's "Spend" trace all bound to the same lifetime entities (explains the ever-climbing staircase lines instead of a monthly pattern); and "Monthly Grid Usage" chart rendering completely empty — its `sensor.prepaid_import_monthly` trace was missing the `statistic: state, period: month` keys that the working "Daily Grid Import" panel directly above it has for its daily equivalent. FIX: backed up the storage file, then via direct JSON edit (user chose this over manual UI edit): tile + both spend traces repointed to `sensor.prepaid_actual_spend_this_month`, Fixed Cost trace repointed to `input_number.prepaid_month_fixed_paid`, Monthly Grid Usage trace given the missing `statistic`/`period` keys. **`.storage` dashboards are loaded into memory once and cached — this fix needs an HA restart (or at minimum a full frontend reload) to actually take effect; until then, avoid opening this dashboard's UI editor, since any autosave from the frontend's still-stale in-memory copy would silently revert this fix.** Not committed to git (`.storage` is gitignored, expected).*
*STILL OPEN, not fixed this session: `sensor.prepaid_rolling_30day_usage` (entity_id `prepaid_monthly_usage_true`, prepaid_core.yaml) is mislabeled — despite its name it is NOT a true sliding 30-day window, it just mirrors the last COMPLETED calendar month's total via the utility meter's `last_period` attribute (frozen until the next month boundary, currently 632.02 kWh = June's total). A real rolling window would need to recompute daily. Flagged to user, no decision made yet on whether/how to fix.*
*DOCS: POWER_CONTRACT.md Entity Reference — Energy & Prepaid section updated (new month/year counters, new sensors, estimate-vs-actual distinction documented).*

*2026-07-03 — Geyser Thursday high-usage adequacy fix (maid-day evening reheat):*
*POWER — User reported geyser cold at 6pm Thursday (2026-07-02) and still not hot by ~20:00 shower. Pulled recorder history: morning run 04:00-08:00 (2.18 kWh, normal), midday run 11:00-15:00 (4.25 kWh) read as "Adequate (4.25 / 3.0 kWh)" against the flat `geyser_adequate_daily_energy_by_midday` threshold, so the 17:00 winter early-start branch (geyser_automations.yaml Branch 3) correctly did NOT fire per its own logic — deferred to the 18:30 fallback, which only had ~90 min of reheat by the 20:00 shower. Root cause: the midday kWh delta is used as a proxy for "tank is warm enough to skip an early evening start," but Thursday is a maid day (presence_trust.yaml maid schedule, weekday: [mon, thu], 10:00-17:45) — extra daytime hot-water use for cleaning draws the tank down more than a normal day, so the same midday kWh delta leaves less actual heat by evening. Confirmed via recorder that `sensor.geyser_daily_status` correctly showed `low_energy` during the 04:07-05:36 morning ramp-up (energy >0.05 kWh but <2.0 kWh minimum, not yet at temp) before transitioning to `reached_temp` — the morning slot ran properly, this was purely an evening-scheduling gap.*
*FIX: new `input_number.geyser_thursday_high_usage_extra_kwh` (power_helpers.yaml, default 1.5 kWh) added to `geyser_adequate_daily_energy_by_midday` on Thursdays only (`now().weekday() == 3`) before the Branch 3 adequacy comparison in geyser_automations.yaml — effective Thursday threshold 4.5 kWh, which would have caught the 4.25 kWh 2026-07-02 incident and forced the 17:00 early start. Applied consistently to Branch 3's conditions, logbook message, and notification message (geyser_automations.yaml), and to the dashboard-facing `midday_adequacy` attribute on `sensor.geyser_daily_status` (power_state.yaml) so the displayed threshold matches what the automation actually decides. User explicitly scoped this to Thursday only (not Monday, the other maid day) and chose the raised-threshold approach over an unconditional bypass.*
*DOCS: POWER_CONTRACT.md Geyser Scheduling section (new helper documented, evening branch thresholds noted).*

*2026-07-03 — RUNG 3 shadowing visitor detection + double-query image URL + camera-name fallback (S17b):*
*SECURITY — Yesterday's S17 fix (visitor-at-gate always critical) was live but a real visitor at 07:27 this morning still got no alert at all, classified silently as `family_movement`. Root cause: RUNG 3 (`(grounds or inside_any) and anyhome and not staff` → family_movement) fires on ANY concurrent grounds/inside motion while anyone's home — routine household activity (cam15 passage, cam12 pond, cam04 carport all fired in the same 05:25-05:29 UTC window) matched RUNG 3 first and "first match wins" meant the classifier never reached RUNG 5 (visitor) even though ipcam01 genuinely had a person in the gate-approach zone (`perim_front and entrance_valid`) at the same moment. Fix: added `and not (perim_front and entrance_valid)` guard to RUNG 3 — ambient household motion no longer shadows a concurrent, specific gate signal. Verified the Hikvision app's 07:27 event timestamp is accurate (HA's own ipcam01_street_driveway_up_motion_valid signal landed at the identical second — no camera/NVR logging lag).*
*SECURITY — Two more notification defects found while investigating: (1) the "Camera:" field in notify_security_event still showed the wrong camera even after yesterday's fix — root cause was that gate/arrival/departure/visitor notifications mostly pass a per-ZONE image (security_grounds_front_latest.jpg), not a per-camera one, so yesterday's filename-slug lookup rarely resolved and fell through to a raw regex on the entity_id that mangled "ipcam03_driveway" into "Ipdriveway" (the `cam[0-9]+_` pattern matches the "cam03_" substring inside "ipcam03_driveway", not just NVR-style prefixes). Added a middle-tier fallback: look up cam_entity's own live friendly_name directly via `state_attr()` before ever reaching the regex. (2) Notification images were rendering as a plain clickable browser link instead of inline in the push — every per-zone image path is stored WITH a "?v=<write-time-ts>" already appended, and the notify script's cache-buster blindly appended a SECOND "?v=<send-time-ts>", producing a malformed double-query URL iOS could no longer parse as an image attachment. Fixed by stripping any existing query string before appending a single fresh one.*
*DOCS: SECURITY_CONTRACT.md Section 6 (BUG-S57/S58/S59), Section 10 changelog updated.*

*2026-07-02 — Visitor-at-gate always critical (RUNG 5 collapse + gate_loitering) + Stage1 departure/arrival ipcam03-entrance corroboration (S17):*
*SECURITY — Two live incidents this morning drove this session: (1) a real visitor at 09:34/09:35 got two WARNING-only pushes ("🔔 Activity at gate — arrival or visitor?" / "⚠️ Activity on front perimeter") instead of critical, because RUNG 5a ("visitor") only fired critical when `allhome` or `not anyhome` — with "some" family home it fell to the neutral RUNG 5b ("gate_activity"), capped to warning severity by day; the 09:37 repeat got no notification at all, suppressed by the 1800s cooldown that kicks in once `staff_on_site` is on (maid_on_site had been manually toggled at 08:00, two hours before the scheduled 10:00 window). (2) At 12:50 Vicky's arrival was pushed as "🚗 Departure — vehicle leaving" — Stage1's exit_valid trigger only checks `ipcam01_street_driveway_up_motion_valid` recency to rule out a false departure-on-arrival, and the street camera never fired for this arrival at all; the real tell was `ipcam03_driveway_entrance_valid` firing 35s *before* the gate even opened, which Stage1 never looked at. Only an unrelated WiFi-AP arrival pipeline caught the true answer 4 minutes later. User directive: false "arrival" noise (can't find your keys) is fine; missing a real visitor is not — so visitor-at-gate should always be critical and fast (<15s), with staff-hours filtering based on movement pattern (loitering) rather than who's home.*
*FIX 1 (severity) — `security_logic.yaml` RUNG 5a/5b collapsed into one rung: `perim_front and entrance_valid and not gate and not arriving` → `visitor` (critical) always, UNLESS `staff` (low_trust_present) is on AND the person is NOT loitering, in which case → `service_person` (silent) — staff walking straight through the gate stays quiet, but anyone lingering (including during staff hours) still fires critical. New `binary_sensor.security_gate_loitering` (cameras_processing.yaml) — `delay_on: 7s` on ipcam01 regionentrance, so only sustained presence (not a quick pass-through) counts as loitering. `security_automations.yaml` visitor/gate_activity router cooldowns simplified from `1800s if staff_on_site else 30s` to a flat 30s (staff filtering now happens upstream in the classifier, so the long cooldown was redundant and could mask a genuine new visitor arriving mid-staff-shift). RUNG 5c (`gate_activity` for gate+grounds ambiguity, unrelated to this fix) still shares the same router branch — its title/message/image corrected from stale perimeter-front wording to grounds-based wording since RUNG 5b (the only other RUNG that used to reach that branch) no longer exists. End-to-end latency for a non-staff visitor: ~4-5s (well under the 15s target); staff-hours loitering case: ~11-12s (7s loiter threshold + 4s snapshot delay).*
*FIX 2 (departure/arrival direction) — `security_gate_vehicle_stage1` exit_valid branch: added `ipcam03_driveway_entrance_valid` recency (state=on OR age<90s) as an OR alongside the existing `ipcam01_recent` check before accepting an exit-zone trigger as a genuine departure. Uses the earliest available on-site evidence (the driveway's own entrance zone, which fired before the gate even opened in the confirmed incident) instead of relying solely on the street camera, which had simply missed the approach both times it happened today.*
*DOCS: SECURITY_CONTRACT.md Section 3 (new gate_loitering entity), Section 10 changelog, Section 11 classification table updated for S17.*

*2026-07-01 — P4 forecast ratio adjustment + geyser midday adequacy display + Stage1 arrival fallback + Pass2 WiFi notification:*
*POWER — P4 grid charge (power_automations.yaml): root cause of 2026-07-01 SOC plateau at 81% vs 90% target found and fixed. Solcast forecast today was 46 kWh (sunny-day forecast), but actual solar was poor. `sensor.solcast_pv_forecast_forecast_remaining_today` reflects the forecasted afternoon kWh regardless of actual performance — on bad-solar days this kept `solar_for_battery` large and `kwh_shortfall` below the 5 kWh `gap_threshold` at every 14:00-15:30 checkpoint, blocking both the normal enable condition AND the rate_urgent check (both gate on kwh_shortfall > gap_threshold). Only the 16:00 deadline_zone fired, giving 1 hour to close a gap too large to fill. Fix: multiply `remaining_forecast` by `sensor.solar_vs_forecast_ratio_today / 100` before the shortfall calculation. By 14:00 the ratio reflects 7+ hours including peak solar window (11:00-14:00) — highly reliable. On bad days (ratio 30%) adjusted remaining = 13.5 × 0.30 = 4 kWh → kwh_shortfall realistic → grid enables at 14:00-15:00 instead of 16:00. Good days (ratio ~100%) unaffected. New variables: `remaining_forecast_raw`, `ratio_today`, `remaining_forecast` (adjusted). Logbook on Enable P4 and Hold-off branches updated to show "raw X kWh × ratio Y% = Z kWh". POWER_CONTRACT.md updated with full incident note.*
*POWER — geyser_daily_status attribute (power_state.yaml): added `midday_adequacy` attribute to `sensor.geyser_daily_status`. Shows "In progress — X.X kWh (need Y.Y)" before 15:00; after 15:00 snapshot shows "Adequate (3.81 / 3.0 kWh) — early start 17:00" or "Low (X.X / 3.0 kWh) — 18:30 fallback". Enables a single dashboard row to answer "was midday enough for a good evening?" Live confirmed: "Adequate (3.81 / 3.0 kWh) — early start 17:00". POWER_CONTRACT.md geyser_daily_status entity reference updated.*
*SECURITY — Stage1 arrival fallback (security_automations.yaml): added `binary_sensor.ipcam01_street_driveway_up_entrance_valid` as a second trigger for Stage1 (`security_arrival_stage_1`). Handles the case where a car remote-opens the gate before approaching the camera zone (no approach phase → gate trigger fires but ipcam01_recent may not be). The entrance_valid trigger requires `gate_recent` (gate opened < 60s ago) AND NOT `exit_recent` (ipcam03 exit_valid NOT recent — departure not in progress). Also extended ipcam01_recent window 120→180s. SECURITY_CONTRACT.md BUG-S38 and Stage1 trigger note updated.*
*PRESENCE — Boundary Resolver Pass2 WiFi fallback notification (presence_boundary.yaml): Pass2 arrival branch now sends an "Arrived (via WiFi)" information notification when Stage2 security pipeline hasn't run within 5 minutes (stage2_age > 300s). Prevents silent arrivals on days when the security camera pipeline is delayed or unavailable. PRESENCE_CONTRACT.md Pass2 flow updated.*

*2026-07-01 — CRITICAL FIX: all security/light/system/water push notifications silently failing since 2026-06-28 PART 5:*
*Root cause: `notify.send_message` schema accepts ONLY `message` and `title` — any extra field (including nested `data: {...}`) returns HTTP 400 Bad Request. All 4 notify scripts (notify_security_events, notify_light_events, notify_system_event, notify_water_events) had nested `data:` blocks containing `image`, `actions`, `push`, `channel`, `ttl`, `priority` — all returning 400 silently (swallowed by `continue_on_error: true`). Confirmed via recorder: 2026-06-30 Stage1/Stage2 pipeline ran correctly (ipcam01/03 signals, correct names in input_text.last_arrival_person), but zero push notifications reached any phone.*
*KEY FINDING: legacy per-device notify services (`notify.mobile_app_iphone16promax_ryan`, `notify.mobile_app_ap_0223_1001`, `notify.mobile_app_honor10_dash`, `notify.mobile_app_honorx7_dash`) are STILL registered and their schema includes a `data` field — the 2026-06-28 PART 5 investigation was wrong to conclude all legacy services were gone. These services accept `data: { push: { interruption-level: critical }, channel: alarm, ttl: 0, priority: high }` (HTTP 200 confirmed via live test).*
*FIX (two-tier): (a) information + warning → `notify.send_message` with `target: entity_id:` list (schema-valid, HTTP 200, image URL embedded as `📷 {{ img }}` text in message body); (b) critical → 4 separate legacy per-device service calls with `data:` field for alarm channel + iOS DND bypass. Applied to notify_security_events.yaml (3 branches), notify_light_events.yaml, notify_system_event.yaml, notify_water_events.yaml. Scripts reloaded; live end-to-end tests passed (HTTP 200 for both tiers). Action buttons (OPEN_GATE, DOGS_INSIDE_ON) cannot be sent via `notify.send_message` schema — removed from scripts; `dogs_inside_from_notification` automation becomes unreachable (manual toggle remains).*
*STILL OPEN (not this session): notify.STD_Alerts used by 16 alert: definitions — see 2026-06-28 PART 7 above.*

*2026-06-29 — Missing 7:05am perimeter-departure alert investigated; maid_start shifted; midnight/midday auto-clears added for 4 manual overrides + entertaining_mode:*
*Investigated user report of zero notification for a ~7:05am departure. Recorder DB (home-assistant_v2.db) showed NO camera (ipcam01/02/03, any NVR cam) recorded any motion/event signal during either gate-open window that morning (06:47–06:47:48 and 07:18–09:45) — not a cooldown/quiet-hours/classifier bug, the cameras genuinely saw nothing. Separately and structurally: ipcam01/ipcam02 (perimeter front) can never alert on departure even when working — Region Exiting was deliberately disabled on both in the 2026-05-20 BUG-S29 fix (approach-direction-only by design). Departure notifications run through `binary_sensor.ipcam03_driveway_exit_valid` (driveway camera, inside the gate) instead — see SECURITY_CONTRACT.md new "Design note (2026-06-29)" under the Street-Down Camera section. First real camera activity all morning was 09:47–09:59 (ipcam01/ipcam03 entrance), consistent with the maid arriving closer to 10am than the scheduled 09:00.*
*FIX (live via Supervisor API, `input_datetime.set_datetime`): `input_datetime.maid_start` 09:00→10:00. `maid_end` unchanged (17:45). Confirmed value held through subsequent `input_datetime.reload`.*
*Also found `input_boolean.staff_on_site_override` had been stuck ON since 2026-06-27 15:11 (BUG-P14) — checked live state, confirmed it had since correctly auto-cleared (the 4h safety net from the 06-28 fix worked).*
*FOLLOW-UP REQUEST (same session): user asked to add an auto-clear for `dogs_inside` (documented gap — "no auto-off" in SECURITY_CONTRACT.md) and asked whether ALL manual override-style booleans should default to clearing at midnight. Surveyed every `*_override`-named helper plus `dogs_inside` across all packages. Added midnight auto-clears for: `dogs_inside` (security_automations.yaml `dogs_inside_midnight_clear`), `boundary_permissive_override` (presence_trust.yaml), `water_refill_force_override` (water_tank_refill_control.yaml). Per user's correction: `geyser_morning_override` instead gets a **midday (12:00)** clear (geyser_automations.yaml `geyser_morning_override_midday_clear`) since "morning" ending at midnight didn't match its purpose. `entertaining_mode` already had a 06:00 daily clear — per user's request this is now secondary: `bar_bedtime_cutoff` (lighting_bar_presence.yaml) clears it directly when it turns off the patio lights (bedtime + nobody at bar = real party-end signal); the fixed-clock clear moved 06:00→01:00 as backstop only.*
*Explicitly excluded from midnight-clear (flagged to and confirmed by user): `inside_cameras_schedule_override` (can legitimately stay on for days while away), the 5 power `simulator_*_override` helpers (active test sessions may span midnight), and `holiday_mode`/`guest_mode` (not "override"-named, often genuinely multi-day).*
*All 6 changed files reloaded live via Supervisor API (`automation.reload`, `input_boolean.reload`, `input_datetime.reload`, `script.reload`) — all returned success. Verified all 5 new/changed automations registered as `state: on` post-reload. `ha core check` passed (config valid, no errors/warnings).*
*DOCS: SECURITY_CONTRACT.md (dogs_inside auto-off closed; new design note on perimeter-front departure limitation), PRESENCE_CONTRACT.md (maid_start value, boundary_permissive_override clear), POWER_CONTRACT.md (geyser_morning_override midday clear), WATER_CONTRACT.md (water_refill_force_override midnight clear), LIGHTING_CONTRACT.md (entertaining_mode clear-on-patio-off + 01:00 backstop).*

*2026-06-28 PART 7 — ⚠️ OPEN, NOT YET FIXED: BUG-N12 remaining piece, notify.STD_Alerts:*
*Status check after PARTS 4-6 below: confirmed fixed — the 19 dispatcher-script call sites (notify_power_event/water/system/light/presence/security_events) and the 4 tablet brightness automations (disabled, see PART 6). Confirmed still broken via `ha core logs`: `"Failed to call notify.STD_Alerts, retrying at next notification interval"`.*
*`notify.STD_Alerts` is used by 16+ `alert:` integration definitions via `notifiers: [STD_Alerts]` across packages/alerts/*.yaml: alerts_batteries, alerts_garden, alerts_network, alerts_doors, alerts_presence, alerts_temperature (×4), alerts_device_power, alerts_security, alerts_water (×3), alerts_system_health, alerts_media, alerts_power. Harder than the dispatcher-script fix: `alert:` calls a bare notify service internally (same dead mechanism as everything else this session), but unlike a script, `alert:`'s `notifiers:` field can't be redirected to `notify.send_message` + entity_id targets — it needs a literal notify-domain service/group entity.*
*Two options, neither attempted yet: (a) HA "Notify Group" UI helper — config-entry/UI-only, not git-trackable, and unverified whether `alert:` can call an entity-based group the same way it called the old legacy group service; (b) migrate all 16 alert definitions to drop `notifiers:` entirely and dispatch via the now-fixed `notify_*_event` scripts on `alert.*` state change instead — more code, but stays fully in YAML/git, consistent with everything else fixed this session. Next session should test option (a) live via the Supervisor API (same method used to confirm the tablet brightness gap in PART 6) before committing to a direction — do not guess/implement blind. See NOTIFICATIONS_CONTRACT.md "Platform Delivery Targets" for full affected-file list and current STD_Alerts config (left defined in configuration.yaml, harmless either way).*

*2026-06-28 PART 6 (BUG-N12 follow-up — CONFIRMED: companion-app commands are gone, not a syntax issue; 4 tablet brightness automations disabled):*
*PART 5 (below) fixed the notification-delivery half of BUG-N12 (plain title/message via notify.send_message) but the tablets.yaml conversion ALSO used a nested `data: {message, data: {command}}` structure for the screen-brightness companion-app command — copied from the old (now-dead) legacy per-device service pattern. User reported the morning-restore automation "seems to not deactivate" the night dim. Investigated properly this time using the live HA REST API directly (SUPERVISOR_TOKEN → `http://supervisor/core/api/`) instead of guessing from outside: (1) confirmed `automation.tablets_restore_screen_brightness_after_night` DOES fire on schedule (last_triggered 04:32 today) — not a trigger/condition bug; (2) called `notify.send_message` against `notify.honor_10_dash_mobile_app` directly via the API with several payload variations — plain `message`+`title` returns 200 OK, but ANY extra field (`data: {...}` nested, `data: {command: 20}`, or a flat `command` key) returns 400 with `extra keys not allowed @ data['data']` / `@ data['command']` — confirmed identically by the user via Developer Tools → Actions; (3) checked `GET /api/services` directly — `notify.send_message`'s registered schema has exactly two fields, `message` and `title`, no `data` field at all; (4) checked all 40+ entities on both tablet devices — every one is a read-only `sensor.`/`binary_sensor.`, no `number.`/`select.`/`button.` entity exists for brightness. Conclusion: the mobile_app entity migration (2026-05-12) removed the companion-app "send a command" channel from the public API entirely, with no replacement controllable entity exposed yet — this is an upstream HA/companion-app capability gap, not fixable from config. Affects all 4 tablet brightness automations identically (night dim, morning restore, away dim, return restore) — none have been able to actually change physical screen brightness since 2026-05-12, the user only noticed the "stuck dim" direction.*
*FIX: disabled all 4 automations (`tablets_brightness_night_dim`, `tablets_brightness_morning_restore`, `tablets_brightness_away_dim`, `tablets_brightness_return_restore` — `enabled: false` in tablets.yaml) per user's choice, rather than leaving them erroring or silently no-op'ing. Re-enable if/when mobile_app exposes a controllable brightness entity or restores command support.*

*2026-06-28 PART 5 (BUG-N12 — CORRECTED FIX: notify.STD_* groups + tablets.yaml. PART 4's "fix" below was wrong — mobile_app/telegram are entity-only, no bare service exists at all):*
*PART 4 (same day, below) renamed the STD_* group members from legacy service names to current entity slugs (e.g. `ryan_iphone16_mobile_app`) and reported the bug fixed. It wasn't — the user got the exact same "unknown action: notify.std_warning" repair item again, plus a NEW one from `automation.tablets_dim_screens_at_night` ("unknown action: notify.honor_10_dash_mobile_app"), proving the PART 4 tablets.yaml fix was equally wrong. Checked `ha core logs` directly this time instead of assuming: confirmed `homeassistant.exceptions.ServiceNotFound` for EVERY group member, including `notify.telegram_bot_5527` — which had nothing to do with the mobile_app migration and was previously assumed safe. Root cause, corrected: mobile_app AND the Telegram integration are both fully entity-only now (no bare `notify.<x>` service registered for ANY of them) — the legacy `notify: platform: group` mechanism (which calls members by bare service name internally) cannot work with ANY current target, regardless of what name is configured. Renaming the member list was never going to fix it.*
*REAL FIX: converted all 19 `notify.STD_Information`/`STD_Warning`/`STD_Critical` call sites (across `notify_power_event.yaml`, `notify_water_events.yaml`, `notify_system_event.yaml`, `notify_light_events.yaml`, `notify_presence_events.yaml`, `notify_security_events.yaml`) to call `notify.send_message` directly with an explicit `target.entity_id` list of the 4 device entities — the only mechanism that still works post-migration (mirrors the Telegram call pattern already used elsewhere in the same files). Removed the now-unused `STD_Information`/`STD_Warning`/`STD_Critical` group definitions from configuration.yaml. `packages/admin/tablets.yaml`'s 8 call sites converted the same way (action: notify.send_message + target.entity_id, replacing the direct `action: notify.honor_10/x7_dash_mobile_app` calls).*
*NOT FIXED, FLAGGED FOR LATER: `notify.STD_Alerts` is used by 16+ `alert:` integration definitions (`packages/alerts/*.yaml`, `notifiers: [STD_Alerts]`) — `alert:` calls bare notify services internally too, the exact mechanism just proven dead for every target, and `alert:`'s `notifiers:` field can't be redirected to `notify.send_message` + entity_id the way a script can. Left `STD_Alerts` group defined in configuration.yaml (harmless, already broken either way) — needs either an HA "Notify Group" UI helper (unverified whether `alert:` can call an entity-based group — config-entry/UI-only, not git-trackable) or migrating those 16 alert definitions to dispatch via the now-fixed `notify_*_event` scripts instead of `alert:`'s built-in notifiers. This is a separate, larger follow-up — not attempted this session.*
*`ha core check` passed after all fixes. DOCS: NOTIFICATIONS_CONTRACT.md Section 7 rewritten — corrected platform list, two-pass bug note, working call pattern, explicit STD_Alerts gap with full affected-file list.*

*2026-06-28 PART 4 (SUPERSEDED BY PART 5 ABOVE — fix described here was incomplete/wrong, kept for the record):*
*Root cause as understood at the time: configuration.yaml's four `notify: platform: group` definitions (STD_Information, STD_Warning, STD_Critical, STD_Alerts) referenced legacy mobile_app service names that stopped existing when mobile_app migrated to per-device notify entities (2026-05-12). Attempted fix: renamed group members to current entity slugs. This did NOT actually fix the bug — see PART 5.*

*2026-06-28 PART 3 (BUG-S52/BUG-P14/BUG-L15 — alert run-through requested by user: classifier mislabel, stuck staff override, evening arrival lighting):*
**⚠️ ACTION REQUIRED BY USER: `input_boolean.staff_on_site_override` was found stuck ON since 2026-06-27 15:11:52 (24+ hours) — manually toggle it OFF now. The new auto-clear automation only prevents recurrence, it doesn't retroactively clear the current session.**
*User asked for a run-through of recent alerts after flagging: alerts feel delayed, recurring carport-side false positives, staff-on-site not suppressing alerts on Saturday 06-27, and evening-arrival lighting (entrance/dining/patio) not firing. Investigation + fixes:*
*1) NOTIFICATION TITLE MISLABEL (BUG-S52): two screenshots showed "⚠️ Perimeter activity" titled notifications whose body said `zones: Grounds` and whose image was cam04 (carport) or ipcam03 (driveway, car plainly at an OPEN gate). Root cause: `sensor.security_event_classification`'s 8-rung ladder (security_logic.yaml) has a bottom catch-all (`else: perimeter_threat`) for anything that doesn't match a specific rung, and the router hardcodes the title "Perimeter activity" for that classification regardless of actual zone. Two gaps fed the catch-all: RUNG 7 (intruder) requires `ip_cam or high_conf` — a lone cam04 (NVR, no AI) trigger never clears that; RUNG 7 also requires gate CLOSED — a car sitting at an open gate with `family_arriving` not yet AP-confirmed matched no rung at all. Fix: added RUNG 5c (`gate_activity` — gate open + grounds camera fired, not yet arrival/departure-confirmed, reuses existing gentle "arrival or visitor?" notice) and RUNG 7b (`grounds_low_confidence` — same as RUNG 7 minus the ip_cam/high_conf bar, new router branch titled "⚠️ Activity in grounds") in security_logic.yaml + security_automations.yaml.*
*2) WRONG CAMERA NAME IN NOTIFICATIONS: same screenshots showed "Camera: Security Event Router" in the footer instead of the actual camera. Root cause: notify_security_events.yaml's `cam_entity` read `{{ source }}`, and every router branch passes `source: "security_event_router"` (the automation's own name) — the correct line (`states('input_text.security_last_motion_camera')`) was present but commented out one line above. Fixed: restored it.*
*3) STAFF_ON_SITE NOT SUPPRESSING ALERTS SATURDAY 06-27 (BUG-P14): pulled recorder history directly from home-assistant_v2.db. Confirmed the gardener schedule fired correctly (gardener_on_site ON at 08:00, staff_on_site correctly ON at 08:00) — NOT a schedule bug. At 15:11:52 `input_boolean.staff_on_site_override` was switched ON (manual dashboard toggle; no automation anywhere writes to this entity) and has remained ON ever since (confirmed still ON as of this session, 24+ hours later) — it unconditionally forces `staff_on_site`/`low_trust_present` to false, which is exactly why subsequent alerts that evening (and since) weren't trust-suppressed. Fix: added two safety-net automations to presence_trust.yaml — warn at 2h, auto-clear + notify at 4h. Does not retroactively clear the current stuck state — user must toggle it off manually.*
*4) EVENING ARRIVAL LIGHTING (BUG-L15): confirmed `automation.lighting_arrival_night` Scenario 2 ("someone home" — the normal gate→door arrival case) never included `switch.dining_room_light` in its turn-on list (only the "nobody home" scenario did), and its 5-minute patio/front-security auto-off only checked `bar_occupied`, with no check for whether bedtime had actually started — so patio switched off again ~5 min after any arrival regardless of how early in the evening. Fix: added dining_room_light to Scenario 2's turn-on list; added `input_boolean.bedtime_mode == 'on'` as a second required condition for the auto-off (alongside bar_occupied == off) — patio now stays on if arrival happens before bedtime.*
*5) ALERT DELAY / CARPORT FALSE POSITIVES (diagnosed, not code-fixed this session): worst-case notification latency from raw motion to HA dispatch is ~30-35s, dominated by the 30s `delay_off` debounce on motion_valid sensors (cameras_processing.yaml) — by design, not a bug. Carport (cam04) false positives are a known, already-documented hardware gap (raw NVR pixel-diff motion, no AI filtering) pending an IP camera replacement — see SECURITY_CONTRACT.md AcuSense roadmap. RUNG 7b above reduces the blast radius (correct title/cooldown) but does not eliminate cam04's underlying false-trigger rate.*
*6) DEBOUNCE REDUCTION (user follow-up, same session): asked to cut the alert-delay debounce but only on the new IP cameras since they're far less prone to false positives than the old NVR cameras. `cameras_processing.yaml`: cut `delay_off` 30s/45s → 5s on the 5 AcuSense `*_motion_valid` sensors (ipcam01/02/03/04/05) only — left the NVR cameras (cam04/05/07/09/12/14/15) and the ipcam entrance_valid/exit_valid sensors (different mechanism, arrival/departure direction logic) untouched.*
*DOCS: SECURITY_CONTRACT.md (BUG-S52, classifier output states table, debounce reduction note), PRESENCE_CONTRACT.md (BUG-P14, Section 11), LIGHTING_CONTRACT.md (BUG-L15) updated.*

*2026-06-28 PART 2 (P4 fix confirmed live + first clean 90% hit; Battery-First-during-midday idea analyzed and rejected):*
*RELOAD CONFIRMED: user reloaded Automations + Helpers in HA after the Part 1 fix below. Verified live via recorder: `input_number.p4_max_grid_charge_rate_kw` present with value 5.0 and `binary_sensor.load_shedding_active` reporting "off", both with fresh timestamps — config picked up without restart.*
*FIRST CLEAN TARGET HIT SINCE THE FIX: 06-28 logbook — 14:00 disabled (solar still 5416W, shortfall negative/on-track), held through 15:30 (shortfall still negative-to-flat, no urgency), deadline push fired correctly at 16:00 (SOC 78%, 12% gap, 60 min left, solar down to 344W) → SOC closed to 83% by 16:30, **peaked at 90% at 16:57** — the first day since the original 06-23/24 incident that the orchestrator_target_soc_by_sunset was actually reached. rate_urgent did not need to fire this run (deadline_zone alone was sufficient — shortfall hadn't grown enough earlier in the day to trip the rate check), which is expected on a better-solar day; it remains the backstop for days like 06-26 where the gap balloons before 16:00.*
*BATTERY-FIRST-DURING-MIDDAY — ANALYZED, NOT IMPLEMENTED (user request: "optimise total grid import/day — compare shipped fixes vs this idea"): user asked whether shifting `select.inverter_1_energy_pattern` from "Load First" to "Battery First" during the 10:30-15:00 best-solar window (let grid cover the lighter daytime load, divert all solar to battery) would reduce daily grid usage. Checked solar export/curtailment on `sensor.inverter_grid_power` across 09:00-16:00 for 06-25/26/27: essentially zero export (0-0.1% of samples below -50W, max -216W) — meaning ALL midday solar surplus already reaches the battery today; there is no wasted/curtailed solar for Battery First to recover. Conclusion: the swap would be a straight 1:1 reallocation (every kWh moved from load→battery must be replaced by an equal kWh of grid→load) plus battery round-trip conversion loss (~10-15%) on energy that currently goes straight from solar to load for free — a net INCREASE in total daily grid import (estimated +6-9 kWh/day at observed averages), not a decrease. No time-of-use tariff exists in this config (`input_number.grid_tariff_nominal` is a single flat rate), so there's no cost-timing offset to justify it either. By contrast, the rate-urgency/deadline-push fixes (Part 1, same day) are grid-neutral — they change *when* within the existing 14:00-17:00 window grid kicks in, not how much total grid is needed to close a given shortfall (physics: kWh needed = soc_gap × battery_capacity_kwh regardless of timing). User confirmed: keep the current path, do not implement Battery-First-during-midday.*

*2026-06-28 PART 1 (P4 grid-charge rate-urgency fix; load_shedding dead-entity bug fixed; P4 17:00 trigger bug fixed):*
*P4 STILL PLATEAUING AFTER 06-24 FIX: user asked to confirm SOC effects over the last few days. Pulled recorder history (sqlite, home-assistant_v2.db) directly for 06-24 through 06-27 — all four days peaked short of the 90% orchestrator_target_soc_by_sunset target (78%, 83%, 76%, 85%). 06-25 was informative: the 06-24 deadline-push fix worked as designed — P4 went to Grid at 15:30 and `inverter_grid_power` stayed at 5.5-7.6kW continuously through 17:00 (confirmed the inverter's own P4 program window hard-cuts grid import at 17:00 regardless of HA select state) — yet SOC only closed 66%→83%. So the toggle logic was fine there; the gap had simply grown too large to close with the time/rate available. 06-26/06-27 logbook entries (queried from the `events`/`event_data` tables, event_type `logbook_entry`) showed the actual mechanism: `solar_tapered` (input_number.p4_grid_charge_solar_gate_w, 1500W) reacts to instantaneous solar watts, not the accumulating kWh shortfall — on 06-26, solar sat above the 1500W gate from 14:00 to 15:30 while `kwh_shortfall` grew 4.1→8.6 kWh ("P4 hold — shortfall X kWh > threshold but solar still YW"), leaving only the fixed 16:00 deadline_zone to close a hole that had already outgrown what the inverter can physically charge in time.*
*FIX (power_automations.yaml, automation.inverter_p4_grid_charge_control): added a `rate_urgent` check — each evaluate now computes `required_rate_kw = kwh_shortfall / hours_to_deadline` against new `input_number.p4_max_grid_charge_rate_kw` (5.0 kW default, grounded in the observed 06-25 net SOC-closing rate: +8.2 kWh of the 48.2kWh bank over 1.5h of continuous grid ≈ 5.5kW net, well below the raw 6-7kW grid import since load shares the same import). If closing the shortfall by the deadline would need a sustained rate above that, the existing "Deadline push" branch now also fires on `rate_urgent`, regardless of `solar_tapered` or the fixed 16:00 hour — same forcing action, triggered as soon as the math says waiting will make the target unreachable instead of waiting for a fixed clock hour. The "Disable — solar on track" branch's guard was extended to also block on `rate_urgent` so it can't undo this.*
*LOAD SHEDDING PLUMBED IN (per user's request to "work around" sheds inside the P4 window): the effective deadline used in the rate_urgent calc is no longer always 17:00 — if `binary_sensor.load_shedding_active` is on, the deadline collapses to "now" (grid already unavailable, no benefit waiting); if the area sensor's `forecast` attribute shows a shed starting before 17:00, the deadline pulls forward to that start time (grid won't be chargeable during the outage either). Both naturally drive `required_rate_kw` up and trip `rate_urgent` early — this is how the P4 automation now charges ahead of a scheduled outage instead of waiting for a deadline that assumed grid would still be there.*
*BUG FOUND + FIXED while wiring the above: `load_shedding_templates.yaml`'s three derived sensors (`load_shedding_status_card`, `binary_sensor.load_shedding_active`, `load_shedding_minutes_remaining`) were all still reading `sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark` — an entity with state `None` and no attributes (stale area-code/integration version). The live entity is `sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c`, which POWER_CONTRACT.md's "Entity Name Corrections" table already documented as the rename target and which `load_shedding_automations.yaml` was already using correctly — only the templates file never got updated. `binary_sensor.load_shedding_active` happened to still read "off" by coincidence (no shedding was scheduled while broken), which is why it went unnoticed. All 3 templates repointed to the live entity.*
*BUG FOUND + FIXED (same automation): Branch 2 ("17:00 restore — unconditionally Disabled") referenced `trigger.id == 'restore'`, but no trigger in the automation ever emitted that id — dead code since the automation's creation. Added the missing `time: "17:00:00", id: restore` trigger. Had no charging-behaviour impact in practice — confirmed via recorder that the inverter's own P4 program window (14:00-17:00 hardware schedule) already hard-cuts grid import at 17:00 regardless of what the HA select shows (`inverter_grid_power` drops to near-zero at 17:05 on 06-25 even though the select stayed "Grid") — but the select entity was misleadingly stuck on "Grid" all evening/overnight until the next day's 14:00 evaluate, confirmed via recorder state history (06-25 15:30 → 06-26 12:23, single unbroken "Grid" state).*
*DOCS: POWER_CONTRACT.md updated — new `p4_max_grid_charge_rate_kw` helper documented, new "Rate-urgency + load shedding" subsection under E5-3, 17:00 trigger bug noted, load_shedding dead-entity bug closed in both the "Load Shedding" entity section and the "Entity Name Corrections" table. `ha core check` passed after all edits. Remaining manual step: reload Automations + Helpers (input_number) in HA Developer Tools → YAML to activate (not done from this session — no API/UI access available here).*

*2026-06-24 (battery_runtime_status_card bug fix; P4 deadline-push fix for SOC plateau):*
*BUG FIX: sensor.battery_runtime_status_card (battery_runtime.yaml) — "Battery charging" branch read `{{ charge_eta }}%` where charge_eta = states('sensor.markdown_ss_battery_charge_time_left'), an entity that doesn't exist (always "unknown"). Dashboard showed "Battery charging – target unknown%". The correct value, `target_soc` (= sensor.ss_battery_capacity, the program-aware target SOC), was already computed one line above and used correctly in the other branch — just never wired into this one. Fixed: now reads {{ target_soc }}%. Removed the now-unused charge_eta variable (only consumer was this bug).*
*P4 GRID-CHARGE: user reported SOC plateauing ~78-80% both 2026-06-23 and 2026-06-24, never reaching the 90% orchestrator_target_soc_by_sunset, despite yesterday's solar-aware delay fix. Root cause: on 2026-06-24, P4 enabled Grid at 16:00 (shortfall 6.5kWh > threshold, solar tapered to 799W), SOC jumped 72%→78% in that one 30-min slot — then at the 16:30 checkpoint the shortfall recalculation (using the SAME live SOC) came out to 4.6kWh, just under the 5kWh threshold, so the "Disable — solar on track" branch fired and cancelled grid for good. No evaluate trigger exists between 16:30 and the unconditional 17:00 restore, so there was no chance to re-enable even though 12% of gap remained with only 30 min left — not realistically closable, but the bigger issue is the system trusted a single noisy 30-min projection at the worst possible time (right after grid charging itself had just inflated the SOC reading). Fix (power_automations.yaml, automation.inverter_p4_grid_charge_control): added a "deadline push" branch — from input_number.p4_deadline_zone_start_hour (16:00 default) onwards, abandons the solar-aware/shortfall projection logic entirely and forces/keeps Grid on if soc_gap > input_number.p4_deadline_gap_threshold (3% default), regardless of solar_tapered or the shortfall calc. The existing "Disable — solar on track" branch was also guarded so it can't undo the deadline push while the gap still exceeds that threshold. Real-time target-reached disable (added 2026-06-23) still takes priority and clears it immediately once SOC actually reaches target.*
*DASHBOARD AUDIT (appliance-control, power-control, inverter-control views): power-control's compact "Geyser Controls" card is missing sensor.geyser_target_run_hours_today (target hrs for comparison — pool's equivalent card has this), sensor.geyser_control_status (13-state reason text incl. active-window context — currently only the appliance-control view's fuller card has it), and the morning_kwh/midday_kwh/evening_kwh attributes already computed on sensor.geyser_daily_status (never surfaced anywhere — would need "type: attribute" entity rows, same pattern as the borehole card added 2026-06-22). power-control's "Orchestrator State & Thresholds" card is missing the three new P4 helpers from 2026-06-23/24 (p4_grid_charge_solar_gate_w, p4_deadline_zone_start_hour, p4_deadline_gap_threshold) and battery_capacity_kwh. Not fixed directly — dashboard .storage JSON requires HA stopped to hand-edit safely (per 2026-06-14 session precedent); gave user copy-paste entity lists instead for the dashboard UI editor.*
*GEYSER MIDDAY-ADEQUATE THRESHOLD RAISED (power_helpers.yaml, input_number.geyser_adequate_daily_energy_by_midday): user asked whether the midday→evening adaptive split (17:00/17:30 early start vs 18:30 fallback) was missing anything — confirmed it already implements the front-load-solar-first design being described; the real lever was the 1.25 kWh "adequate" threshold itself, which dated from the 2026-06-18 bug fix and was never grounded in actual heat-up cost. Pulled input_number.geyser_last_heat_up_minutes history (5 days, excl. zero/reset entries): range 42-257 min, median 136 min, mean 147 min → at 1.25kW that's 2.8-3.1 kWh for a typical full reheat, more than double the old threshold. Raised to 3.0 kWh — biases the system toward requiring midday (the cheap/solar window) to do nearly all of that reheat work before agreeing it's "adequate" and deferring to the likely grid-funded 18:30 fallback. Also discussed: 200L geyser tank vs 3-person morning/evening shower blocks (~60-70L hot water per winter shower) is at or past capacity with zero recovery time between showers (heat pump recovery 2-3h per the same duration data) — a real physical blind spot since there's no tank temp/level sensor; binary_sensor.geyser_at_temperature only reports point-in-time full-tank state and can't see stratification after back-to-back showers. Not fixable in software without added hardware (temp probe) — flagged as a hardware/demand-side gap, not a scheduling one. POWER_CONTRACT.md updated (3 references to old 1.25 kWh threshold corrected to 3.0 kWh).*

*2026-06-23 (CRITICAL BUG FIX — power_statistics.yaml duplicate `template:` key silently dropped 6 sensors; P4 grid-charge made solar-aware):*
*Root cause: power_statistics.yaml had TWO top-level `template:` keys. YAML keeps only the last mapping value for a duplicate key, so the entire first `template:` block was silently dropped from the loaded config — not a startup/restart delay as first suspected when the user reported "Solar Performance Stats" dashboard card showing Unavailable for 7-Day Accuracy %, 7d/30d Mean Production, 7d Std Dev. Casualties: sensor.inverter_production_7d_mean, inverter_production_30d_mean (in the dropped template: block), AND four `platform: statistics` sensors (inverter_production_7d_stdev, house_load_24h_mean, house_load_7d_mean, solar_forecast_accuracy_7d) that were ALSO incorrectly nested inside that same dropped template: list using the wrong schema (platform: belongs under the legacy sensor: platform-list, not template:). Worse: sensor.solar_weather_correlation's 7-day trend inputs (ratio_7d, stdev, cv) all silently defaulted to neutral values (100/0/0) — it could only ever reach 'degraded' via the same-day ratio_today branch, never via sustained 7-day under-performance, for as long as this bug existed. house_load_24h_mean being unavailable also fed directly into the P4 grid-charge shortfall calc (see below) — this is what made that calc treat house load as zero. FIXED: restructured into ONE sensor: list (6 platform:statistics sensors) and ONE template: list (5 derived template sensors) — no behavioural change, just makes the config load as originally intended. Unknown how long this had been broken — file's own changelog comments suggest the structural split happened across the 2026-06-14 P6 session and 2026-06-18 daily-snapshot edit.*
*P4 grid-charge tuning (power_automations.yaml, automation.inverter_p4_grid_charge_control): user observed grid actively charging the battery (6.8kW import) while solar was also producing 4kW, and wanted grid use delayed until solar is actually exhausted, plus immediate release once target SOC is reached (was waiting till 17:00 or the next 30-min checkpoint). Added: (1) real-time numeric_state trigger on sensor.inverter_battery_soc crossing orchestrator_target_soc_by_sunset → disables P4 Grid immediately (Branch 0, highest priority) instead of waiting; (2) new input_number.p4_grid_charge_solar_gate_w (default 1500W) — Grid only enables if current sensor.inverter_pv_power has dropped below this gate, even when the forecast-shortfall math says it's needed (new "Hold off" branch logs when shortfall is projected but solar is still strong); (3) more frequent evaluate triggers — added 15:30/16:00/16:30 (was only 14:00/14:30/15:00) so the system keeps re-checking as solar tapers through the afternoon instead of locking in a decision at 15:00 for the rest of the window. Trade-off flagged to user: this favours lower grid cost over a guaranteed 90% SOC by sunset — if solar stays strong then drops sharply right at the end, there may not be enough time/power left to fully close the gap via grid.*
*P4 follow-up same day: user flagged that the gate shouldn't be sticky — 14:00-15:00 should still have good sun, so a brief dip shouldn't lock grid on for the rest of the window. Fixed: "Disable P4" branch condition now also fires `OR NOT solar_tapered` (previously only checked soc_gap/shortfall) — so if Grid was enabled at one 30-min checkpoint but solar recovers above the gate by the next, it releases back to solar immediately and re-checks again 30 min later. Fully re-evaluates every slot instead of committing once.*

*2026-06-20 (Security trigger-camera priority — BUG-S41 actually fixed):*
*Incident: repeated notifications showed the carport (cam04) or kitchen (cam07) NVR image for "Perimeter activity"/grounds events instead of the relevant IP camera. SECURITY_CONTRACT.md had marked BUG-S41 "FIXED 2026-05-23" but the live code was never actually changed — confirmed by reading security_logic.yaml. Root cause: `sensor.security_trigger_camera` selected the camera by recency (`sort(attribute='last_changed', reverse=True)` over all active cameras), not by the priority order its own comment claimed — so a flickering NVR camera firing a beat after the real IP camera would win and overwrite the shared `security_image_grounds_front`/`_rear` zone-image slots. Fixed in security_logic.yaml: rewrote as a true first-match priority list (`cams | select('is_state','on') | list`, take first). All 5 IP cameras (ipcam01-05) now rank above their NVR zone counterpart (ipcam03 > cam04/cam07; ipcam04/ipcam05 > cam12/cam09); NVR only selected when no IP camera in that zone is active. Inside cameras with no IP equivalent (cam14 lounge, cam15 passage, cam05 garage) unaffected — kept original top/bottom priority. No entity renames; `sensor.security_trigger_camera` output format unchanged. See SECURITY_CONTRACT.md BUG-S41 for full detail.*

*2026-06-21 (Geyser morning extension + earlier midday trigger + borehole conserve gate):*
*Incident: cold (11°C), heavily overcast winter day (sensor.solar_weather_correlation=degraded, Solcast forecast_today 6.9 kWh vs ~30-50 kWh normal), whole family home all day (binary_sensor.all_family_home) — morning run alone wasn't enough, tank cold again by 11am showers. Added Branch 1/1b to automation.geyser_turn_off (geyser_automations.yaml): at the normal morning hard-off, if winter AND ambient temp < geyser_cold_ambient_threshold_c (14°C default) AND solar_weather_correlation in [poor,degraded] AND all_family_home — skip the turn-off, flag input_boolean.geyser_morning_extended_today, and turn off geyser_morning_extend_minutes (60 min default) later instead via new triggers 08:30/09:00/10:00. New helpers: input_boolean.geyser_morning_extend_enabled (master toggle), input_number.geyser_morning_extend_minutes, input_number.geyser_cold_ambient_threshold_c, input_boolean.geyser_morning_extended_today (internal, reset at 00:01). Also added an 11:00 midday trigger (opportunistic, reuses existing solar-gated Branch 2 — Solcast peak can land ~10-11am on bad days).*
*Borehole/water: tightened binary_sensor.water_refill_allowed (water_templates.yaml) to also block when sensor.energy_orchestrator_state == 'conserve', unless sensor.water_tank_level <= 20 (tank-critical override preserved). Previously only critical/loadshedding/loadshedding_critical blocked refill — conserve (which can fire from degraded solar + SOC<60%, or SOC<50%) did not. Driven by the same 2026-06-21 incident — borehole pump adds ~1kW+ battery draw on top of an already bad-solar, high-household-load day. Added new blocked-reason priority 3b ("Conserving battery...") to water_refill_blocked_reason, new conserve_blocked attribute on water_refill_allowed, and a matching "Blocked — conserving battery" priority in sensor.borehole_control_status (water_state_extensions.yaml). See WATER_CONTRACT.md "Conserve gate" for full detail.*

*2026-06-20 (Geyser midday forced-minimum):*
*Incident: poor-solar winter day left geyser cold, required a manual run at ~15:30 (the existing midday window is purely solar-gated and never fired). Added Branch 2b to automation.geyser_turn_on (geyser_automations.yaml) — forces the geyser on regardless of solar surplus if, by the time needed to guarantee a minimum runtime before the 15:00 hard-off, the tank is still cold and the switch is still off. New triggers 13:30/14:00/14:30. Minutes required: 60 winter normal / 90 winter on staff-on-site days or weekends (binary_sensor.staff_on_site, never the manual input_boolean) / 30 non-winter. New helpers in power_helpers.yaml: input_number.geyser_midday_forced_minutes_winter (60), _winter_extended (90), _summer (30) — all UI-adjustable. Still hard-blocked only at loadshedding_critical, same as morning/evening (hot water essential). Notifies severity: warning when the bypass fires. See POWER_CONTRACT.md "Midday forced minimum" for full detail.*
*BUG FIX — same-day, found while diagnosing why no midday alert fired on 2026-06-20: geyser_period_energy_snapshot's midday alert and geyser_daily_minimum_check's 20:00 "all good" branch both gated on input_boolean.geyser_reached_temp_today — a flag that resets only at midnight, so a geyser that reached temp once overnight/early-morning stayed flagged "reached_temp_today=on" all day even after the tank cooled from daytime use, silently suppressing both checks. Fixed: both now use binary_sensor.geyser_at_temperature (real-time) for the decision; the sticky flag is retained only for logging context. Midday alert also escalated information→critical and now auto-triggers a 60-min forced run via the existing manual-run mechanism (input_select.geyser_manual_run_duration="60" + input_boolean.geyser_manual_run_active) as a backstop in case Branch 2b above doesn't run the geyser.*

*2026-06-19 (Stale doc sweep + code fixes):*
*CONTRACT UPDATES: ALERTS — BUG-A03 closed (alerts_temperature.yaml uses script.notify_system_event, all 4 routing automations verified). POWER — Issues 10/11/12/13 closed (Solcast entity correct; automations.yaml empty; unique_id typo fixed 2026-04-21; grid_risk uses inverter_load_power); Issue 8 corrected (group renamed house_security_power_sensors, defined in power_templates.yaml); watchman table and legacy automations table updated. INFRA — BUG-BKP01 closed (github.yaml uses notify_system_event); BUG-INF01 closed (printer_cartridge_low trailing }} removed). NETWORK — BUG-NET01/02/03 closed (CPU/memory availability and packet loss formula corrected). SECURITY — BUG-S29 closed (camera zones repositioned by user 2026-05-20; HA workaround S9 already in place).*
*CODE FIXES: binary_sensor.water_refill_allowed now includes input_boolean.water_tank_refill_enabled check (was returning true even with master switch off — Water Issue 7). solar_helpers.yaml: added missing low_solar_forecast_trigger, high_solar_forecast_trigger, inverter_solar_mode_helper (Power Issue 9 — gated by use_legacy_solar_scenes=off; clears watchman noise).*
*WATER_CONTRACT Issue 7 closed. POWER_CONTRACT Issue 9 closed.*

*2026-06-19: Stale P1/P2 bug audit — all previously-listed broken sensors verified ALREADY FIXED in code, docs were stale. battery_night_survival: both sensor.battery_energy_available (SOC×capacity formula) and sensor.average_night_consumption (platform:statistics, sampling_size removed) are implemented and reference real entities. grid_charging_while_solar: references inverter_pv_power + grid_to_battery_power — both exist. security_correlation: unique_id present at security_logic.yaml:414. average_night_consumption sampling_size:20 bug: already removed per in-code comment. prepaid_buy_decision automation: already checks ['buy_now','buy_soon'] not 'hold'. Entertainment mode M1: fully wired (button_on, scene_on/off, daily_clear at 06:00). M2 energy_saving_mode auto-trigger: IMPLEMENTED — automation.energy_saving_mode_auto_enable (SOC < threshold OR orchestrator critical/loadshedding) + automation.energy_saving_mode_auto_disable (both SOC > recovery AND orchestrator normal/surplus required before clearing). Added to power_automations.yaml.*

*2026-06-18 (Power/Geyser/Water/Dashboard session):*
*GEYSER BUG FIX — Evening early start never fired: `geyser_energy_at_midday_end` stores cumulative daily total (includes morning run); condition was comparing total (3+ kWh) against 1.5 kWh threshold → always false → 17:30 never fired. Fixed: condition now computes delta = `midday_end − morning_end` (midday-window energy only). Threshold lowered 1.5→1.25 kWh (~1h runtime). Season-aware timing added: winter triggers `evening_early_winter` at 17:00; non-winter keeps `evening_early` at 17:30. Helper renamed: `geyser_adequate_daily_energy_by_midday` → "Geyser Adequate Midday-Window Energy".*
*FORCE CHARGE: script.force_charge_batteries — saves all P1–P6 charging/SOC settings, enables Grid on all programs, pauses inverter_programme_auto. automation.force_charge_monitor auto-restores at target SOC. script.force_charge_restore restores with 2s inter-write delays and correct order (sync INV1→INV2 before re-enabling programme auto). script.force_charge_cancel dashboard button. New helpers: input_boolean.force_charge_active, input_number.force_charge_target_soc, input_text.force_charge_saved_charging.*
*POWER STATISTICS FIX: sensor.inverter_production_7d_mean / 30d_mean rewritten as template sensors. Previously were platform:statistics on inverter_today_production with sampling_size:7 (= mean of last 3.5 minutes of readings, not 7-day daily mean). Now: intermediate statistics sensors compute `change` (7d/30d total) on sensor.inverter_total_production (lifetime, never resets); template divides by days for true daily mean. Both _mean sensors removed from recorder exclude list (now recorded for chart history). Intermediate _total sensors excluded from recorder.*
*MONTHLY PRODUCTION CHART: statistic:sum on _today sensors gives cumulative lifetime total, not monthly delta. Fix: switch to monthly utility meters with statistic:state (per HA rules for total_increasing sensors, only state/sum are valid — change not supported). Added battery_discharge_monthly + battery_charge_monthly utility meters to energy_helpers.yaml.*
*BOREHOLE STATUS: sensor.borehole_control_status updated — "Waiting — outside solar window" now shows solar surplus (Xw) as context.*
*DASHBOARD: multiple plotly-graph and apexcharts-card fixes applied directly in .storage. See session notes for detail.*
*GEYSER RUN ALERTS (added to geyser_period_energy_snapshot): morning alert if < 0.5 kWh produced AND not at temperature (severity: warning — tells user tank cold, early start firing); midday alert if midday-window delta < adequate_threshold AND not at temperature (severity: information — confirms which time early start will use). Both alerts fire at 08:00/15:00 snapshot points.*
*LEGACY AUTOMATIONS CLEANUP: automations.yaml reduced from 7 active automations to 0. Deleted: Load Shedding Inverter Scene Switcher (superseded by orchestrator); 4× Solar Forecast Update automations (all select actions were enabled:false, scene targets never existed, superseded by E5-2 inverter_energy_pattern_control); Solar Forecast Check Appliances 8am-2pm (dead scenes). Migrated: Grid Usage Warning per day → automation.grid_import_daily_warning in power_automations.yaml with numeric_state triggers (no repeat-fire), script.notify_power_event routing, dead entity refs cleaned.*

---

## 🏠 System Identity

| Field | Value |
|---|---|
| Name | HABiesie |
| Location | Johannesburg, South Africa |
| Timezone | Africa/Johannesburg |
| Platform | Home Assistant OS (Supervisor enabled) |
| Network | 10.10.1.x LAN |
| Config path | `/config/` |
| Snapshot/image path | `/config/www/` → served as `/local/` |
| Backup script | `./gitupdate.sh "message"` |
| Validation | HA Developer Tools → YAML → Check Configuration |

---

## 🧱 Architecture Rules (NEVER VIOLATE)

1. **Package-based modular design** — all config in `packages/` via `!include_dir_named packages`
2. **Layering order within each package:**
   - `*_helpers.yaml` — input booleans, input numbers, counters, timers
   - `*_templates.yaml` — template sensors (derived values only)
   - `*_state.yaml` — state tracking and aggregation
   - `*_automations.yaml` — business logic
   - `*_core.yaml` — main integration config
3. **No new automations in `automations.yaml`** — legacy UI file, 3836 lines, DO NOT TOUCH
4. **Diagnostic sensors excluded from recorder** — maintain for all new diagnostic entities
5. **Trusted proxy** `172.30.0.0/16` for Docker reverse proxy HTTPS
6. **All notifications through central scripts** — never call `notify.*` directly
7. **Zone arming separates from zone motion** — aggregation sensors emit raw zone motion (ground truth). Arming gates are separate template sensors. Consumers (classifier, alerts, dashboard) read both. Do not merge motion + arming into a single "armed-and-firing" sensor (that collapses ground truth).

---

## 📦 Actual Package Structure (Verified 2026-04-16)

```
packages/
  power/          # 25 files — dual inverter, solar, battery, prepaid, energy, automations, statistics
  water/          # 20 files — tank lifecycle, safety aborts, audit
  alerts/         # 16 files — see ALERTS_CONTRACT.md for actual file list (alerts_batteries.yaml added 2026-05-27, alerts_device_batteries.yaml added 2026-08-21)
  lighting/       # 14 files — presence-aware and time-based scenes
  notifications/  # 12 files — routing, quiet hours, severity (includes water/power/presence/security/system scripts)
  security/       # 7 files  — cameras_core, cameras_processing, security_helpers,
                 #             security_core, security_logic, security_zones,
                 #             security_automations
  presence/       # 6 files  — presence_helpers, presence_core, presence_confidence,
                 #             presence_boundary, presence_validation, presence_trust (migrated from context/ 2026-04-30)
  context/        # 2 files  — context_global, context_night (context_presence + context_schedules deleted 2026-04-30/28)
  core/           # 4 files  — core_helpers (startup guard), ha_monitoring, sensor_smoothing,
                 #             tuya_health (Tuya Cloud staleness watchdog + auto-reload — added 2026-08-04)
  network/        # 3 files  — network_helpers (WAN health, UniFi speed, latency, jitter, NOC status);
                 #             network_ups (EcoFlow River Pro UPS monitoring — added 2026-05-27);
                 #             network_nas (Synology NAS graceful shutdown + WoL restore — added 2026-05-28)
  alerts/ already counted above; also covers: network, temperature, system_health, media, doors, water, security, presence, power
  backup/         # 1 file   — github.yaml (5AM git backup, script, status sensor)
  integrations/   # 1 file   — sonoff.yaml (Sonoff/EWElink config)
  office/         # 1 file   — office_helpers.yaml (printer cartridge monitoring)
  weather/        # 2 files  — weather_core.yaml, weather_helpers.yaml (OWM health tracking)
  sensors/        # 1 file   — filter.yaml (sensor smoothing utilities)
  load_shedding/  # 2 files  — load_shedding_templates.yaml (migrated from power/ 2026-04-21)
                 #             load_shedding_automations.yaml (3 warning automations migrated 2026-04-29)
  admin/          # 1 file   — tablets.yaml (screen brightness management for dashboard tablets — added 2026-05-27)
  utilities/      # 8 files  — Water Cooler (4 files, added 2026-08-31) + Gas Bottles
                 #             (4 files, added 2026-09-02) — recurring consumable-delivery
                 #             subsystems, see UTILITIES_CONTRACT.md. Row added 2026-09-02
                 #             (missing since this package's creation — this whole table is
                 #             dated 2026-04-16 and is broadly stale beyond just this row,
                 #             e.g. `integrations/` below also predates vacuum.yaml; not
                 #             fixed comprehensively here, out of scope for a targeted sweep)
```

**Important:** Several `*_CONTEXT.md` documents list incorrect filenames. The `*_CONTRACT.md`
files are authoritative for actual file inventory.

---

## ⚡ Hardware Summary

### Inverters
- **Dual Master/Slave Sunsynk** — Master: grid/losses/BMS, Slave: PV/load/battery
- Aggregated in `power_core.yaml` into unified sensors
- Solar forecast: Solcast, cached in `solcast_solar/`

### Solar PV Array (physical spec — added 2026-08-21, sourced from install/COC PDFs;
corrected same day against `private_docs/POWER_SYSTEM_AUDIT_2026.md` + `SOLAR_UPGRADE_ROI_2026.md`,
which are the actually-authoritative, actively-maintained record for this hardware —
check those (and this file's 2026-06-17 session log entry) before editing this block again)
- **Panels:** 24× 430W JA Solar, 4 strings × 6 panels, installed 2021-07-01 (SANS10142/
  NRS097 test report, Order IN070721A, OPS360/Shabi Electrical) — **documented "north
  facing"** at install. Wattage confirmed 2026-08-21: 430W is the fleet spec (matches the
  install PDF and the "9.89 kWp" figure, 23×430W); 435W — previously mislabeled onto the
  whole fleet in an earlier session — is specifically the one damaged panel's own rating.
- **Inverters:** 2× SunSynk 5.5kW hybrid (48V), 11kW combined nameplate AC — **pending
  replacement, now 2026-08-31 (Monday)**, slipped from 2026-08-28 (user-reported
  2026-08-29, no reason given) — with a single 12kW Deye + 12× 620W Canadian bifacial
  panels added (Vantage quote; see `SOLAR_UPGRADE_ROI_2026.md`).
- **Batteries:** **3× Greenrich AF1600 (GFM052-314-N00-A), 51.2V/314Ah/16kWh each — 48.2kWh
  bank total.** Swapped 2026-06-17 from the original 2× Freedom Won 15kWh (30kWh) units,
  which had 2 damaged cells traced back to a 2025-05-16 lightning/BMS event — do NOT use
  the Freedom Won spec, it's superseded. Full swap detail: this file's 2026-06-17 session
  log entry (further down) and `POWER_SYSTEM_AUDIT_2026.md` §3.3.
- **Known fault (2026-08):** 1 panel disconnected/broken (rated 435W, see wattage note
  above). Effective working DC nameplate: 23 × 430W = **9,890W (9.89kWp)**.
- **Solcast site config** (`solcast_solar/solcast-sites.json`, "Home Biesie",
  resource `a900-21df-4d5b-a523`) — was miscalibrated (capacity 10.2kW AC / 10.8kW DC,
  azimuth 41°, tilt 26°, loss_factor 0.9), **corrected 2026-08-21, confirmed live via
  the Solcast portal's Site Summary panel: azimuth → 0°, capacity_dc → 9.89kW,
  tilt → 18.4°.** AC capacity (10.2kW) left unchanged — no evidence it was wrong. Local
  cache file above will show the old values until the integration's next site-data sync
  (`solcast_solar.clear_all_solcast_data` action forces it immediately if wanted; forecast
  data itself is computed server-side by Solcast, so it's already using the corrected
  geometry regardless of local cache staleness). **Monitoring, not yet re-evaluated:**
  user chose to keep `select.solcast_pv_forecast_use_forecast_field` at `estimate10` and
  watch whether the geometry fix alone moves the actual-vs-forecast ratio before deciding
  whether to revert to `estimate` — see POWER_CONTRACT.md Issue 28 for the comparison plan
  (exclude pre-2026-08-21 days, corrected geometry; also exclude pre-~2026-08-15 days,
  tree felling).
  **Tilt derivation (2026-08-21):** no roof pitch documented in the install/CoC PDFs
  (unlike azimuth's explicit "north facing"). User phone-measured the roof with the iOS
  Measure app's Level tool: +facing up-roof = -5°, facing down-roof = -18° — the two
  readings don't mirror cleanly (implies either a large phone-level calibration offset or
  an orientation inconsistency between the two readings), but both are clearly well below
  the configured 26°. User then matched the readings against PVWatts' standard roof-pitch
  conversion table (rise/run → tilt degrees) and picked **4/12 pitch = 18.4°** — a common
  residential rafter pitch, and consistent with both raw phone readings pointing well
  below 26°. Treat as user-confirmed, not re-derive without new evidence. See
  POWER_CONTRACT.md Issue 28 for the forecast-bias investigation this feeds into.

### Cameras (verified 2026-05-17 — 7 NVR + 5 IP active)
- NVR: Hikvision DS-7116HGHI-F1 (16-channel hybrid DVR, analog 1080p, no AI)
- Active NVR (7): cam04, cam05_inside_garage, cam07, cam09, cam12, cam14, cam15
- Active IP AcuSense (5): ipcam01, ipcam02, ipcam03, ipcam04, ipcam05 — fully wired 2026-05-08
- NVR roadmap: existing NVR analog cameras will be progressively replaced by IP cameras. Empty NVR channels (formerly cam02/03/11/13/16-future) were removed 2026-05-17. No new analog cameras will be added to this NVR.
- Deprecated 2026-05-07: cam01_street_driveway (→ipcam01+02), cam10_pool_bar (→ipcam04)
- Uninstalled 2026-05-08: cam06_front_entrance (→ipcam03)
- All deprecated/uninstalled entity definitions purged from packages/security/ 2026-05-17
- cam05 renamed: cam05_front_driveway → cam05_inside_garage (2026-05-17)
- Entity prefix: `ipcam` for all IP cameras (ipcam01–ipcam05) — standardised 2026-05-08
- Streaming: go2rtc (timeout instability with NVR)
- See SECURITY_CONTRACT.md Section 3 for the full zone map
- Frigate: **planned, NOT implemented**
- Integration: `hikvision_next` (custom, unstable)

### Water
- Borehole pump: `switch.borehole_pump`
- Depth sensor: Tuya (`sensor.water_tank_depth_validated`)
- Pressure pump: `switch.water_pressure_pump`

### Network
- WAN Router: ASUS ZenWiFi XD6 (192.168.1.3) | Gateway: UniFi Dream Machine (downstream LAN routing) | APs: 5x UniFi
- ASUS ROG router (192.168.1.1) — NOT a WAN router; provides dual-LAG bonded LAN connectivity for the Synology NAS only (corrected 2026-07-13, was previously mislabeled "WAN Router: ASUS ROG" here)

### Zigbee Door/Gate Sensors (SNZB-04P, added 2026-08-23)
- Hub: SONOFF Dongle-M (zha), 21 devices / 233 entities on the Zigbee network as of this
  writing.
- Fleet: `bar_door_sensor`, `front_door_sensor`, `front_security_gate_sensor`,
  `lounge_door_sensor` (pre-existing before this session) + `kitchen_door_sensor`,
  `laundry_door_sensor`, `laundry_security_gate_sensor`, `garage_security_gate_sensor`,
  `reading_room_door_sensor` (new 2026-08-23). Each device exposes `binary_sensor.<name>`
  (opening) + `_tamper`, `sensor.<name>_battery`, `_lqi`, `_rssi`, `update.<name>_firmware`,
  `button.<name>_identify`.
- `garage_security_gate_sensor` is a separate pedestrian/security gate at the garage,
  distinct from the existing `binary_sensor.garage_door_sensor` (Sonoff, DW2-WiFi) —
  two different physical sensors covering two different things.
- See "Zigbee Door/Gate Sensors" under Locked Entity Names below for the entity-ID
  rename history, and ALERTS_CONTRACT.md's Doors Domain section for tier placement.

---

## 🔒 Locked Entity Names (DO NOT RENAME)

### Camera Entities
```
camera.camXX_location_description             e.g. camera.cam04_car_port_front
binary_sensor.camXX_location_motiondetection  NVR raw signal
# 2026-05-17: cam05 renamed cam05_front_driveway → cam05_inside_garage
# NVR hardware entity cam05_front_driveway_motiondetection name unchanged (integration-provided)
# Active NVR: cam04/05_inside_garage/07/09/12/14/15 + IP: ipcam01-05
# DO NOT reintroduce: cam01_street_driveway, cam06_front_entrance, cam10_pool_bar
binary_sensor.camXX_location_motion_valid     debounced output
```

### Core Security Sensors
```
sensor.security_trigger_camera
sensor.security_event_classification
sensor.security_correlation            ← unique_id: security_correlation (BUG was stale — verified present 2026-06-19)
sensor.security_threat_level
sensor.security_threat_score
sensor.security_movement_confidence
sensor.security_movement_path
sensor.security_mode
sensor.security_lighting_intent
input_text.security_event_session
input_text.security_last_path
input_text.security_last_motion_camera
input_text.security_last_motion_image
counter.security_grounds_low_confidence_count   ← added 2026-07-17 (IMPROVEMENT-S67), daytime downgrade repeat counter
input_datetime.security_event_start
input_datetime.last_security_event
input_datetime.last_intruder_event
input_datetime.last_visitor_event
input_boolean.security_alert_active
input_boolean.security_system_enabled
input_boolean.inside_cameras_armed              ← auto-managed by arming automation
input_boolean.inside_cameras_schedule_override  ← dashboard override, force-arms cam14/cam15
binary_sensor.security_gate_loitering           ← added 2026-07-02 (S17), delay_on 7s on ipcam01 regionentrance
input_boolean.security_visitor_alerts_suppressed ← added 2026-08-31 (BUG-S77), scoped mute for the visitor router branch only
                                                  ← dashboard entry added 2026-09-02 (was built but never wired into any
                                                    dashboard — Operations → Security → "Camera System Control" card)
```

### Presence Persons
```
sensor.ryan_ap_location      sensor.vicky_ap_location
sensor.luke_ap_location      sensor.tayla_ap_location
person.ryan_dunnington        person.vicky_dunnington
person.luke_dunnington        person.tayla_dunnington
device_tracker.ryan_iphone_tracker   ← NOTE: _tracker suffix is correct
device_tracker.vicky_iphone_tracker  device_tracker.luke_iphone_tracker
# ⚠️ 2026-08-24 (BUG-P22): Tayla's sibling entity, device_tracker.tayla_iphone_tracker,
# was renamed to device_tracker.iphone14_tayla_tracker during an unrelated entity-
# renaming cleanup — despite this section's own "DO NOT RENAME" heading. Broke
# sensor.tayla_ap_location + unknown-AP detection silently (same class as BUG-P08)
# until caught and the 4 template references updated. Tayla is now the one exception
# to the `<name>_iphone_tracker` pattern above — do not "fix" her template references
# back to tayla_iphone_tracker, that entity no longer exists.
device_tracker.iphone14_tayla_tracker   ← Tayla's actual entity_id (see warning above)
```

### Trust Model — CRITICAL RULE
```
# ALWAYS USE THESE (derived, auto-set by schedules):
binary_sensor.staff_on_site
binary_sensor.low_trust_present

# NEVER USE THESE in automations (manual, never auto-set):
input_boolean.staff_on_site      ← BUG IV-01 FIXED 2026-04-14 (4 security automations corrected)
input_boolean.low_trust_present
```

### Zigbee Door/Gate Sensors (added 2026-08-23)
```
binary_sensor.kitchen_door_sensor               # renamed from kitchen_kitchen_door_sensor
binary_sensor.reading_room_door_sensor          # renamed from reading_room_reading_room_door_sensor
binary_sensor.garage_security_gate_sensor       # renamed from garage_garage_security_gate_sensor
binary_sensor.laundry_security_gate_sensor      # renamed from laundry_laundry_security_gate_sensor
binary_sensor.laundry_door_sensor               # entity_id was already clean, no rename needed
# All 4 renames: doubled-prefix bug from Area not being set before HA auto-named the
# entity at creation. unique_id untouched on all 4 — only entity_id changed.
# DO NOT reintroduce the doubled-prefix names above.
automation.laundry_entry_event                  # presence_boundary.yaml, mirrors house_entry_event
automation.laundry_departure_event              # presence_boundary.yaml, mirrors house_departure_event
automation.house_secured_check                  # alerts_doors.yaml, bedtime + everyone-left sweep
input_datetime.house_secured_check_time         # default 21:30, dashboard-adjustable
```

### Door Alert Inputs (BUG-A06 — added 2026-04-16)
```
input_number.tier3_door_evening_start          # 21.0 h — evening mode start for lounge/bar
input_number.entry_door_night_escalation_minutes # 5 min — tier-2 night escalation
sensor.door_alert_context                       # SINGLE SOURCE — replaces doors_open_alert_severity
# sensor.doors_open_alert_severity DELETED
# 2026-08-23: garage_door_sensor split out of the shared Tier 2 loop into its own
# condition block — home-branch now gated on binary_sensor.security_lighting_required
# AND binary_sensor.all_family_home instead of the generic night flag. See
# ALERTS_CONTRACT.md Doors Domain section.
```

### Laundry Door + Gate Alert Mute (added 2026-09-02)
```
input_boolean.laundry_door_alert_notify         # alerts_doors.yaml — covers BOTH
                                                 # binary_sensor.laundry_door_sensor AND
                                                 # binary_sensor.laundry_security_gate_sensor
                                                 # (one boolean, both entities split out of the
                                                 # shared Tier 2 loop into their own gated block)
automation.laundry_door_alert_midnight_reset    # forces the boolean back to `on` at 00:00:00 —
                                                 # same pattern as dogs_inside_midnight_clear
                                                 # (security_automations.yaml)
# Dashboard: Operations → Security → "Door Control" entities card.
# Deliberately does NOT gate automation.house_secured_check (different kind of check —
# physical-security sweep, not the nuisance-alert pipeline).
```

### Gate Alert Camera + Cancel (BUG-A13 — added 2026-07-17)
```
input_boolean.gate_alert_snoozed                # per-open-cycle mute; auto-clears when
                                                 # sensor.door_alert_context returns to "normal"
automation.gate_alert_cancel_from_notification  # handles CANCEL_GATE_ALERT (phone) /
                                                 # /cancel_gate_alert (Telegram) taps
automation.gate_alert_snooze_reset              # auto-clear on door_alert_context == normal
/config/www/gate_alert_main_gate_latest.jpg           # fresh snapshot, camera.ipcam03_driveway
/config/www/gate_alert_front_security_gate_latest.jpg # fresh snapshot, camera.cam04_car_port_front
```

### Cancel Alert Rollout (BUG-A19 — added 2026-08-18)
```
# Same gate_alert_snoozed pattern (BUG-A13 above), rolled out to every other
# critical alert domain's repeat-reminder stream. Naming convention:
# input_boolean.<x>_alert_snoozed / automation.<x>_alert_cancel_from_notification
# / automation.<x>_alert_snooze_reset — one triplet per stream.
input_boolean.power_alert_snoozed                          # alerts_power.yaml
input_boolean.water_alert_snoozed                           # alerts_water.yaml (tank/safety)
input_boolean.water_borehole_fault_snoozed                  # alerts_water.yaml (tier 2, >=3)
input_boolean.water_borehole_critical_fault_snoozed         # alerts_water.yaml (tier 3, >=5)
input_boolean.wan_temp_alert_snoozed                        # alerts_temperature.yaml
input_boolean.lan_temp_alert_snoozed                        # alerts_temperature.yaml
input_boolean.device_temp_alert_snoozed                     # alerts_temperature.yaml
input_boolean.storage_temp_alert_snoozed                    # alerts_temperature.yaml
input_boolean.device_power_alert_snoozed                    # alerts_device_power.yaml
input_boolean.media_alert_snoozed                           # alerts_media.yaml
input_boolean.network_device_down_alert_snoozed             # alerts_network.yaml
input_boolean.wan_down_alert_snoozed                        # alerts_network.yaml
input_boolean.wan_degraded_alert_snoozed                    # alerts_network.yaml
input_boolean.device_restart_alert_snoozed                  # alerts_network.yaml
input_boolean.security_alert_snoozed                        # alerts_security.yaml (replaces the
                                                              # security_alert_notify global-mute
                                                              # workaround the repeat reminder used)
input_boolean.dash_battery_alert_snoozed                    # alerts_batteries.yaml
input_boolean.presence_alert_snoozed                        # alerts_presence.yaml
input_boolean.garden_alert_snoozed                          # alerts_garden.yaml (alongside existing
                                                              # TURN_OFF_POND_PUMP action button)
input_boolean.critical_sensor_health_alert_snoozed          # alerts_system_health.yaml
# --- Same snooze/cancel/auto-reset pattern, but NOT part of BUG-A19's rollout (that was
# specifically the repeat-reminder streams, alert:/binary_sensor.*_active architecture).
# BUG-S77 (2026-08-31) reused the pattern for a different alert class — a one-shot
# classifier-transition push, no repeat: schedule involved:
input_boolean.visitor_alert_snoozed                          # security_automations.yaml
                                                              # (BUG-S77) — automations are
                                                              # security_visitor_alert_cancel_
                                                              # from_notification / _snooze_reset
                                                              # (security_ prefix, not the plain
                                                              # visitor_alert_* BUG-A19 convention,
                                                              # to match this file's existing IDs)
# Each has a matching automation.<x>_alert_cancel_from_notification (handles the
# CANCEL_<X>_ALERT phone action / /cancel_<x>_alert Telegram tap) and
# automation.<x>_alert_snooze_reset (auto-clears on the underlying binary_sensor/
# context sensor returning to off/normal). Not added: Camera Health — its
# alert: repeat schedule has no live notifier at all, nothing to cancel.
# notify_power_event.yaml / notify_water_events.yaml / notify_presence_events.yaml
# gained actions:/telegram_action: passthrough (notify_system_event.yaml already had
# actions:, gained telegram_action:) — same convention as notify_security_events.yaml.
```

### Dogs Inside Notification Toggle (BUG-S68 — added 2026-07-17)
```
automation.dogs_inside_off_from_notification    # DOGS_INSIDE_OFF action → turns off
                                                 # input_boolean.dogs_inside (pairs with the
                                                 # pre-existing dogs_inside_from_notification/
                                                 # DOGS_INSIDE_ON handler — see line 1288 below)
```

### Water Entities (expanded 2026-04-16)
```
sensor.water_refill_blocked_reason              # "none" or human-readable block reason
binary_sensor.water_borehole_fault_alert_active # ON when faults_today >= 3
binary_sensor.water_borehole_critical_fault_active # ON when faults_today >= 5
counter.water_borehole_faults_today             # daily fault counter — YAML only (storage duplicate removed 2026-04-21)
counter.water_borehole_faults_this_week         # weekly fault counter — YAML only (storage duplicate removed 2026-04-21)
alert.water_borehole_fault                      # level 2 fault alert
alert.water_borehole_critical_fault             # level 3 fault alert
```

### Presence Alert Entities (B1 — added 2026-04-16)
```
binary_sensor.unknown_ap_alert_active           # delay_on 5 min, delay_off 2 min
binary_sensor.occupancy_anomaly_alert_active    # delay_on 15 min — info only
sensor.presence_alert_context                   # warning/critical by AP duration
alert.presence_alert                            # STD_Alerts, repeat 60/120 min
input_boolean.presence_alert_notify             # pipeline toggle
```

### Security Dedup Gate (triple-fire fix — added 2026-04-16)
```
binary_sensor.security_intruder_active          # ON when correlation = intruder/intruder_high
                                                # delay_off: 30s — prevents re-fire on flap
```

### Alert Architecture Sensors
```
sensor.global_alert_context
sensor.alert_device_entities      ← SINGLE SOURCE OF TRUTH
sensor.active_alert_entities
sensor.total_critical_alert_devices
sensor.total_warning_alert_devices
sensor.total_info_alert_devices
sensor.total_active_alerts
sensor.total_acknowledged_alerts
sensor.total_unacknowledged_alerts
```

### Dashboard Battery Alert Entities (added 2026-05-27)
```
# alerts_batteries.yaml
input_boolean.dash_battery_alert_notify           ← pipeline suppress toggle
input_number.dash_battery_warning_threshold       ← 30% low alert trigger
input_number.dash_battery_critical_threshold      ← 15% critical severity
input_number.dash_battery_overcharge_threshold    ← 95% overcharge trigger
input_number.honor10_dash_battery_capacity_wh     ← 39.1 Wh (Honor Pad 10, HEY3-W00 — 10100 mAh × 3.87V)
input_number.honorx7_dash_battery_capacity_wh     ← 27.2 Wh (Honor Pad X7, JMS-W09 — 7020 mAh × 3.87V)
sensor.honor10_dash_battery_time_remaining_est    ← minutes remaining (-1 = charging/unavailable)
sensor.honorx7_dash_battery_time_remaining_est    ← minutes remaining (-1 = charging/unavailable)
binary_sensor.honor10_dash_battery_low            ← delay_on 1 min, < warning_threshold AND not charging
binary_sensor.honorx7_dash_battery_low            ← same for X7
binary_sensor.honor10_dash_battery_overcharge_active ← delay_on 2 h, ≥95% while charging
binary_sensor.honorx7_dash_battery_overcharge_active ← same for X7
binary_sensor.dash_battery_alert_active           ← master trigger (any low OR overcharge)
sensor.dash_battery_alert_context                 ← canonical context sensor for aggregator
alert.dash_battery_alert                          ← 30/60 min repeat, STD_Alerts

# tablets.yaml (admin package) — screen brightness
input_number.dash_brightness_day                  ← 150 (0–255 scale)
input_number.dash_brightness_night                ← 20
input_number.dash_brightness_away                 ← 40
automation.tablets_brightness_night_dim           ← night_mode ON → night brightness
automation.tablets_brightness_morning_restore     ← night_mode OFF → day (or away if nobody home)
automation.tablets_brightness_away_dim            ← nobody_home ON → away brightness (if not night)
automation.tablets_brightness_return_restore      ← nobody_home OFF → day brightness (if not night)

# Entities to ENABLE in HA Settings → Entities (currently disabled by integration):
#   binary_sensor.honor10_dash_interactive  — screen on/off (useful for future proximity logic)
#   binary_sensor.honorx7_dash_interactive  — screen on/off
#   binary_sensor.honorx7_dash_doze_mode    — device doze/sleep state
```

### Device Battery Alert Entities (added 2026-08-21, sparse-reporter override added 2026-08-31)
```
# alerts_device_batteries.yaml — ALL other battery devices (door/gate sensors,
# gate doorbell, phones, watches, laptops). Excludes inverter/UPS (Power domain)
# and the Honor dashboards (alerts_batteries.yaml above — do not merge/duplicate).
# Onboarding a new device = apply the "battery_monitor" label to its battery
# entity — NOT a per-device YAML edit. See label_id below.
input_boolean.device_battery_alert_notify              ← pipeline suppress toggle
input_boolean.device_battery_alert_snoozed             ← BUG-A13 per-cycle Cancel Alert mute
input_number.device_battery_warning_threshold          ← 10% low alert trigger (fleet-wide, not per-device)
input_number.device_battery_critical_threshold         ← 5% critical severity
input_number.device_battery_critical_days_remaining    ← 1 day — rate-based critical trigger
input_number.device_battery_stale_hours_sparse         ← 96h — BUG-A22 (2026-08-31), overrides
                                                           device_battery_stale_hours for entities
                                                           ALSO labelled battery_monitor_sparse_reporter
sensor.device_battery_history_log                      ← self-referencing daily snapshot log, 14-day window per entity
sensor.device_battery_fleet                             ← full roster (all severities) — dashboard reads this directly
sensor.device_battery_alert_context                     ← canonical context sensor for aggregator (non-normal devices only)
binary_sensor.device_battery_alert_active               ← master trigger, delay_on 1 min
alert.device_battery_alert                               ← 30/60 min repeat, STD_Alerts, Cancel Alert button

# core.label_registry
label_id: battery_monitor   ← name "Battery Monitor" — apply to any new battery
                               device's entity (or its device) to onboard it into
                               both this alert pipeline and the Batteries dashboard
                               view automatically, no restart needed for onboarding

label_id: battery_monitor_sparse_reporter   ← name "Battery Monitor (Sparse Reporter)",
                               added 2026-08-31 (BUG-A22). Apply ALONGSIDE battery_monitor
                               (not instead of it) to a device whose battery attribute
                               reports are naturally infrequent — makes
                               sensor.device_battery_fleet use
                               input_number.device_battery_stale_hours_sparse (96h)
                               instead of the global device_battery_stale_hours (24h) for
                               that entity's staleness check. Applied 2026-08-31 to the
                               10 SNZB-04P Zigbee door/gate battery entities below
                               (marked ★) via config/entity_registry/update — do NOT
                               apply to phones/watches/laptops or the non-ZHA
                               garage_door_sensor_battery/doorbell, which report
                               normally and should keep using the global threshold.

# Initial 2026-08-21 rollout was documented as 15 entities labelled battery_monitor —
# ⚠️ 2026-08-23: found live at ZERO entities labelled (sensor.device_battery_fleet
# reporting an empty roster) despite this doc and ALERTS_CONTRACT.md both saying PASS.
# Root cause not identified. Restored the original 15 below AND added the 5 new
# door/gate sensors from the same session's Zigbee onboarding (see this file's
# 2026-08-23 session log entry) — 20 entities labelled as of 2026-08-23, applied live
# via the HA WebSocket API (config/entity_registry/update), verified live via
# sensor.device_battery_fleet showing all 20.
sensor.bar_door_sensor_battery                    ★
sensor.front_door_sensor_battery                  ★
sensor.front_security_gate_sensor_battery         ★
sensor.gate_sensor_battery                        ★  (Main Gate)
sensor.lounge_door_sensor_battery                 ★
sensor.garage_door_sensor_battery                    # Sonoff platform, NOT ZHA — no ★, reports normally
sensor.ezviz_main_gate_doorbell_battery              # no ★, reports normally
sensor.ryan_iphone16_mobile_app_battery_level
sensor.iphone13promax_vicky_battery_level
sensor.iphone14_tayla_mobile_app_battery_level  # renamed 2026-08-24, was tayla_iphone14_mobile_app_*
sensor.luke_iphone15_mobile_app_battery_level
sensor.iphone16promax_ryan_watch_battery_level
sensor.luke_iphone15_mobile_app_watch_battery_level
sensor.ap_0223_1001_internal_battery_level  # name_by_user set 2026-08-31 -> "Ryan Macbook
                                             #   Pro Mobile App" (BUG-A23) — this IS Ryan's
                                             #   MacBook (Mac14,10), was displaying its raw
                                             #   hostname because no custom name had been set
# sensor.ryan_macbook_pro_mobile_app_internal_battery_level — REMOVED 2026-08-31 (BUG-A23).
#   Confirmed duplicate mobile_app device registration of the SAME physical Mac as
#   ap_0223_1001 above (both device model Mac14,10, both hostname "AP-0223-1001") — this
#   was the ORIGINAL Jan-2025 registration, orphaned since the Companion App re-registered
#   under a new device ID in May 2026; it stopped reporting at HA's 2026-08-24T18:16 restart
#   and would never report again. Deleted outright (DELETE /api/config/config_entries/entry/
#   {id} on config entry 01JH87XY6X1DQVYEHY2J4NNW81 — mobile_app registers one config entry
#   per app install, so this cleanly removed the dead device + its ~23 entities with no
#   effect on the live one). Resolves the "flagged, not deduped" open question from the
#   2026-08-21 rollout entry below.
# + 5 new (2026-08-23):
sensor.kitchen_door_sensor_battery                ★
sensor.reading_room_door_sensor_battery           ★
sensor.garage_security_gate_sensor_battery        ★
sensor.laundry_door_sensor_battery                ★
sensor.laundry_security_gate_sensor_battery       ★
# ★ = also carries battery_monitor_sparse_reporter (BUG-A22, 2026-08-31) — all 10 are
#     the real SNZB-04P Zigbee platform entities; see label_registry note above.
# Deliberately NOT labelled: sensor.ha_system_monitor_battery (HA host/Pi has no
# real battery). No iPad has a battery entity yet (device_tracker.tayla_ipadair5th
# / device_tracker.ipadpro_luke are unifi presence trackers only, no HA app) —
# the dashboard's Tablets section shows an onboarding hint until one exists.
# ⚠️ Watch coverage gap found 2026-08-24: only Ryan's and Luke's Apple Watches have
# battery entities (via their phone's mobile_app Watch extension). Checked live —
# Vicky's iPhone (iphone13promax_vicky, 21 mobile_app entities) has NO watch_battery_
# level sensor at all; likely her Watch app's "Share Watch Battery with HA" permission
# was never enabled on her phone (phone-side fix, not a HA config change). Tayla's
# watch is a Huawei, not Apple — there is no native HA path for Huawei Health battery
# data at all (Apple's WatchKit battery bridge is iOS-only); would need a manual
# relay (e.g. Tasker/HTTP shortcut) to ever appear here, nothing to onboard today.

# Dashboard: packages/operations dashboard, new "Batteries" view
# (.storage/lovelace.dashboard_operations, path battery-monitor, inserted right
# after "Mobile Devices" at view index 12) — sections for iPhones / Watches /
# Tablets (non-dashboard) / Other Devices, each a markdown card templated off
# sensor.device_battery_fleet's `devices` attribute (auto-updates, no per-device
# card edits ever needed).
```

### NAS Protection Entities (added 2026-05-28)
```
# network_nas.yaml — Synology graceful shutdown + WoL restore
input_boolean.ups_nas_auto_shutdown_enabled    ← cancel gate; toggle OFF to abort pending shutdown
input_boolean.ups_nas_was_shutdown             ← flag set at shutdown; gates WoL on HA restart
input_number.ups_nas_shutdown_warn_minutes     ← Stage 1 threshold (default 15 min)
input_number.ups_nas_shutdown_grace_minutes    ← grace between warning and forced shutdown (default 5 min)
input_number.ups_pi_shutdown_wait_minutes      ← wait after NAS shutdown before Pi shutdown (default 4 min)
binary_sensor.ups_nas_shutdown_imminent        ← ON when on battery AND runtime < warn threshold

# External entities (not HA-defined — Synology DSM + WoL integrations)
button.guardians_shutdown                      ← Synology DSM shutdown
button.wol_synology_nas_00_11_32_ad_af_a5      ← WoL magic packet to LAN 1 NIC (MAC 00:11:32:ad:af:a5)
                                                  broadcast_address: 192.168.1.255 (fixed 2026-05-28 — was wrong unicast IP)
```

### Power Core Sensors
```
sensor.inverter_battery_soc      sensor.inverter_load_power
sensor.inverter_grid_power       sensor.inverter_pv_power
sensor.inverter_battery_power    sensor.grid_energy_import_today
sensor.inverter_today_production group.inverter_grid
group.inverter_battery_state     sensor.prepaid_units_left_safe
```

### Force Charge Entities (added 2026-06-18)
```
input_boolean.force_charge_active        ← ON while force charge run is in progress
input_number.force_charge_target_soc     ← target SOC % (50–100, default 100)
input_text.force_charge_saved_charging   ← JSON snapshot of P1–P6 charging+SOC before override
script.force_charge_batteries            ← starts force charge; saves settings, enables Grid all programs
script.force_charge_cancel               ← dashboard cancel button → calls force_charge_restore
script.force_charge_restore              ← internal restore: INV1 settings → sync INV2 → re-enable auto
                                           FIXED 2026-06-19: re-enable inverter_programme_auto_enabled and
                                           clear force_charge_active NOW run BEFORE the guard condition.
                                           Previously guard abort left programme_auto OFF indefinitely.
automation.force_charge_monitor          ← template trigger: fires when force_charge_active=on AND SOC ≥ target
                                           BUG FIXED 2026-06-19: original SOC-only trigger missed case where
                                           SOC was already AT target when force charge activated. force_charge_active
                                           is now embedded in the trigger template so it fires on both SOC change
                                           AND force_charge_active transition to ON.
```

### Power Statistics Sensors (revised 2026-06-18)
```
# Monthly utility meters added to energy_helpers.yaml
sensor.battery_discharge_monthly         ← utility_meter monthly on inverter_today_battery_discharge
sensor.battery_charge_monthly            ← utility_meter monthly on inverter_today_battery_charge

# 7d/30d mean — now template sensors (were platform:statistics, wrong computation)
sensor.inverter_production_7d_total      ← statistics change/7d on inverter_total_production (EXCLUDED recorder)
sensor.inverter_production_30d_total     ← statistics change/30d on inverter_total_production (EXCLUDED recorder)
sensor.inverter_production_7d_mean       ← template: 7d_total / 7 (INCLUDED in recorder)
sensor.inverter_production_30d_mean      ← template: 30d_total / 30 (INCLUDED in recorder)
```

### Geyser Scheduling (updated 2026-06-18)
```
# Threshold renamed and corrected:
input_number.geyser_adequate_daily_energy_by_midday  kWh 1.25  — midday-WINDOW energy threshold
                                                              (was 1.5 kWh, compared total daily energy
                                                               — now compares midday_end − morning_end delta only)
# New trigger id in geyser_turn_on automation:
# evening_early_winter → fires 17:00 in winter (was always 17:30 evening_early for all seasons)
# evening_early        → fires 17:30 in non-winter seasons (unchanged id)

# Midday forced minimum (added 2026-06-20) — bypasses solar gate if tank still cold
input_number.geyser_midday_forced_minutes_winter           min 60  winter, normal weekday
input_number.geyser_midday_forced_minutes_winter_extended  min 90  winter + staff_on_site OR weekend
input_number.geyser_midday_forced_minutes_summer           min 30  any non-winter day
# Trigger id: midday_force_check at 13:30/14:00/14:30 (force-on = 15:00 - required minutes)
```

### Power Statistics + Weather Correlation (power_statistics.yaml — P6 2026-06-14)
```
sensor.inverter_production_7d_mean        kWh  7-day rolling mean of daily PV production
sensor.inverter_production_30d_mean       kWh  30-day rolling mean
sensor.inverter_production_7d_stdev       kWh  7-day std dev (feeds CV → solar_weather_correlation)
sensor.house_load_24h_mean                W    24h mean load (used by P4 grid charge)
sensor.house_load_7d_mean                 kWh  7-day mean daily load
sensor.solar_vs_forecast_ratio_today      %    today actual/forecast; 100 before 10am (neutral guard)
sensor.solar_forecast_accuracy_7d         %    7d stats mean of solar_vs_forecast_ratio_today
sensor.solar_vs_forecast_ratio_7d         %    legacy: 7d_mean / today's Solcast (retained)
sensor.solar_weather_correlation               4-state: excellent/good/poor/degraded
                                               degraded triggers energy_state conserve branch
                                               'ratio' attribute preserved for energy_state decision_reason
sensor.solar_season_efficiency_factor          float: summer 1.0 / autumn 0.75 / winter 0.55 / spring 0.85
                                               calibrate after 3+ months of data
sensor.solar_production_vs_capacity       %    PV output / system capacity (solar_clipping.yaml)
input_number.system_solar_capacity_w      W    10800 — rated system capacity (update after upgrades)
input_number.solar_factor_{summer|autumn|winter|spring}  UI-adjustable season multipliers
```

### Power Runtime & Battery Sensors (battery_runtime.yaml)
```
sensor.ss_battery_capacity             ← program-aware target SOC
sensor.ss_soc_battery_time_left        ← runtime seconds to target
sensor.ss_soc_battery_time_left_friendly ← human-readable runtime
sensor.markdown_ss_discharge_time      ← ETA clock (HH:MM)
sensor.battery_runtime_severity        ← charging/ok/warning/critical
sensor.battery_minutes_remaining_safe  ← numeric minutes (automation-safe)
sensor.battery_runtime_confidence      ← high/medium/low/none
binary_sensor.battery_runtime_unreliable ← ON when data is bad
# NOTE: entity prefix is ss_* (NOT ssa_* — those were a dead legacy integration)
```

### Power Diagnostic Sensors (renamed 2026-04-21)
```
sensor.inverter_device_1_since_last_update  ← was inverter_1_device_since_last_update
sensor.inverter_device_2_since_last_update  ← unchanged
# unique_id updated — HA will create new entity; old entity_id becomes orphaned in registry
```

### Load Control (load_control.yaml — added 2026-04-21)
```
input_boolean.load_control_geyser_enabled    ← OFF turns off + blocks geyser
input_boolean.load_control_pool_enabled      ← OFF turns off + blocks pool pump
input_boolean.load_control_borehole_enabled  ← OFF turns off + blocks borehole (WARNING level)
```
Devices: switch.geyser_heat_pump_switch, switch.pool_pump_switch, switch.borehole_pump
# Renamed 2026-04-22: _switch_1 → _switch for both geyser and pool pump (HA entity registry + all YAML + dashboards)
✅ DONE 2026-04-29: load_control_borehole_enabled added to binary_sensor.water_refill_allowed state condition (water_templates.yaml:299)

### Geyser Entities (power_helpers.yaml + power_state.yaml + geyser_automations.yaml)
```
# Control booleans
input_boolean.geyser_sports_night              ← Tue/Thu auto-set 17:00; clears 00:01 daily
input_boolean.geyser_morning_override          ← bypasses load_control_geyser_enabled for morning
input_boolean.geyser_manual_run_active         ← ON while manual run in progress
input_boolean.geyser_reached_temp_today        ← set on at_temperature → on; reset midnight (2026-06-17)

# Window timing helpers
input_number.geyser_morning_start_weekday      h  4.5  Mon–Sat
input_number.geyser_morning_start_weekend      h  5.5  Sunday
input_number.geyser_morning_end_weekday        h  7.5  Mon–Sat (triggers 07:30/08:00)
input_number.geyser_morning_end_weekend        h  8.5  Sunday (trigger 09:00)
input_number.geyser_winter_start_offset        min 30
input_number.geyser_midday_surplus_threshold   W  300
input_number.geyser_last_heat_up_minutes       min    elapsed time at-temperature per session

# Daily minimum / adaptive evening (added 2026-06-17)
input_number.geyser_adequate_daily_energy_by_midday  kWh 1.5  gate for 17:30 early start
input_number.geyser_min_daily_energy_kwh             kWh 2.0  20:00 check minimum
input_number.geyser_energy_at_morning_end            kWh      snapshot at morning hard-off
input_number.geyser_energy_at_midday_end             kWh      snapshot at 15:00
input_number.geyser_grid_offline_critical_soc  %  20   critical window SOC floor when grid offline

# Sensors
sensor.geyser_control_status         ← 13-state priority display (power_state.yaml)
binary_sensor.geyser_at_temperature  ← ON when power < 50W sustained 5 min while switch on
sensor.geyser_daily_status           ← reached_temp/heating/low_energy/no_run (added 2026-06-17)
sensor.geyser_target_run_hours_today ← season-aware daily target (2.0h summer / 3.5h winter)
```

### Tuya Cloud Health Entities (core/tuya_health.yaml — added 2026-08-04, BUG-INFRA-TUYA01)
```
sensor.tuya_last_activity_age        ← seconds since ANY tracked Tuya entity last updated
                                        (geyser/pool/pond switches + water tank depth/liquid
                                        sensors) — no single entity is a safe canary alone
sensor.tuya_cloud_health             ← healthy (<2h) / delayed (<4h) / stale (>=4h)
script.tuya_reload_and_verify        ← reload Tuya config entry + 45s settle + re-check;
                                        shared by the watchdog and the notification retry button
automation.tuya_cloud_state_feedback_stale            ← id: tuya_cloud_stale_alert
automation.tuya_cloud_state_feedback_recovered        ← id: tuya_cloud_recovery
automation.tuya_cloud_retry_reload_from_notification  ← id: tuya_reload_from_notification;
                                        handles the TUYA_RETRY_RELOAD mobile action button
```

### Energy Orchestrator (power_helpers.yaml + energy_state.yaml — added E1 2026-06-14)
```
# State sensor — 6-state priority ladder
sensor.energy_orchestrator_state         ← loadshedding_critical/loadshedding/critical/conserve/surplus/normal
sensor.orchestrator_decision_reason      ← human-readable string explaining current state

# Master gate
input_boolean.orchestrator_enabled       ← when OFF, orchestrator always returns 'normal' (initial: true)

# SOC threshold ladder (all UI-adjustable, all read by template sensors via states()|float(default))
input_number.orchestrator_emergency_soc_threshold        ← default 15%
input_number.orchestrator_critical_soc_threshold         ← default 25%
input_number.orchestrator_conserve_soc_threshold         ← default 50%
input_number.orchestrator_pre_shed_soc_threshold         ← default 80%
input_number.orchestrator_conserve_degraded_soc_threshold ← default 60% (degraded solar branch)
input_number.orchestrator_surplus_export_threshold       ← default 1000W
input_number.orchestrator_target_soc_by_sunset           ← default 90% (future use)
input_number.orchestrator_load_first_soc_threshold       ← default 65% (future use)
input_number.orchestrator_pre_shed_hours_warning         ← default 3h

# Pool winter scheduling
input_number.pool_winter_start_hour                      ← default 10h (future use by pool automation)
```

### Inverter Sync (power_helpers.yaml + power_automations.yaml — added E1 2026-06-14; expanded E5)
```
input_boolean.inverter_sync_status       ← ON=in sync; OFF=mismatch; written by inverter_sync_check
automation.inverter_sync_check           ← triggers on energy_pattern + all 6 program_N_charging changes;
                                           90s delay; compares 7 entities (energy_pattern + 6 selects + SOC)
script.force_inverter_sync               ← dashboard-callable; copies energy_pattern + 6 charging selects
                                           + 6 SOC numbers from INV1 → INV2 (expanded E5 2026-06-14)
```

### Inverter Programme Automation (power_helpers.yaml + power_automations.yaml — added E5 2026-06-14)
```
# Gate toggles
input_boolean.inverter_programme_auto_enabled  ← default true — enables automated Battery/Load First control
input_boolean.use_legacy_solar_scenes          ← default false — gates solar_forecast.yaml scenes (rollback)

# P4 grid charge thresholds
input_number.orchestrator_p4_charge_trigger_soc  ← default 70% — SOC gap trigger for P4 grid charge
input_number.orchestrator_solar_gap_threshold    ← 5 kWh — shortfall threshold for P4 grid charge (updated from 2 kWh for 48 kWh bank)
input_number.battery_capacity_kwh               ← 48 kWh — 3× Greenrich AF1600 (updated from 30 kWh on 2026-06-17)
input_number.p4_grid_charge_solar_gate_w        ← 1500W default, added 2026-06-23 — Grid only enables if
                                                    current sensor.inverter_pv_power is below this, even when
                                                    the forecast-shortfall math says it's needed. Re-evaluated
                                                    (not sticky) every 30 min 14:00–16:30 — releases back to
                                                    solar if production recovers above the gate at a later check.

# Automations
automation.inverter_energy_pattern_control      ← Battery First / Load First crossover; 4 branches:
                                                  1. poor energy → Battery First
                                                  2. morning low SOC → Battery First
                                                  3. good solar 09–16h → Load First
                                                  4. 16:30 evening → Battery First
automation.inverter_p4_grid_charge_control      ← dynamic P4 grid charge 14:00–17:00 window;
                                                  enables if SOC gap > 5% AND kWh shortfall > threshold;
                                                  17:00 trigger always restores to Disabled
```

### Energy Simulator (power_helpers.yaml + pyscript/energy_simulator.py — added E8 2026-06-14)
```
# Safety gate + scenario control
input_boolean.simulator_active               ← default: false — must be ON to run simulator
input_select.simulator_scenario              ← 8 presets + Custom

# Override inputs (used in Custom mode; also shown in all scenario outputs)
input_number.simulator_soc_override          ← 0–100%
input_number.simulator_solar_override        ← 0–15000W
input_number.simulator_load_override         ← 0–8000W
input_number.simulator_hour_override         ← 0–23h
input_number.simulator_shed_stage_override   ← 0–6

# pyscript service (read-only — fires persistent_notification + event only)
pyscript.energy_simulator                    ← call via Developer Tools → Services
```

### Air Fryer + Unknown Draw Detection (power_helpers.yaml + power_automations.yaml — added E7 2026-06-14)
```
# Air fryer load control
input_boolean.load_control_airfryer_enabled    ← default true — auto-cut gate for airfryer_critical_cut
automation.airfryer_critical_cut               ← cuts switch.philips_airfryer_plug when grid OFF + SOC
                                                 < orchestrator_critical_soc_threshold; NOT fired by
                                                 orchestrator state alone — requires actual grid-off event
automation.airfryer_restore_on_recovery        ← info notify on grid return; does NOT auto-restart

# Unknown draw detection (Tier 4 alerts)
input_boolean.tier4_warnings_enabled           ← default true — gate for unknown_draw_warning
input_number.unknown_draw_warning_threshold    ← default 1500 W — warning threshold
input_number.unknown_draw_critical_threshold   ← default 3000 W — critical threshold
input_number.unknown_draw_duration_trigger     ← default 3 min — sustained draw before alert fires
input_datetime.unknown_draw_last_alert         ← cooldown stamp; checked against 30-min window
automation.unknown_draw_warning                ← dual-branch (critical/warning); fires when
                                                 orchestrator in [conserve, critical, loadshedding*]
                                                 + unknown_load_power_W > warning_threshold + 30-min cooldown
```

### Three-Zone Inside Arming (S2 2026-05-17; S9h 2026-05-20)
```
binary_sensor.security_inside_garage_motion   ← raw garage zone motion (cam05 + DSC hook)
binary_sensor.security_inside_main_motion     ← raw main-house zone motion (cam14 + DSC hook)
binary_sensor.security_inside_bedrooms_motion ← raw bedroom passage zone motion (cam15 + DSC hook)
binary_sensor.security_inside_house_motion    ← backward-compat union of the three (updated S2)

# Arming gates — binary_sensor.inside_*_armed = auto OR manual (either arms the zone)
binary_sensor.inside_garage_armed    ← auto OR manual
binary_sensor.inside_main_armed      ← auto OR manual
binary_sensor.inside_bedrooms_armed  ← auto OR manual (bedrooms: nobody home only)

# Auto booleans — written by security_inside_cameras_arming automation
input_boolean.inside_garage_armed    ← auto: ON when nobody home (or bedtime for lounge/garage)
input_boolean.inside_main_armed      ← auto: ON when nobody home (or bedtime)
input_boolean.inside_bedrooms_armed  ← auto: ON when nobody home AND NOT dogs_inside

# Manual booleans — dashboard override only, never auto-set
input_boolean.inside_garage_armed_manual    ← manual override / DSC stub (S6+)
input_boolean.inside_main_armed_manual      ← manual override / DSC stub
input_boolean.inside_bedrooms_armed_manual  ← manual override / DSC stub
```

### Dogs / Occupancy Suppression Booleans (S9h — 2026-05-20)
```
input_boolean.security_dogs_out  ← suppress rear/pool alarm during garden walk (10min auto-off)
input_boolean.dogs_inside         ← suppress inside camera alerts when dogs home alone
                                    (manual — set before leaving, clear on return, OR tap
                                    DOGS_INSIDE_ON/DOGS_INSIDE_OFF on the departure prompt —
                                    genuinely two-way as of BUG-S68 2026-07-17, see below)
```

### Security Outdoor Corroboration (S11 — 2026-05-22)
```
binary_sensor.security_external_motion_recent  ← ON for 5min after any perimeter/grounds
                                                   camera fires. Used by RUNG 8 to require
                                                   outdoor path corroboration.
                                                   Defined in security_logic.yaml.
binary_sensor.security_front_approach_recent   ← ON for 5min after a FRONT-APPROACH camera
                                                   fires (ipcam01/02, ipcam03, cam04, cam07,
                                                   cam09, ipcam05). Excludes cam12+ipcam04
                                                   (pond/pool — rear NVR cameras that fire
                                                   for animals). Used by RUNG 2.5 instead of
                                                   ext_recent. Added S16 2026-06-14.
```

### Security Classifier Presence Signals (S2 — 2026-05-17; S8 logic updated 2026-05-19)
```
binary_sensor.family_arriving    ← someone in home zone NOT in 60s snapshot (snapshot-delta, S8)
                                   CHANGED from recency-only (BUG-S26 fix)
binary_sensor.family_departing   ← someone Away who WAS in 60s snapshot (snapshot-delta, S8)
binary_sensor.all_family_home    ← ALL family APs in home zones (visitor disambiguation — BUG-P13)
```

### Per-Zone Snapshot Image Helpers (added S8 — 2026-05-19)
```
input_text.security_image_perimeter_front  ← last snapshot from ipcam01/02 (street cameras)
input_text.security_image_perimeter_rear   ← last snapshot from ipcam05 (rear boundary)
input_text.security_image_grounds_front    ← last snapshot from ipcam03/cam04/cam07 (front grounds)
input_text.security_image_grounds_rear     ← last snapshot from ipcam04/cam09/cam12 (rear grounds)
input_text.security_image_inside_garage    ← last snapshot from cam05 (garage interior)
input_text.security_image_inside_main      ← last snapshot from cam14 (lounge)
input_text.security_image_inside_bedrooms  ← last snapshot from cam15 (passage)
input_text.security_image_arrival_locked   ← locked at Stage 1 T+5s; Stage 2 reads this (BUG-S41 fix)
# Router reads zone-matched slot; fallback to security_last_motion_image if slot empty
```

### Presence Snapshot Helpers (added S8 — 2026-05-19)
```
input_text.who_was_home_snapshot  ← rolling 60s "Ryan,Vicky" list; feeds family_arriving/departing
input_text.arrival_who_was_home   ← point-in-time snapshot at Stage 1 arrival; Stage 2 delta reads
input_text.departure_who_was_home ← point-in-time snapshot at Stage 1 departure; Stage 2 delta reads
input_datetime.last_critical_event ← 90s cooldown gate for critical_intrusion branch (BUG-S39 fix)
```

### UI-Managed Utility Meters (DO NOT recreate in YAML)
```
sensor.solar_to_battery_energy_today  ← utility_meter, created 2026-03-20
sensor.grid_to_battery_energy_today   ← utility_meter, created 2026-03-20
```

### Water Core Entities
```
sensor.water_state
sensor.water_tank_level            ← friendly name "Water Tank Level %" — entity_id has % stripped
sensor.water_tank_depth_validated
switch.borehole_pump
input_boolean.water_refill_cycle_active
input_boolean.water_refill_aborted_due_to_safety   ← can be stuck ON after Tuya reconnect false-aborts
binary_sensor.water_refill_allowed
input_boolean.water_alert_notify   ← suppress water alert pipeline
input_boolean.water_refill_force_override ← bypass solar window/SOC/last-sun gates; safety hard-stops still apply (added 2026-06-14)

# Daily usage analytics (added E7 2026-06-14):
sensor.water_tank_consumption_integral  ← platform:integration, m, cumulative; resets on restart
sensor.water_usage_today                ← utility_meter daily cycle; resets midnight ⚠️ HA restart to activate
sensor.water_daily_usage_mean           ← statistics mean, 7d — ~50% of true daily mean (ramp bias); ×2 for calibrated value
sensor.water_effective_fill_target      ← demand_target normally; water_target_depth_full (1.6m) when predictive+conserve
```

### Water Demand Planning Entities (added 2026-05-25)
```
sensor.water_target_depth_tomorrow             ← stop depth for tonight's refill; reads tomorrow's select
  attributes: demand_type, tomorrow_day
sensor.water_demand_today                      ← today's demand type string (dashboard)
sensor.water_refill_blocked_reason             ← updated: now compares against water_target_depth_tomorrow

# Day demand selectors — one per day, options: Normal | Wash/Clean | Irrigation | Pool
input_select.water_demand_monday
input_select.water_demand_tuesday
input_select.water_demand_wednesday
input_select.water_demand_thursday
input_select.water_demand_friday
input_select.water_demand_saturday
input_select.water_demand_sunday

# Target depth input_numbers (entity_ids unchanged; initials + names changed):
input_number.water_target_depth_normal         ← 1.00m (was 1.55m "Normal Day")
input_number.water_target_depth_partial        ← 1.20m "Wash/Clean" (was 1.65m "Partial Irrigation")
input_number.water_target_depth_irrigation     ← 1.25m NEW (Irrigation days)
input_number.water_target_depth_full           ← 1.60m "Pool Day" (was 1.85m "Full Irrigation")

# Season preset scripts
script.water_demand_set_summer_profile
script.water_demand_set_winter_profile

# Safety hard-stop automation (added 2026-05-25 — water_safety.yaml)
# automation id: water_safety_battery_hard_stop
# Triggers on SOC < water_battery_soc_hard_stop; no safety-state exemption

# Daily target stop automation (added 2026-05-25 — water_tank_refill_control.yaml)
# automation id: water_stop_at_daily_target
# Stops pump when depth reaches sensor.water_target_depth_tomorrow

# REMOVED 2026-05-25 — 21 demand planning input_booleans (were defined but never read by any automation):
# irrigation_full_monday/tuesday/wednesday/thursday/friday/saturday/sunday
# irrigation_partial_monday/tuesday/wednesday/thursday/friday/saturday/sunday
# washing_heavy_monday, washing_heavy_thursday
# house_cleaning_monday, house_cleaning_thursday
# washing_partial_friday, washing_partial_saturday, washing_partial_sunday
```

### Smart Cleaning (Vacuum) Entities (added 2026-08-31, V8)
```
vacuum.deebot_t80s_biesie                          ← the device's own entity, canonical
sensor.deebot_t80s_biesie_error                    ← fault code source, message always live-read not hardcoded
sensor.deebot_t80s_biesie_battery
image.deebot_t80s_biesie_map                       ← was broken 2026-08-29/30 (upstream), self-resolved 2026-08-31
binary_sensor.deebot_alert_active                  ← ours — fault OR low-lifespan (V11), NOT the raw error sensor
sensor.deebot_alert_context                        ← aggregator feed, base "deebot" resolves to alert.deebot_alert
alert.deebot_alert                                 ← needs 1 HA restart to activate, see SMART_CLEANING_CONTRACT.md
sensor.vacuum_water_refill_estimate
sensor.vacuum_dirty_water_estimate
sensor.vacuum_detergent_level
input_button.vacuum_log_water_refill
input_button.vacuum_log_dirty_water_empty
input_button.vacuum_log_detergent_new_bottle
input_button.vacuum_log_manual_clean                ← added 2026-09-02, manual roller/debris clean tracker (Section 3e)
sensor.vacuum_manual_clean_estimate
# Full entity registry + pipeline: docs/domains/SMART_CLEANING_CONTRACT.md
```

### Gas Bottles Entities (added 2026-09-02)
```
input_select.gas_stove_bottle_identity              ← Owned/Swap/None, which bottle feeds the stove — NOT a stock count
input_select.gas_heater_bottle_identity              ← same, heater — "None" most of the year, not an error state
input_select.gas_owned_bottle_status                 ← per-CYLINDER status, independent of which appliance it's on
input_select.gas_swap_bottle_status                  ← don't confuse with the two *_bottle_identity selects above
sensor.gas_stove_days_remaining                      ← main dashboard figure most of the year (stove = main use)
sensor.gas_heater_days_remaining                     ← only meaningful once gas_heater_bottle_identity ≠ "None"
input_number.gas_avg_days_per_bottle_stove           ← seed 75d, flagged guess — separate EMA from heater's
input_number.gas_avg_days_per_bottle_heater          ← seed 21d, flagged guess — NOT the same clock as stove's
input_boolean.gas_do_refill / gas_do_swap            ← TRANSIENT — read once by gas_confirm_completed then reset
                                                        off after logging; do not treat as persistent state
input_button.gas_confirm_completed                    ← the real completion event, works with or without an order
sensor.gas_transaction_log                            ← event-triggered (gas_transaction_logged), NOT triggered off
                                                        input_button.gas_confirm_completed's own state change — see
                                                        UTILITIES_CONTRACT.md Section 8e for why (race avoidance)
sensor.gas_gauge_history                              ← Claude-maintained via /log-gas-reading, trend-only, does
                                                        NOT feed the avg-days EMAs above
alert.gas_alert                                       ← needs 1 HA restart to activate, not yet done as of 2026-09-02
# Full design + pipeline: docs/domains/UTILITIES_CONTRACT.md Section 8
```

---

## 🔴 DO NOT TOUCH Without Full Review

- Security event router automation
- Notification central scripts (`script.notify_*_event`)
- Water lifecycle capture start/end automations
- Presence resolver boundary logic
- Power core aggregation sensors

---

## 📋 Domain States (Post-Audit 2026-04-16)

> **⚠️ This table is a historical changelog, not a live status board — read the domain
> contracts (`docs/domains/*_CONTRACT.md`) for current state.** Rows are dated
> individually (per their own "Updated" tag) and some haven't been touched since April,
> even though the underlying domain kept changing for months after. All 9 non-Power
> domain contracts plus POWER_CONTRACT.md got a fresh deep-drift sweep 2026-08-21 (see
> OPEN TODO entries above) — those are current as of that date; this table is not, and
> wasn't updated as part of that sweep (out of scope — this table narrates history, the
> contracts state current fact). Two concrete stale claims found and fixed here in a
> follow-up pass the same day: Alerts' file count (14→16) and Lighting's "SOC energy
> saving trigger remains future work" (shipped 2026-06-19, confirmed in the same
> follow-up). Garden has no row at all — see GARDEN_CONTRACT.md.

| Domain | Status | Notes (as of 2026-04-16) |
|---|---|---|
| 🛡️ Security | S16 + dogs notification 2026-06-14 | **S16 (2026-06-14):** BUG-S48 — arrival image shows cam07 kitchen instead of ipcam03 driveway; fixed: Stage 1 arrival lock now reads `ipcam03_driveway_history` (per-camera, ipcam03-only) preferentially over shared `security_image_grounds_front` slot (cam04+cam07 share it). BUG-S49 — departure confirmed time was T+3.5min not actual departure; fixed: `depart_time` captured at Stage 1 T=0, passed in `habiesie_departure_detected` event data as `left_at`, Stage 2 captures before its delay and uses in message. BUG-S50 — RUNG 2.5 false critical at night with `home:all`; root: cam12 (back pond NVR) fires for frogs/moonlight → ext_recent ON for 5min → cam14 catches any light change → critical_intrusion; fixed: new `binary_sensor.security_front_approach_recent` (5min delay_off, excludes cam12+ipcam04) added to security_logic.yaml; RUNG 2.5 uses this instead of ext_recent; ext_recent retained for RUNG 8. BUG-S51 — inside alerts when Luke home; root: Luke AP drops briefly → anyone_connected_home OFF (2min) → passage arms → cam12+cam15 → RUNG 8; fixed: anyone_connected_home delay_off increased 2min→5min in presence_core.yaml. cam14 zone reduction (monitor light area) applied camera-side by user — helped reduce genuine NVR motion from screen light changes (not fixable in code). **2026-05-28 (ipcam02 BUG-S28 re-attempt):** Camera fully restarted, reconfigured with Smart Events (Intrusion/Line Crossing/Region Entrance), deleted from HA, and re-added. Still only `scenechangedetection` discovered — AcuSense sensors (fielddetection/linedetection/regionentrance) not appearing. BUG-S28 remains open. `ipcam02_street_driveway_down_motion_valid` now includes scenechangedetection as fallback trigger (committed 2026-05-27). **S15 (2026-05-27):** BUG-S47 — stale image in critical_intrusion/visitor notifications fixed: RUNG 2.5 snapshot now 4s delayed after motion capture; visitor reads zone-matched image slot (not global fallback). RUNG 5 three-way split: `perimeter_front` (ipcam01/02 without gate approach), `visitor` (entrance_valid OR gate_activity), `gate_activity` (gate opened + street motion without entrance_valid). ipcam03 AcuSense dog misclassification noted (dogs trigger intrusion zone → false visitor). **S14 (2026-05-26):** BUG-S44 — go2rtc reconnect replays cam14/cam15 as false motion; fixed: delay_on 3s added to cam14_lounge_motion_valid + cam15_passage_motion_valid. BUG-S46 — RUNG 8 fires critical_intrusion during family arrival (AP transition race); fixed: `not arriving` guard added. BUG-S45 partial — ipcam04 alarm fires camera-side independently of HA; rest_command.ipcam04_deactivate_alarm added; dogs_out_cancel_alarm automation added. **S11 (2026-05-22):** anyone_connected_home delay_off 2min added (false intruder on departure). RUNG 7 + 8d: `not departing` guard. `binary_sensor.security_external_motion_recent` (5min delay_off) — outdoor corroboration sensor. RUNG 2.5 + RUNG 8: require `ext_recent OR ip_cam OR high_conf` — NVR inside cameras alone no longer escalate to critical without outdoor precedent. zone_label: 'grounds front'/'grounds rear' split (trigger camera based). Router reason/zone: from trigger.to_state.attributes (not stale live re-evaluation). Visitor: 45s grace removed → immediate notification, 30s cooldown. **S10 (2026-05-21):** BUG-S36 — departure misclassified as arrival fixed (ipcam01 recency in Stage 1 cam_dir: entrance_valid = arrival only if street camera fired ≤120s ago). Mutual 120s suppression added to Stage 1 (second sensor of a traversal blocked). RUNG 4 (service_person): `perim` removed — perimeter always active regardless of staff schedule. RUNG 5 (visitor): `not staff` removed — visitor fires at gate even when maid on site. Router service_person: always silent logbook (no push notification). Stage 2 departure: Unknown suppressed when staff_on_site. Zone display: human-readable names. **S9h (2026-05-20):** dogs_inside boolean (suppresses interior alerts when dogs home alone). Auto/manual arming split — arming schedule writes `inside_*_armed` (auto); `_manual` variants dashboard-only. binary_sensor.inside_*_armed = auto OR manual. dogs_out timer 20→10min. **S9g (2026-05-20):** Auto-arm S2 classifier booleans with capture booleans in arming schedule. Nobody home → `inside_main_armed_manual` + `inside_bedrooms_armed_manual` + `inside_garage_armed_manual` all ON → RUNG 8 fires critical_intrusion for inside cameras → critical push notification. Dog caveat: use guest_mode when dogs home alone. **S9f (2026-05-20):** threat_level rule 1 false critical fixed — `inside AND (nobody OR night)` changed to `inside AND nobody` (rule 1) plus `inside + stay-mode` (rule 1b). Cam15 at night with family home no longer triggers critical HA dashboard alert. **S9e (2026-05-20):** Stay-mode lounge intrusion (RUNG 2.5) — after full bedtime, lounge fires → critical_intrusion → push notification. Camera health false alert for cam14/cam15 fixed — staleness only flagged when someone is home (indoor cameras legitimately quiet when empty). Passage arming confirmed correct (nobody_home only). **S9d (2026-05-20):** Reload filter on event router (unknown→X transition suppressed). Inside-only fallback rung (cam15 reload flicker → family_movement not perimeter_threat). Double visitor cooldown race fixed (last_visitor_event set before delay). **S9c (2026-05-20):** Gate-open-alone false perimeter_threat fixed (RUNG 8c: gate+no-motion→idle). Arrival Stage 2 "Unknown" fixed (arrival_who_was_home now uses 60s rolling snapshot, not T=0 AP state which included the arriving phone). Visitor grace period 20s→45s + stronger suppression (gate-age + ipcam03 entrance check). **S9b (2026-05-20):** Stage 1 delay 90s→5s (camera zones non-overlapping, direction from trigger entity). Per-ipcam latest snapshot files added (ipcam01-05, `/local/ipcamXX_latest.jpg`). Visitor grace period 20s (gate-open check prevents false visitor before family arrival). ipcam01/ipcam03 camera configs documented and updated. Street-down camera recommendations documented. HA human/vehicle separation limitation documented. **S8 (2026-05-19): BUG-S21–S27 structural fixes.** ipcam05 (rear boundary) was triggering visitor classification (BUG-S21 — RUNG 5 now front-perimeter only). Wrong-camera image in notifications fixed via 7 per-zone image helpers + zone-aware router lookup (BUG-S22/S23). Stage 2 arrival/departure now uses delta logic: Stage 1 snapshots who was home before the event; Stage 2 computes who newly arrived/departed (BUG-S24/S25). family_arriving/departing rewritten from recency-only to snapshot-delta — AP roaming no longer triggers false arrival (BUG-S26). Visitor branch cooldown 60s added (BUG-S27). zone_label attribute now distinguishes perimeter front vs perimeter rear. New entities: 7 `input_text.security_image_*` per-zone helpers (security_helpers.yaml); `input_text.who_was_home_snapshot`, `arrival_who_was_home`, `departure_who_was_home` (presence_helpers.yaml); automation `presence_snapshot_who_home` (presence_boundary.yaml). ipcam02 confirmed dead (BUG-S28 — firmware V5.8.13 H13U incompatible with hikvision_next Smart Events; perimeter_front = ipcam01 only in practice). Pool alarm tiered 2026-05-10. **Notifications silent since 2026-04-17 — FIXED 2026-05-08.** Root cause 1: `security_event_router` had no `elevated` branch — single-camera daytime events (score 25–74) were silently dropped. Root cause 2: cam07 excluded from confidence front tier — `security_intruder_active` never fired for cam07-only events. Both fixed. Camera fleet restructured: cam06 removed (uninstalled); cam01/cam10 deprecated; cam05 moved to inside garage (grounds_front zone); all 5 IP cameras fully wired (ipcam01–ipcam05); perimeter_rear now active for first time (ipcam05). AcuSense entrance/exit debounce sensors added. Visitor trigger corrected to street cameras only (ipcam01/02). Arrival uses ipcam03_entrance_valid as primary. Entity naming standardised: all IP cameras now use `ipcam` prefix (was mixed camip/ipcam). 2026-05-10: `security_pool_alarm_trigger` tiered threat gate added — arms from 20:00 (not waiting for night_mode ~22:30); critical always fires; warning suppressed when family asleep (night_mode ON + home). |
| ⚡ Power | Updated 2026-06-18 | **Dashboard session (2026-06-14):** Operations dashboard restructured. Power Control view replaced (3 columns: Orchestrator state mushroom + live inputs + thresholds; Appliance controls — geyser/pool/borehole/energy-saving; Inverter sync + season + load-shedding). Geyser Control: new dedicated view (mushroom + 4 themed markdown cards for window/temperature/sports-night/manual-run; read-only status entities; controls entities; run button; logbook). Inverter Control: new view (Inv1 programme 6-slot; Inv2 sync card; load-shedding level scenes; solar modes; orchestrator context; diagnostics — today/totals/battery/grid/solar PV/health/areas; solcast). Appliance Control: pool_winter_start_hour added to Pool Controls; SOC Target Summer/Winter and Last Sun Slot Start labels added. Correction: `sensor.load_shedding_next_start` does not exist — area sensor used instead. Scene names: `stage_2_load_shedding` / `stage_4_load_shedding` / `stage_6_load_shedding_priority_charge`. Dashboard changes are in .storage (git-ignored) — requires HA core restart to reload, NOT just browser refresh. **E8 periodic fix (2026-06-14):** inverter_programme_controller periodic trigger reduced /5→/30 min; `if trigger.id != 'periodic'` guard added to Battery First and Load First notification branches — periodic re-enforcement is now silent (prevents notification floods when inverter reverts between polls). **P6 solar statistics (2026-06-14):** New diagnostic entities feeding E8 simulator: `sensor.solar_production_vs_capacity` (PV % of rated capacity, solar_clipping.yaml); `sensor.solar_vs_forecast_ratio_today` (actual vs Solcast today, solar_state.yaml); `sensor.inverter_production_7d_stdev`, `sensor.house_load_7d_mean`, `sensor.solar_forecast_accuracy_7d` (power_statistics.yaml); solar_weather_correlation updated to 4-state; `sensor.solar_season_efficiency_factor`; `input_number.system_solar_capacity_w` + 4 season efficiency input_numbers (power_helpers.yaml). Recorder excludes updated (configuration.yaml). | **Session E1 (2026-06-14):** Energy Orchestrator core implementation. sensor.energy_orchestrator_state rebuilt: old 5-state basic sensor replaced with 6-state priority ladder (loadshedding_critical/loadshedding/critical/conserve/surplus/normal). sensor.orchestrator_decision_reason added. 12 new input helpers (orchestrator_enabled + 9 SOC/threshold input_numbers + pool_winter_start_hour + inverter_sync_status). binary_sensor.water_refill_allowed updated: orchestrator gate added — blocks refill when state in [critical, loadshedding, loadshedding_critical]. Inverter sync: automation.inverter_sync_check + script.force_inverter_sync added (partial — energy_pattern only; other entities unconfirmed). Spec correction: battery_state_health has no 'warning' state — conserve branch uses 'low' only. Degraded SOC floor parameterised: input_number.orchestrator_conserve_degraded_soc_threshold (60%). ha core check passed. Template reload + helper reload completed — functional check: state=normal, SOC 64%, battery healthy. | **2026-05-27:** BUG fix — pyscript sync_power_groups.py was wiping group.flexible_power_loads and group.critical_power_loads to [] on every startup, overriding YAML definitions. Removed the two wipe lines; pyscript now only maintains real_power_loads (auto-detected). YAML in power_templates.yaml is now the sole owner of flexible/critical groups. Plug change: Water Cooler Plug → LG Combo Washer Plug (2026-05-25); sensor.lg_combo_washer_plug_power added to known_power_loads, flexible_power_loads, house_laundry_power_sensors. Issue 16 in POWER_CONTRACT resolved. **Session H (2026-05-25):** Eskom outage triggered BMS protection — pool pump drew ~3.9kW at 07:25 (grid offline, SOC ~38%), causing BMS shutdown and house trip at 07:45. Fixed: (1) pool pump now blocked during grid outage when SOC < `grid_offline_soc_min_pool` (60%, Branch 2a) — bypasses minimum run time; (2) pool pump now blocked in last-sun-slot (14:00–16:00) when SOC < `sensor.last_sun_soc_target` (80%/90% by season, Branch 2b); (3) same two conditions added as gates to Branch 5 turn-on; (4) new triggers: `group.inverter_grid` state change + `sensor.inverter_battery_soc` below threshold. Borehole: `binary_sensor.water_refill_allowed` updated with last-sun-slot gate (SOC below target → block, but bypassed if tank ≤20%). New shared helpers added to power_helpers.yaml (9 new input_numbers). New template sensors: `sensor.last_sun_soc_target` (80%/90% season-aware), `sensor.geyser_target_run_hours_today` (2/3.5h season-aware). Geyser: helpers added but automation NOT implemented — see POWER_CONTRACT Sprint 5 for design questions. Session G (2026-04-28): pool_pump_solar_control two bugs fixed — (1) no 16:00 hard-stop: pump ran until 18:05 when manually stopped; fixed by adding "16:00:00" trigger (id: end_of_day) + Branch 0 unconditional turn-off, removing before:"16:00:00" from global condition, moving it to Branch 5 turn-on only; (2) pool_pump_last_on never updated: mode:single caused Branch 1 to be silently dropped when Branch 5 turned pump on, leaving pool_pump_continuous_run_minutes stale at ~18h; fixed by changing to mode:queued. Session F (2026-04-22): power_statistics.yaml created — inverter_production_7d_mean/30d_mean, house_load_24h_mean (statistics platform), solar_vs_forecast_ratio_7d + solar_weather_correlation (template). All 5 excluded from recorder. pv3/pv4 broken voltage entries removed from 3 dashboards (operations/overview/testing). Broken suppress-counter entity cards removed from dashboard_operations. prepaid_balance_confidence | min(30) bug fixed (must wrap in list). Dashboard card last_changed guards added for pool_pump_control_status, pool_target_run_hours_today, pool_pump_continuous_run_minutes. pyscript sync_power_groups.py: hass.* proxy is fully restricted — hass.data, hass.services, homeassistant.helpers imports, and task.executor(pyscript_fn) all blocked. Final working solution uses state.names(domain="sensor") + service.call() builtins only; label-based flexible/critical groups left empty (entity registry inaccessible from pyscript). Session D (2026-04-22): pool_pump_solar_control added to power_automations.yaml; 4 old pool pump blocks removed from automations.yaml; pool helpers added to power_helpers.yaml; pool runtime sensors (history_stats + 2 templates) added to power_state.yaml; POWER_CONTRACT pool section added. Session A (2026-04-21): grid_to_house_power fixed; energy flow sensors corrected; battery_energy_available + average_night_consumption statistics sensor added; solar_forecast.yaml syntax error + 3× sensor.inverter_power fixed; grid_risk.yaml + alerts_system_health.yaml sensor.inverter_power fixed; power_core.yaml inverter_today_energy_import deprecation documented; pyscript event.fire fixed. Session B (2026-04-21): power_automations.yaml created; grid_status_monitoring + inverter_pwer_monitoring commented out (superseded by alerts_power.yaml); inverter_alert_battery_soc_critical migrated with ssa_*→ss_* fixes; alerts_power.yaml 3× inverter_power fixed; load_shedding migrated to new packages/load_shedding/ dir (⚠️ restart required); B4 sensor rename; B5 group ref fixed; B6 prog1_capacity fixed. Session C (2026-04-22): sync_power_groups.py startup trigger added + import rewritten (hass.data.get); battery_runtime.yaml 6× int default fixed; battery_energy_available device_class removed; number.inverter_battery_capacity storage template int default fixed. Pool pump automations researched (C3 — 4 blocks, lines 2222–3174, research only). |
| 💳 Prepaid | Clean | Drift history tracking added 2026-04-23. Confidence layer done. |
| 🚰 Water | Updated 2026-06-14 E7 | **2026-06-14 E7:** Daily usage analytics + predictive fill wired. (1) `sensor.water_tank_consumption_integral` (platform:integration, m) integrates `sensor.water_tank_consumption_rate` (m/h). (2) `sensor.water_usage_today` (utility_meter daily) — daily depth consumed, resets midnight ⚠️ HA restart. (3) `sensor.water_daily_usage_mean` (statistics, 7d) — rolling planning sensor; ~50% of true daily mean due to utility_meter ramp; multiply ×2 for calibrated estimate. (4) `sensor.water_effective_fill_target` (template) — returns demand_target normally; returns water_target_depth_full (1.6m) when predictive fill ON + orchestrator=conserve. (5) `water_stop_at_daily_target` updated to trigger on `sensor.water_effective_fill_target`. (6) Branch 4.7 predictive fill added — fires when `water_predictive_fill_enabled=on` + `orchestrator=conserve` + `level < threshold%` + `depth < effective_target`. Default 50% threshold is below all demand targets — set to 70-80% for effective proactive fill. **2026-06-14 E6:** Two blocking bugs fixed + force override added. (1) BUG Issue 14 FIXED — `water_block_refill_when_not_allowed` had no safety-state exemption; safety-state refills looped endlessly (Branch 1 start → block kill → repeat). Fixed: added NOT(safety) + NOT(critical+grid) exemptions to block automation, matching existing mid_run_shutdown exemptions. (2) BUG Issue 15 FIXED — Branch 2 (critical+grid) incorrectly required `water_refill_allowed = on` (SOC ≥ 50%), violating contract (critical+grid = any SOC). At 2% tank fill with SOC=45% and grid on, nothing fired. Fixed: Branch 2 now checks `group.inverter_grid = on` + `load_control_borehole_enabled = on` directly; block automation exempts critical+grid starts. (3) `input_boolean.water_refill_force_override` added — bypasses solar window, SOC, and last-sun-slot gates; safety hard-stops (40% battery floor, 1.95m max depth) still apply; Branch 4.5 fires when override ON. Saturday demand selector changed to Pool for Friday-night prep fill (1.6m target). **2026-05-25 (session H — three commits):** (1) binary_sensor.water_refill_allowed updated — last-sun-slot gate added (borehole blocked if SOC below overnight target during 14:00+ window, unless tank ≤ 20% critical override). (2) water_borehole_mid_run_shutdown automation added — stops pump mid-cycle if water_refill_allowed goes OFF or solar window closes; safety state exempt; does NOT set safety abort flag. (3) water_safety_battery_hard_stop automation added in water_safety.yaml — stops pump + sets safety abort flag when SOC < water_battery_soc_hard_stop (40% temp, lower to 20% after new batteries). Issue 6 FULLY RESOLVED. (4) Demand-based refill targets: 21 per-day demand booleans removed (were unused); replaced by 7 input_select per day (Mon–Sun, options: Normal/Wash/Clean/Irrigation/Pool); sensor.water_target_depth_tomorrow and sensor.water_demand_today added; Case 4 refill trigger now fires when depth < tomorrow's target (not just state=low); water_stop_at_daily_target automation added — pump stops at demand target instead of always 1.95m. (5) Thresholds corrected: critical 0.50→0.25m, minimum_safety 0.60→0.35m, low_trigger 1.00→0.80m, target_depth_normal 1.55→1.00m, target_depth_partial (Wash/Clean) 1.65→1.20m, target_depth_full (Pool Day) 1.85→1.60m; water_target_depth_irrigation added (1.25m). Season preset scripts: water_demand_set_summer_profile / water_demand_set_winter_profile. |
| 🔔 Alerts | Updated 2026-05-27 (stale — see ALERTS_CONTRACT.md for current state, re-audited 2026-08-21) | alert.camera_health added 2026-04-29 (alerts/ = **16 files** as of 2026-08-21, doc-drift corrected — was 14, then grew further with alerts_device_batteries.yaml). **2026-05-27:** alerts_batteries.yaml added (15th file) — full battery alert pipeline for Honor 10 Dash + Honor X7 Dash. Pipeline: per-device low + overcharge binary sensors → dash_battery_alert_context → alert.dash_battery_alert → aggregator. Screen brightness management added in packages/admin/tablets.yaml (night dim, away dim, morning/arrival restore). ⚠️ Requires HA restart (alert: entity). |
| 🔔 Notifications | Scripts correct | All C-series bypasses resolved. BUG-N02 counter entity correct. |
| 🧭 Presence | Alert pipeline live | Unknown AP alert + occupancy anomaly implemented. Trust chain intact. |
| 💡 Lighting | Stable | All L01–L20 fixed. BUG-L11–L14 (2026-06-14): morning_wake noon ceiling; arrival cooldown always-blocked fix; nobody-home front security light added; wrong garage entity (stw_3gang→garage_light) in all 3 arrival scenarios. M1/M2/M3 implemented. ~~SOC-based energy saving trigger remains future work (power session).~~ **✅ Done — doc-drift correction 2026-08-21:** `energy_saving_mode_auto_enable`/`_auto_disable` shipped in `power_automations.yaml` 2026-06-19 (confirmed live during this session's LIGHTING_CONTRACT.md sweep). **BUG-L20 (2026-08-29):** an out-of-band copy of `lighting_arrival_night.yaml` built on a pre-2026-06-14 base re-introduced BUG-L03/L12/L13/L14/L15 at once; caught before reload and all re-fixed — ⚠️ `automation.reload` still owed. |
| 🌐 Network | Updated 2026-05-28 | **2026-05-28:** sensor.ups_accessories_power (sum of USB1/2/3 + TypeC + DC out) and sensor.ups_visibility_score (accessories / total × 100) added to network_ups.yaml — mirrors load_visibility_score pattern from power domain. **2026-05-27:** network_ups.yaml added — EcoFlow River Pro UPS monitoring (packages/network/). Sensors: ups_on_battery, ups_runtime_seconds/friendly/eta, ups_status_card, ups_runtime_severity, ups_load_percent/status, ups_load_markdown. Helpers: 6 input_numbers + 1 input_boolean. Automations: AC Always On enforcement, on-battery notify, grid restore notify, battery warning/critical, load warning. All alerts via script.notify_power_event. | BUG-NET01 fixed 2026-04-28: unifi_cpu_5m_max availability → has_value(unifi_gateway_cpu_utilization). BUG-NET02 fixed 2026-04-28: unifi_memory_5m_max availability was self-referencing → has_value(unifi_gateway_memory_utilization). BUG-NET03 fixed 2026-04-28: packet loss removed from wan_health_score (ping_sum_5min is latency sum not pass count); score now uses latency + jitter only. BUG-NET04 fixed 2026-04-21. All verified live in network_helpers.yaml. |
| 🏗️ Context | All fixed | BUG-CTX01 fixed 2026-04-30: context_presence.yaml → presence/presence_trust.yaml. BUG-CTX02 fixed 2026-04-28: context_schedules.yaml deleted, bedtime_mode in lighting_helpers. BUG-CTX03 fixed 2026-04-30: home_context now derives from security_nobody_home + night_confirmed, no longer imports sensor.security_mode from security/. |
| 🔧 Infra | All fixed 2026-04-28 | BUG-CORE01 fixed (ha_events_per_second removed). BUG-INF01 fixed (printer_cartridge_low dangling }}). BUG-BKP01 fixed (github.yaml routed through notify_system_event). BUG-WEA01 confirmed already fixed. |

---

## 🚀 Verified Priority Work Queue

### Group V — Ecovacs Deebot T80S Omni Integration Buildout (opened 2026-08-29 — scenario/schedule planned 2026-08-30, no HA-side code yet)
```
[x] V1. packages/integrations/vacuum.yaml created 2026-08-30 with the mat-removal
        reminder (see V9 below): input_datetime.vacuum_reminder_time_workday/
        _weekend, input_boolean.vacuum_mat_reminder_enabled, automation.
        vacuum_mat_removal_reminder_workday/_weekend (note: entity_id derives
        from `alias`, not `id` — confirmed live via Supervisor API, `ha core
        check` valid, all reloaded live). V3/V4 (alert/notification wiring),
        listed here as still unbuilt at the time, were both completed
        2026-08-31 — see below, not stale.
        **Doc gap closed 2026-09-02:** the "full 31-entity inventory" was
        never actually written anywhere — `vacuum.yaml`'s header pointed to
        PROJECT_STATE.md's Known Integration Issues (which never had the
        list, only the map-bug row) and SMART_CLEANING_CONTRACT.md Section 4
        pointed back to `vacuum.yaml` (which never had it either) — a
        circular pointer neither side actually satisfied. Fixed by pulling
        the real registry live from `.storage/core.entity_registry`
        (`platform: ecovacs`) and writing the full 31-row table into
        SMART_CLEANING_CONTRACT.md Section 4, both cross-pointers corrected
        to point there. **Also caught a real inaccuracy while there**: the
        domain breakdown the contract had stated (10 sensor) was wrong —
        actually 13 sensor (missing `area_cleaned`, `total_area_cleaned`,
        `total_cleaning_duration` from the count), corrected in place.
        CLAUDE.md's package table row and this file's Document Index row
        for SMART_CLEANING_CONTRACT.md were both already correct (added by
        V14) — nothing to change there.

[x] V2. Mapping finished 2026-08-30. Stable room layout confirmed:
        Bedroom Main, Bedroom Luke, Bedroom Tayla, Bathroom Main, Bathroom kids,
        Kitchen, Corridor, Living room, Reading room (map "Biesie Main House", in use).
        Scenario/schedule design done same day — see session note below and the
        dated entry in OPEN TODO. Three app scenarios: Daily Clean (Living room +
        Kitchen + Reading room + Corridor, daily 09:00), Bedroom Clean (Bedroom Main/
        Luke/Tayla, Mon/Wed/Fri 10:30), Bathroom Clean (Bathroom Main/kids,
        Tue/Thu/Sat 10:30). Corridor bundled into Daily as the high-traffic
        connector into the bedroom wing — user can move it to ride with
        Bedroom/Bathroom instead if preferred. Duration numbers unverified — see
        V1 note above; check sensor.deebot_t80s_biesie_cleaning_duration /
        _total_cleaning_duration / _area_cleaned after first real runs of each
        scenario and tighten the 10:30 gap if Daily overruns on mop days.

[x] V3. Fault alerting done 2026-08-31 (user asked directly — "why am I not
        getting priority alerts... same pattern as normal alerts with cancel
        alert etc"). Built in packages/integrations/vacuum.yaml, copied
        verbatim from alerts_batteries.yaml's canonical pipeline:
        binary_sensor.deebot_alert_active (sensor.deebot_t80s_biesie_error
        not in [0,unknown,unavailable], delay_on 20s — the error sensor
        bounces through unknown/unavailable/0 on every integration reload,
        confirmed live) → sensor.deebot_alert_context (severity: critical
        when vacuum.state=='error' i.e. genuinely halted/stuck, else warning)
        → automation.route_deebot_alert (real delivery — script.
        notify_system_event, NOT the alert: entity's own notifiers:
        STD_Alerts, which alerts_batteries.yaml's own header already
        documents as broken/zero-delivery) with a Cancel Alert action button
        (mobile + Telegram) wired to input_boolean.deebot_alert_snoozed,
        auto-reset when the fault clears. `alert.deebot_alert` also added for
        aggregator/dashboard visibility — **needs one HA restart to
        activate** (alert: entities can't be reloaded, same restriction
        documented in alerts_batteries.yaml); real notification delivery
        does NOT depend on it and is already live. Naming follows the
        "sensor.<base>_alert_context" / "alert.<base>_alert" convention
        exactly, so sensor.alert_device_entities (alerts_summary.yaml)
        picks this up automatically — zero changes needed to the aggregator
        itself, confirmed by reading its resolver logic (base="deebot" →
        tries alert.deebot then alert.deebot_alert). **Real trigger found
        same day, not synthetic**: error code 103 (WheelAbnormal — wheel
        jammed, confirmed via web search against trunetto.com's Deebot 103
        troubleshooting page) fired twice 2026-08-31, code 323 fired three
        times (meaning unconfirmed — not in the community deebot_client
        error-code tables checked; codes 301-319 in that table are the
        water-tank range, e.g. 305="Dirty Water Tank is full", so 323 is
        plausibly a newer Omni-station water/dock code not yet catalogued,
        but this is a guess, not confirmed — the new pipeline captures the
        live description text automatically next time it fires, closing
        this gap with real data instead of more guessing). Consumable
        lifespan warnings (the other half of this item) NOT done — only
        fault/error alerting was built; low-lifespan thresholds still open,
        genuinely low priority (all three sensors reading 92-97% as of
        today) — split into new V11 below so V3 doesn't stay ambiguously
        half-open.

[x] V4. Done 2026-08-31 — NOT built around event.deebot_t80s_biesie_last_job:
        checked 3 days of real history first (dozens of completed jobs since
        2026-08-29) and it has never fired once — same class of gap as the
        getMapSet bug, don't rely on it without re-checking first. Also NOT
        a per-segment started/completed push — Auto Clean runs many short
        cleaning→docked room segments per session (15+ seen in one morning
        live), so per-segment notifications would be pure spam. Built
        instead: one daily summary push, triggered when vacuum.
        deebot_t80s_biesie has been docked continuously for 10 min (session
        genuinely over, not just between rooms) AND at least one job ran
        that day (gated on a midnight snapshot of the integration's own
        lifetime counters via input_number.vacuum_total_cleans/area/
        duration_at_midnight, reset daily at 00:01 by
        automation.vacuum_daily_snapshot_reset). Message: "Cleaned X m² in
        Y min across Z jobs today." "Stuck" is intentionally NOT duplicated
        here — already covered by V3's fault alert on vacuum.state=='error'.
        Seeded today's midnight snapshot manually via
        `automation.trigger` immediately after building this (today's
        00:01 had already passed) so the first real summary isn't inflated
        by lifetime totals.

[~] V5. User input 2026-08-31: "v5 not needed - though haven't run at night
        when lounge/passage would fire". Read as: not worth building a
        suppression fix proactively, but explicitly NOT closed as
        verified-fine either — the real risk window (a night Auto Clean run
        near cam14/cam15's lounge/passage coverage) genuinely hasn't
        happened yet, all runs so far have been daytime. Left untested,
        deliberately not marked [x]. Revisit if/when a night run actually
        happens — check SECURITY_CONTRACT.md security_event_classification
        for a false RUNG fire before assuming it's fine.

[x] V6. Scheduling decision made 2026-08-30: fixed time-of-day schedule set
        natively in the Ecovacs app scheduler (see V2's schedule table) — NOT
        HA-driven, NOT solar-gated. HA will not start/stop/pause cleans. Chosen
        times (09:00 / 10:30, nothing after ~11:00, nothing Sunday afternoon)
        were picked by hand to satisfy the user's stated constraint — don't run
        once everyone's home for the evening (after 6/7pm) and not Sunday
        afternoons — so no HA presence-gating automation is needed either.
        Still open/optional: check the Deebot's actual charging/running power
        draw and decide whether it's material enough for
        group.flexible_power_loads / known_load_power (Session E7 pattern) —
        low priority, revisit only if it shows up as a meaningful unknown-draw
        contributor.
        **UPDATE 2026-08-30 evening:** user is now trialling whole-house
        "Auto Clean" (AI) at 06:00 workday / 08:00 weekend via the app's own
        Schedule screen instead — the 3-scenario schedule above is currently
        unscheduled (still valid as a fallback, just not active). Trade-off
        flagged and accepted: bedrooms/bathrooms clean daily now, not every
        2nd day. Re-open this item if the trial gets walked back to the
        per-room split.
        **UPDATE 2026-08-31:** the "still open/optional" power-draw question above
        got its first real number, DB-derived (see `POWER_SYSTEM_PERFORMANCE_LOG.md`'s
        2026-08-31 entry): the dock's brush-drying+charging cycle (user-confirmed
        2-3hr per dock) shows a sustained ~400-600W `house_load_power` uplift for
        at least the first ~70min of a cycle, isolated on an evening with no
        cooking/airfryer running — full-cycle figure still unconfirmed, a later
        evening's automation (geyser) confounded the rest of that read. **User
        decision: buy a dedicated smart power plug to monitor the dock directly**
        (real per-device wattage instead of inferring from whole-house load deltas)
        — **hardware not yet purchased**, no plug bought/installed yet. Once it
        exists: add it to this package following the airfryer's pattern
        (`sensor.philips_airfryer_plug_power` in `office`/relevant package — check
        naming convention there) and only then decide `group.flexible_power_loads`
        inclusion with real data instead of the DB-inferred estimate above.
        **UPDATE 2026-09-02:** natural A/B found — a mop-free (vacuum-only) day gave
        a clean charge-only dock window (72min, airfryer+geyser both confirmed idle
        via their own sensors) with **no clear sustained load uplift** over the
        household's ordinary idle range, unlike 08-31's drying-cycle window. Points
        at the drying/heater element as the dominant draw, not charging itself — see
        `POWER_SYSTEM_PERFORMANCE_LOG.md`'s 2026-09-02 entry. Still DB-inference, not
        a substitute for the smart plug (still not purchased).

[x] V7. Done 2026-08-30 — pushed live via the Lovelace WebSocket API
        (`lovelace/config` + `lovelace/config/save`, Supervisor-token auth,
        no HA restart — same no-restart technique as the 2026-06-14 dashboard
        re-enable session, this time scripted directly rather than ad hoc).
        **Operations → new "Vacuum" view** (path `vacuum`, matches the house's
        `type: sections` / `ios-dark-mode-dark-blue-alternative` theme
        convention, footer navbar copied verbatim from the `office` view):
        mushroom-vacuum-card (start/pause/stop/return_home/locate +
        battery/fan-speed built in), live map (image.deebot_t80s_biesie_map),
        today's-activity + lifetime-totals markdown, conditional error banner,
        consumables (brush/filter lifespan %, mop-attached), controls
        (work_mode/water_flow/active_map/clean_count/toggles/relocate),
        logbook-card activity feed, and a manually-maintained schedule note
        card (HA can't read the app's Schedule screen — this needs hand-
        updating if the app schedule changes again, per the V6 update above).
        **Home dashboard**: compact status block inserted into section 0
        (heading badges + mushroom-template-card, tap → Operations/vacuum),
        conditional error chip, matches the existing "Network Health"-style
        block pattern. Both pushes verified live afterward by re-fetching the
        dashboard configs. `.storage/lovelace_dashboards` unaffected — no new
        dashboard registered, both are edits to existing dashboards, so
        nothing to add to CLAUDE.md's package table for this.
        **Fixed same evening, user caught it:** two gaps found and corrected
        in a follow-up push. (1) Consumables reset-button row only had 2 of 3
        buttons (missing Reset Side Brush) — added. (2) All three reset
        buttons (`button.deebot_t80s_biesie_reset_main_brush_lifespan` /
        `_filter_lifespan` / `_side_brush_lifespan`) were `disabled_by:
        integration` in the entity registry — not in the state machine at
        all, so the two that *were* on the card wouldn't have worked. Enabled
        all three via `config/entity_registry/update` over the same
        WebSocket connection; HA auto-reloaded the `ecovacs` config entry
        after the reported 30s `reload_delay`; confirmed live (state
        `unknown`, the normal never-pressed button state) before re-pushing
        the card. (3) Separately, user reported the map picture wasn't
        rendering — traced via `system_log/list` over the same WebSocket to a
        real upstream bug, not a card/config issue: `deebot_client`'s
        `getMapSet` (and `getCleanInfo`) calls have failed with `code: 20003,
        msg: "rcp not support"` on this device continuously since the
        integration was set up (2026-08-29 12:20 through now) — confirmed via
        web search this is a known `deebot_client`/HA-core limitation on
        several newer Omni models (T50/T90 Pro Omni, N10 MAX+), with an open
        discussion specifically requesting T80S Omni support
        (home-assistant/core issue #173158). **Not fixable from this repo** —
        left the picture-entity card in place (self-resolves if upstream
        ships support) with a markdown caveat explaining why it's broken and
        linking the issue, instead of hiding the gap or leaving an
        unexplained broken image.
        **Second follow-up fix, same evening, user caught it again:** (4) the
        Activity card used `custom:logbook-card` (the HACS card) with an
        `entities:` list — wrong config shape for that specific custom card,
        rendered as a visible "Configuration error — Please define an
        entity." card. Fixed by switching to the native HA `type: logbook`
        card instead (same `entities:`/`hours_to_show` shape works there) —
        copied verbatim from how `lighting_arrival_night`'s dashboard cards
        already use it successfully elsewhere in this house, rather than
        guessing at the custom card's real schema. (5) Both sections were
        rendering 2-wide instead of 3 — none of the vertical-stack cards had
        `grid_options` set, so they fell back to a default width. Fixed by
        giving every vertical-stack `grid_options: {columns: 4, rows: auto}`
        — confirmed from other views in this same dashboard that the sections
        grid is a 12-unit-per-row system (columns:12 = full width, columns:6
        = half, so columns:4 = a third → 3 per row). Also reorganized section
        1 from 2 uneven stacks into 3 even ones (status+controls / map+caveat
        / activity+error+logbook) to actually fill the 3-column layout
        instead of leaving a gap. Both fixes verified live via a fresh
        `lovelace/config` fetch after saving.
        **Third follow-up fix, same evening — user caught a real misread of
        the sections model:** (5) kept the 2 separate `type: grid` sections
        from the previous fix — this was wrong. In HA's Sections dashboards,
        each `sections[]` entry is its own bounded, independently-drag-
        handled box that claims one of the view's `max_columns` slots; it
        does NOT stretch to fill the row on its own. With only 2 sections
        defined and `max_columns: 4`, HA rendered 2 narrow ~25%-width boxes
        side by side (each then cramming its 3 `columns:4` cards into that
        already-narrow box — hence the earlier screenshot's truncated labels
        like "M...98.28%") plus two empty add-card placeholders filling the
        rest of the row. **Fix:** collapsed both sections into a single
        `sections[]` entry holding all 6 vertical-stacks, each still
        `grid_options: {columns: 4}` — with only one section, it expands to
        the view's full width and the 6 cards wrap into 2 true rows of 3,
        aligned across the whole page. `max_columns` dropped 4→3 to match
        (now describes this one section, not a phantom 4-slot row). Verified
        live via a fresh `lovelace/config` fetch: 1 section, 6 cards, all
        `columns: 4`. **Lesson for next dashboard session:** `sections[]` ⇒
        side-by-side boxed groups (use `max_columns` to control how many);
        `grid_options.columns` (12-unit) ⇒ card width *within* one section.
        Don't reach for multiple sections to build a single aligned N-column
        layout — that's what grid_options is for, inside one section.

[x] V8. Done 2026-08-31 — new "Smart Cleaning (Vacuum) Entities" subsection
        added to Locked Entity Names below: the core device entity, the
        error/battery/map sensors, all 3 of our own alert-pipeline entities,
        the 3 water/detergent estimate sensors, and the 3 log buttons.
        Points to SMART_CLEANING_CONTRACT.md for the full registry rather
        than duplicating all ~45 entities here.

[x] V9. Mat-removal reminder — done 2026-08-30 as part of V1 (user asked for
        this directly: "reminder to remove mats from bathroom and main
        bedroom so it doesn't get stuck"). Two time-triggered automations
        (weekday/weekend, gated on input_boolean.vacuum_mat_reminder_enabled)
        fire 10 min before whichever Auto Clean time is currently live (see
        V6), via script.notify_system_event (continue_on_error: true, per
        CODING_STANDARDS). Trigger times are input_datetime helpers, not
        hardcoded — deliberately, since the app schedule already changed once
        in one session and HA can't read it directly. **If the app schedule
        changes again, these two helpers need updating to match** (and the
        Operations Vacuum view's schedule note card, and this checklist).

[~] V10. Tested live 2026-08-31, user supervising as agreed. Confirmed safe
         both attempts: vacuum.deebot_t80s_biesie's state/last_changed
         timestamp was unchanged after each call — the robot never moved.
         **Attempt 1**: called `vacuum.clean_area` (cleaning_area_id:
         ["kitchen"]) via REST — real 500, root cause pulled from HA's own
         error log: `ServiceValidationError: Area mapping is not configured
         for vacuum.deebot_t80s_biesie. Configure the segment-to-area
         mapping before using this action`. Checked
         `.storage/core.entity_registry` — confirmed genuinely unset.
         **Attempt 2 (user: "use the picture of the map i shared for
         mapping")**: found the actual mechanism via HA core source
         (fetched `homeassistant/components/vacuum/__init__.py` +
         `websocket.py` + `ecovacs/vacuum.py` from GitHub rather than
         guessing) — the mapping lives at entity registry
         `options.vacuum.area_mapping` (`dict[area_id, list[segment_id]]`,
         `DOMAIN` key confirmed from source), and the exact same
         `vacuum/get_segments` WebSocket command HA's own "Configure areas"
         dialog uses is callable directly (admin-only, confirmed working —
         no error). Called it for real: **`{"segments": []}`** — the device
         is reporting zero rooms to HA right now. Not a dead end, not a
         guessing failure — this is a second, separate gap in what the
         `ecovacs` integration receives from the device (room-boundary data
         via a `RoomsEvent` the integration listens for, distinct from the
         map image itself, which does render now). Deliberately did NOT
         invent/guess segment IDs to write into `area_mapping` — they're
         device-generated (`f"{map_id}_{room.id}"` per source), a photo
         can't substitute for HA actually receiving them, and a wrong write
         risks the robot going to an unintended real room. **Next step**:
         re-run `vacuum/get_segments` after the vacuum's next fully
         completed job — worth checking without being asked again, same as
         the map bug this may just need a real clean cycle to populate.
         `vacuum.send_command` still untested, lower priority. No camera
         entity exists for this device at all — confirmed not available.

[x] V11. Done 2026-08-31, later same session — user said "go with rest".
         Folded into V3's existing pipeline rather than a parallel one:
         binary_sensor.deebot_alert_active now also fires when
         min(main_brush, side_brush, filter) < input_number.
         vacuum_lifespan_warning_threshold (15%, adjustable), and
         sensor.deebot_alert_context's severity logic + devices attribute
         extended to cover it — warning only, never critical (a worn brush
         doesn't halt the robot, unlike a fault code with vacuum.state==
         'error'). route_deebot_alert's message and the alert: entity's own
         message both changed to build from sensor.deebot_alert_context's
         devices attribute generically, since a trigger can now be a fault
         code, a lifespan warning, or in principle both at once — no longer
         assumes it's always an error code. Validated + reloaded live,
         confirmed sensor.deebot_alert_context reads normal with real
         current lifespans (all 92-97%).

[x] V12. Water/dirty-water/detergent consumption estimator — done 2026-08-31,
         user request after a full day running whole-house Auto Clean:
         "have had to empty dirty water twice and refill water once... add
         water levels or timing against when errors raised... add estimate
         for when to purchase new detergent." No sensor exists for any of
         clean-water level, dirty-water level, or detergent level (confirmed
         — not in the 31-entity registry, not something the integration
         tracks at all, same category of gap as the earlier map/water
         findings). Built entirely on manual logging since there's no other
         signal: 3 input_button entities
         (vacuum_log_water_refill/_dirty_water_empty/_detergent_new_bottle)
         the user presses when they physically do each thing, each
         snapshotting sensor.deebot_t80s_biesie_total_area_cleaned and the
         current time. Two self-calibrating EMA pairs (avg area per event,
         avg days per event — new = old×0.6 + latest×0.4, same smoothing
         weight used elsewhere in this repo) drive
         sensor.vacuum_water_refill_estimate / _dirty_water_estimate (days
         until next needed). Detergent is the one fully deterministic piece
         — user gave the exact ratio (3L tank, 1:200 mix = 15mL/refill = 15
         cap-fulls, matches their own math exactly): 1L bottle ÷ 15mL =
         66 refills/bottle, so sensor.vacuum_detergent_level counts down
         1.5%/refill with zero calibration needed, plus a one-shot low-
         detergent notification below 15%. All seed values (190 m²/refill,
         95 m²/dirty-empty, 1.0/0.5 days) are explicitly flagged in the YAML
         as guesses from the single real day of data available (2026-08-30)
         — **expect rough estimates for 1-2 weeks**, same caveat this repo
         already applies to geyser_last_heat_up_minutes-style trend capture.
         Dashboard: new "Water & Detergent" card with the 3 estimates + the
         3 log buttons + an explanatory note, added to the Vacuum view.
         **UPDATE 2026-08-31 (same day, follow-up ask):** low-detergent
         notification now carries a "Detergent Bought" action button (phone
         + Telegram), mirroring the fault pipeline's Cancel Alert pattern —
         but instead of only silencing, tapping it presses
         input_button.vacuum_log_detergent_new_bottle (reuses
         vacuum_log_detergent_new_bottle's existing reset logic rather than
         duplicating it), since buying detergent means a new bottle is now
         in use. New automation:
         vacuum_detergent_bought_from_notification. Validated + reloaded
         live, confirmed `on`.

[x] V13. Water/Detergent dashboard upgraded to colour-threshold markdown
         cards — user request 2026-08-31 ("markdown cards seem to work well
         for this with colour thresholds"). Replaced the plain entities-list
         card with 4 markdown cards in a 2×2 layout: Clean Water and Dirty
         Water (red/orange/green on the days-remaining estimate — ≤0
         Overdue, <1 Due soon, else OK), Detergent (red/orange/green on %
         remaining — <15/<30/else), and a 4th "Cleaned" card (last job +
         lifetime totals) since "how much vacuum cleaned" was part of the
         same ask and fits naturally alongside the water context. No
         existing colour-threshold-markdown precedent found elsewhere in
         this dashboard to copy — used inline HTML `<span style="color:...">`
         (HA's markdown card renders raw HTML), colour hex chosen to read
         clearly in both themes (#e53935 red / #fb8c00 orange / #43a047
         green). Verified the Jinja renders correctly via the HA template
         API before pushing (`/api/template`). Confirmed live with real
         data: both water estimates showed **red "Overdue"** at push time
         (344 m² mopped since either was last logged — genuinely true,
         nobody had pressed the log buttons since V12 shipped), a good
         real-world proof the thresholds work, not just a happy-path test.

[x] V14. VACUUM_CONTRACT.md created 2026-08-31 — user asked directly ("did we
         add this as a new contract or maybe add to home appliance
         contract?"); answer was no, nothing existed. Followed
         GARDEN_CONTRACT.md's exact structure (8 sections: Overview, File
         Inventory, Pipeline Architecture, Entity Registry, Pipeline Audit,
         Must NOT, Known Issues, Future Scope) as the precedent for a
         single-device domain getting its own contract rather than folding
         into INFRA_CONTRACT.md. Added to PROJECT_STATE.md's Document Index
         and CLAUDE.md's domain-contracts list — also fixed a pre-existing
         gap found while there: GARDEN_CONTRACT.md itself was never added to
         CLAUDE.md's list despite existing since 2026-04-29. CLAUDE.md's
         package table row for `integrations/` updated 1 file → 2 files.
         **Renamed same day** VACUUM_CONTRACT.md → SMART_CLEANING_CONTRACT.md
         (user: "so if add something else it has a home") — `git mv`,
         updated CLAUDE.md's two references + this file's Document Index +
         the contract's own header/Section 1 wording to name the file's
         actual scope ("Smart Cleaning", vacuum-specific content unchanged)
         and note how a second device should be added (new section, not a
         new file) if one ever shows up.

[x] V15. Power spike investigation — done 2026-08-31, user asked ("pushed
         grid import from about 3kw to 7 today... will get power monitor
         plug to make sure"). **Checked with real data before agreeing —
         the vacuum was NOT the cause.** At the exact peak
         (14:24:44, sensor.house_grid_power 8100W), sensor.house_load_power
         (actual house consumption) was only 200-2600W the whole window —
         normal. sensor.house_battery_power was -6000 to -6377W (charging)
         with inverter_battery_soc climbing 90.0%→91.0% live, and
         select.inverter_1/2_program_4_charging = "Grid" confirms this was
         the documented P4 Grid Charge automation (14:00-17:00 window,
         Session E5 in the changelog) doing exactly what it's designed to
         do — topping the battery from grid. Cross-checked the vacuum too:
         at the exact spike moments it was paused/docked following an
         error, not mid-clean. **Conclusion: nothing wrong, unrelated to
         the vacuum, no action needed on the power side.** User's plan to
         get a power-monitor smart plug is still worthwhile for its stated
         purpose though — dashboard status (cleaning/drying/charging)
         inferred from real power draw, same technique as
         binary_sensor.geyser_at_temperature — and would also close V6's
         still-open "check actual power draw for load-visibility" sub-item.
         Not done yet — waiting on the user to actually get the plug.
```

### Group A — Trust Model Chain (Fixes security + lighting + door alerts)
```
✅ A1. Restore input_datetime.low_trust_start/end — SUPERSEDED / NOT NEEDED.
✅ A2. boundary_permissive_window — CORRECTION: 2026-04-15 session only confirmed
       maid_on_site+weekday==0 logic (partial). Full rebuild to use
       binary_sensor.low_trust_present OR guest_mode completed 2026-05-17 (S1.3).
✅ A3. Replace input_boolean.low_trust_present → binary_sensor.low_trust_present
✅ A4. Same fix in lighting_departure.yaml
```
**Group A genuinely complete as of 2026-05-17. PROJECT_STATE previously recorded A2 as done but the 2026-04-15 fix was incomplete — boundary_permissive_window only fired on Monday maid days, not for gardener/contractor/guest_mode. S1.3 applied the full fix.**

### Group B — Alert Pipeline Gaps
```
✅ B1. Implement alerts_presence.yaml — done 2026-04-16
      binary_sensor.unknown_ap_alert_active (5 min delay)
      binary_sensor.occupancy_anomaly_alert_active (15 min delay, info only)
      sensor.presence_alert_context (warning/critical by AP duration)
      alert.presence_alert — STD_Alerts, repeat 60/120 min
B2. Implement alerts_security.yaml — done 2026-04-14 (already complete)
✅ B3. Add network_alert + media_alert to aggregator trigger list — done 2026-04-14
```

### Group C — Notification Bypasses (~36 min)
```
✅ C1. Add continue_on_error: true to Telegram in notify_water_events.yaml (BUG-N01, done 2026-04-14)
✅ C2. Fix counter: water_borehole_faults_week → water_borehole_faults_this_week (done 2026-04-15, water_notifications.yaml:158)
✅ C3. Add input_boolean.notifications_enabled to notifications_helpers.yaml (BUG-N03, done 2026-04-15)
✅ C4. Migrate alerts_temperature.yaml to script.notify_system_event (BUG-N04, done 2026-04-15)
✅ C5. Migrate presence_notifications.yaml to script.notify_presence_event (BUG-N05, done 2026-04-15)
✅ C6. Resolve dual notification in alerts_device_power.yaml (BUG-A04/N06, already clean — verified 2026-04-15)
✅ C7. Add | default('') guard to escaped_title/escaped_message in all 6 scripts (BUG-N08, done 2026-04-13)
✅ C8. Complete MarkdownV2 escape chain — add . and ! to all 6 scripts (BUG-N09, done 2026-04-13)
```

### Group D — Security Quick Fixes ✅ Done 2026-04-15
```
✅ D1. Fix typo: security_poor_visibility → security_visibility_poor
✅ D2. Fix typo: security_low_light_weather → security_weather_low_light
✅ D3. Fix cam06 last event reads cam09 (copy-paste, 1 line)
✅ D4. Fix cam10 image key: cam10_pool_bar_motion_images → cam10_pool_bar_images
✅ D5. Add unique_id to sensor.security_correlation
✅ D6. Remove presence_test_arrival test automation from production
```

### Group E — Water Trigger Integrity ✅ Done 2026-04-15
```
✅ E1. water_borehole_no_rise_protection: added from: "off" to pump ON trigger
✅ E1. water_reconcile_cycle_state: added from: "on" to pump OFF trigger
✅ E2. water_refill_visibility_guard: added for: "00:00:10" stability window
✅ E3. water_refill_allowed verified: checked on critical+low paths; safety/emergency paths intentionally bypass
```

### Group I — Infra / Network / Context Bugs (Audited 2026-04-16 — see INFRA_CONTRACT.md, NETWORK_CONTRACT.md, CONTEXT_CONTRACT.md)
```
✅ BUG-INF01 [HIGH]:   Fix binary_sensor.printer_cartridge_low — extra }} on trailing line
                        Fixed 2026-04-28: Removed dangling }} from state: > block

✅ BUG-NET03 [HIGH]:   WAN packet loss formula was mathematically wrong (ping_sum_5min is latency sum, not pass count)
                        Fixed 2026-04-28: Removed packet loss from health score — score now uses latency + jitter only

✅ BUG-NET01 [MED]:    sensor.unifi_cpu_5m_max availability references non-existent sensor.unifi_cpu
                        Fixed 2026-04-28: Changed to has_value('sensor.unifi_gateway_cpu_utilization')

✅ BUG-NET02 [MED]:    sensor.unifi_memory_5m_max availability was self-referencing (circular)
                        Fixed 2026-04-28: Changed to has_value('sensor.unifi_gateway_memory_utilization')

✅ BUG-BKP01 [MED]:    backup/github.yaml direct Telegram call bypassed quiet hours
                        Fixed 2026-04-28: Failure routes through script.notify_system_event (warning severity — bypasses
                        quiet hours so 5AM failures still alert). Success notification removed (5AM noise).

✅ BUG-WEA01 [LOW]:    weather_api_stale_alert already had full action block — checkbox not updated

✅ BUG-CORE01 [LOW]:   sensor.ha_events_per_second returned total sensor count, not a rate
                        Fixed 2026-04-28: Removed sensor from ha_monitoring.yaml (real delta-based
                        rate needs sensor.recorder_events which isn't available)

✅ IMP-IDS01 [MED]:    IDS Hyyp alarm stub created 2026-04-29: packages/security/security_alarm.yaml
                        Search of automations.yaml found ZERO IDS/hyyp references — integration not yet wired.
                        Stub documents provisional entity interface + migration instructions for when IDS goes live.

✅ BUG-CTX02 [LOW]:    context_schedules.yaml was stub
                        Fixed 2026-04-28: bedtime_mode moved to lighting_helpers.yaml (7 lighting consumers);
                        bedtime_time (unused) deleted; context_schedules.yaml deleted
```

### Group T — Telegram Enhancement (Decided 2026-04-20 — replaces defunct notify.whatsapp)
```
✅ T1. [HIGH] Security events → camera snapshot on Telegram
        Fixed 2026-04-28: telegram_bot.send_photo added after send_message in
        notify_security_events.yaml, guarded by img is not none. Uses existing img URL
        (https://ha.dunners.tech/local/security_latest.jpg). continue_on_error: true.

✅ T2. [MEDIUM] Inline keyboard acknowledgment for critical alerts
        Fixed 2026-04-28: inline_keyboard added to Telegram critical messages in
        notify_security_events.yaml (/ack_security_alert → alert.security_alert off) and
        notify_power_event.yaml (/ack_power_alert → alert.power_alert off).
        Callback automations added to each file.

✅ T3. [LOW] Suppress Telegram sound for information-level notifications
        Fixed 2026-04-28: disable_notification: "{{ sev == 'information' }}" added to
        Telegram data block in all 6 notify_*_event.yaml scripts.

✅ T4. [LOW] HA mobile app alarm sound for critical alerts
        Fixed 2026-04-28: push.interruption-level: critical (iOS) + channel: alarm +
        ttl: 0 + priority: high (Android) added to all STD_Critical notify calls
        across all 6 notify_*_event.yaml scripts.
```

### Group L — Lighting Bugs (Audited 2026-04-16 — see LIGHTING_CONTRACT.md)
```
✅ BUG-L01 [HIGH]: entrance_down_lights:off in scene_night_away — already present, checkbox not updated
✅ BUG-L02 [HIGH]: main_entrance_light behaviour corrected 2026-04-28:
                   Original bug said "add :off to scene_night_away" but correct behaviour is the opposite —
                   main_entrance_light stays ON overnight as deterrence (same as boundary lights),
                   only turns off in morning routine. Fixed: removed from scene_night_away;
                   explicit switch.turn_off added to morning_wake_lights_on action.
✅ BUG-L01+L02: scene_morning_routine_off already applied after scene_night_away in departure — checkbox not updated
✅ BUG-L03 [HIGH]: arrival anyone_home → anyone_connected_home — already fixed, checkbox not updated
✅ BUG-L04 [MEDIUM]: time + cam14 triggers in morning routine — already present, checkbox not updated
✅ BUG-L05 [MEDIUM]: entrance_down_lights + dining + 30s cancel in kids bedtime — already present, checkbox not updated
✅ BUG-L06 [LOW]: from:"off" on departure trigger — already present, checkbox not updated
✅ BUG-L07 [LOW]: kids_bedtime_enabled had no consumers — removed from lighting_helpers.yaml 2026-04-28
✅ BUG-L08 [LOW]: patio_second_wake_time had no consumers — removed from lighting_helpers.yaml 2026-04-28
✅ BUG-L09 [LOW]: M1 + M3 implemented 2026-04-29. M2 helpers added. SOC trigger deferred to power session.
✅ BUG-L10 [MED]: kids_bedtime_week + kids_bedtime_weekend — added continue_on_error: true to both
      script.notify_lighting_event calls. Notification failure can no longer stop the scene from firing.
✅ BUG-L11 [HIGH]: morning_wake_lights_on had no upper-bound time gate — cam14 lounge motion fired
      the morning routine at 23:09 because now() >= morning_start is true all evening. Fixed 2026-06-14:
      condition now rejects any trigger at or after noon (today_at('12:00:00')). Both weekday and weekend
      paths updated. lighting_morning.yaml.
✅ BUG-L12 [HIGH]: arrival lighting always blocked via presence_boundary path — presence_boundary_resolver
      sets last_arrival_time=now() one step before arrival_detected=on; cooldown 0s>60=false always.
      Fixed 2026-06-14: cooldown condition removed. lighting_arrival_night.yaml.
✅ BUG-L13 [LOW]: nobody-home arrival scenario missing switch.front_house_security_light.
      Fixed 2026-06-14: added to Scenario 1 switch list. lighting_arrival_night.yaml.
✅ BUG-L14 [MED]: all 3 arrival scenarios used switch.stw_3gang_garage_switch_3 (wrong entity) instead
      of switch.garage_light. Fixed 2026-06-14: replaced across all 3 scenarios. lighting_arrival_night.yaml.
⚠️ BUG-L12 / L13 / L14 (above), plus BUG-L03 and BUG-L15, were ALL re-introduced at once on
      2026-08-29 by a stale-copy import of lighting_arrival_night.yaml built on a pre-2026-06-14
      base. Caught before reload and re-fixed — see BUG-L20 in LIGHTING_CONTRACT.md. Note
      switch.stw_3gang_garage_switch_3 has never existed in the entity registry: on that Sonoff
      3-gang (1002145922), _1 = switch.garage_light and _3 = switch.boundary_street_light.
      GAP ANALYSIS (2026-04-29):
      --- Entertainment mode ---
      • input_button.entertainment_mode_on exists (defined); input_boolean.entertainment_mode does NOT exist
      • lighting_entertainment.yaml is EMPTY — nothing responds to the button
      • scene.scene_entertainment_mode EXISTS in lighting_scenes.yaml (pool light + patio + entrance_down + dining) — scene is ready
      • CRITICAL: kids_bedtime_week + kids_bedtime_weekend both unconditionally turn off switch.pool_light_switch
        with NO entertainment mode guard — bedtime clobbers the scene mid-party
      PLAN: (1) add input_boolean.entertainment_mode to lighting_helpers.yaml
            (2) add button→boolean automation to lighting_entertainment.yaml (turn on button sets boolean ON; morning routine clears it)
            (3) add condition: entertainment_mode is OFF to both kids bedtime automations before pool_light_switch off
            (4) apply scene.scene_entertainment_mode when boolean turns ON; restore prior scene (or turn off) when it turns OFF
      --- Energy saving mode ---
      • input_button.energy_saving_mode_on + energy_saving_mode_off exist; input_boolean.energy_saving_mode does NOT exist
      • lighting_energy_saving.yaml is EMPTY
      • load_control.yaml already has load_control_geyser_enabled / pool_enabled / borehole_enabled with turn-off automations
      • energy_orchestrator_state sensor likely has states that should gate this (see POWER_CONTRACT.md)
      PLAN: (a) input_boolean.energy_saving_mode belongs in power_helpers.yaml (not lighting)
            (b) Power system sets it ON when battery SOC drops below a configurable threshold (power_automations.yaml)
              → also triggered by energy orchestrator hitting a critical state
            (c) Manual buttons (on/off) act as overrides (lighting_energy_saving.yaml handles them)
            (d) Lighting side checks binary: energy_saving_mode is ON → suppress non-essential lights (bar, patio, feature lights)
            (e) Morning routine or SOC recovery clears the boolean
      OWNER: power_automations.yaml owns the SOC-threshold trigger; lighting_energy_saving.yaml owns the lighting response
```

### Group M — Mode Features (Entertainment + Energy Saving) [PLANNED 2026-04-29]
```
[ ] M1. Entertainment mode — lighting side (lighting package)
        NOTE: input_boolean.entertaining_mode ALREADY EXISTS in context_presence.yaml.
              Button is input_button.entertainment_mode_on (name mismatch: "entertainment" vs "entertaining").
              DO NOT create a new input_boolean — wire the button to the EXISTING entertaining_mode boolean.
        a. Populate lighting_entertainment.yaml:
           - button→boolean: input_button.entertainment_mode_on → input_boolean.entertaining_mode ON
           - scene apply: entertaining_mode turns ON → scene.scene_entertainment_mode
           - scene restore: entertaining_mode turns OFF → revert (or apply scene_evening_routine)
           - morning clear: morning routine → input_boolean.entertaining_mode OFF
        b. Guard both kids_bedtime_week + kids_bedtime_weekend:
           - Add condition: input_boolean.entertaining_mode is OFF before pool_light_switch step
           NOTE: scene.scene_entertainment_mode already exists — no scene work needed

✅ M1. Entertainment mode — lighting side (lighting package) [2026-04-29]
        CORRECTION: Uses input_boolean.entertaining_mode (context_presence.yaml) — same concept,
        same boolean. input_boolean.entertainment_mode duplicate removed from lighting_helpers.yaml.
        a. lighting_entertainment.yaml populated (all wired to input_boolean.entertaining_mode):
           - Button → boolean on (entertainment_mode_button_on)
           - Boolean ON (from: "off") → scene.scene_entertainment_mode (entertainment_mode_scene_on)
           - Boolean OFF (from: "on") → scene.scene_evening_routine_on if civil_night is on (entertainment_mode_scene_off)
           - 06:00 daily clear → entertaining_mode OFF if currently on (entertainment_mode_daily_clear)
        b. kids_bedtime_week + kids_bedtime_weekend: entertaining_mode guard added — if ON, skip
           pool_light_switch in scene; turn off other 4 lights individually instead.

✅ M2. Energy saving mode — power side helpers + auto-trigger (power package) [2026-04-29; auto-trigger 2026-06-19]
        a. input_boolean.energy_saving_mode added to power_helpers.yaml
        b. input_number.energy_saving_soc_threshold (default 25%) + energy_saving_soc_recovery (default 40%)
        c. automation.energy_saving_mode_auto_enable (power_automations.yaml):
           triggers on SOC < threshold OR orchestrator in [critical, loadshedding, loadshedding_critical]
        d. automation.energy_saving_mode_auto_disable:
           clears when BOTH SOC > recovery AND orchestrator in [normal, surplus]
        Manual buttons still work as override at any time.

✅ M3. Energy saving mode — lighting side (lighting package) [2026-04-29]
        a. lighting_energy_saving.yaml populated:
           - Button on/off → boolean wiring automations
           - energy_saving_mode ON (from: "off") → turn off TIER 2 lights:
               switch.pool_light_switch, switch.pool_patio_down_lights, switch.back_house_security_light
           - logbook + notify (continue_on_error: true)
        b. Restoration NOT done here — presence/routine triggers handle it naturally.
        TIER 2 definition: pool light, pool patio, back house security (entertainment/comfort lights).
```

### Group F — Restart-Protection Guards ✅ Done 2026-04-13
Spurious fires on HA restart / template reload — missing `from:` / `not_from:` guards.
```
✅ F1. presence_notifications.yaml — Ryan/Vicky/Luke/Tayla unknown AP triggers: added from: "off"
✅ F2. admin_notifications.yaml — Unknown UniFi AP Detected: added from: "off"
✅ F3. water_notifications.yaml — water_tank_full_notification: added not_from: [unknown, unavailable]
✅ F4. water_notifications.yaml — water_refill_never_reached_full: added from: "off"
✅ F5. lighting_bar_presence.yaml — lighting_bar_presence (bar_occupied): added from: "off"
✅ F6. lighting_arrival_night.yaml — lighting_arrival_night (arrival_detected): added from: "off"
✅ F7. lighting_morning.yaml — morning_wake_wfh_cleanup (bedrooms_occupied): added from: "off"
✅ F8. energy_state.yaml — grid_charging_battery_while_solar_available: added not_from: [unknown, unavailable]
```
Rule: binary sensors / input_booleans → `from: "off"`. Template/state sensors → `not_from: [unknown, unavailable]`.

### Group U — iCloud Re-integration [DEFERRED — blocked on upstream fix]
```
⚠️ 2026-08-24: checked live against GitHub. Issue #155933 (the crash cited below,
"dict has no attribute user_info") IS fixed — merged via PR #156485, shipped HA
2025.11.3 (2025-11-17). This instance runs 2026.8.3, well past that fix. BUT the
actual reason this stays deferred — monthly 2FA reauth pain, non-primary Apple ID
support — is a separate, still-open, ongoing class of bugs through mid-2026, not
the same bug as #155933: #160536 ("stopped working after 2026.1"), #167608 (auth
failure), #170959 (filed May 2026 against HA 2026.5.2, still open as of this check
— Apple never sends the verification code needed to reauth, forcing a full delete-
and-recreate), plus a community report on 2026.5.4 with the same symptom, and a
third-party unofficial patched fork (github.com/mdeuerlein/homeassistant-icloud-fix)
created because the official integration still isn't reliable. Re-adding today would
likely work initially and then hit the same reauth wall again — correctly still
DEFERRED, just don't cite #155933 as the live blocker any more; it isn't.

BLOCKER (historical, resolved): icloud integration broken on HA 2025.11.0 (Issue
         #155933 — dict has no attribute user_info). Fixed HA 2025.11.3.
         Current live blocker is the reauth-reliability class of issues above —
         no single tracking issue, several open in parallel as of 2026-08-24.
         Do not start U1–U3 until integration is confirmed working on current HA version.

U1. [DEFERRED] Re-add iCloud integration via UI (Settings → Integrations → Apple iCloud)
    Use primary Apple ID account (non-primary family accounts unsupported upstream).
    Enable "with family" to expose all family device trackers.

U2. [DEFERRED] presence/presence_icloud.yaml — new file
    - Monitor device_tracker.* iCloud entities going unavailable (auth failure detection)
    - Auto-delete /config/.storage/icloud via shell_command on failure detection
    - Trigger HA restart via homeassistant.restart service
    - Send push notification (script.notify_presence_event, warning severity) with
      deep link to /config/integrations so re-auth is one tap away
    - Startup guard: suppress during system_startup window to avoid false triggers
    Note: 2FA code entry itself cannot be automated — human must approve on Apple device.
    This reduces monthly re-auth from manual diagnostic + terminal + restart to
    a single notification → tap → enter code flow.

✅ U3. [DONE — doc-drift correction 2026-08-21, and it didn't need to wait on U1/U2 at all]
    Apple device battery pipeline is live: `alerts/alerts_device_batteries.yaml`
    (2026-08-21, see ALERTS_CONTRACT.md "Device Battery Domain") covers 4 iPhones + 2
    Apple Watches in its initial rollout — a third option neither A nor B named
    (`alerts_device_batteries.yaml`, the new label-onboarded any-battery-device domain,
    not a dedicated Apple-only file). Turns out this never depended on the iCloud
    integration at all: the battery entities are HA Companion App (`mobile_app`)
    sensors — confirmed live in `core.entity_registry`, naming pattern
    `sensor.<person>_iphone*_battery_level` / `*_watch_battery_level` — which exist
    independent of, and unaffected by, the iCloud blocker below. U3 should never have
    been gated on U1/U2; leaving that framing corrected in case a future session reads
    this section literally.
```

---

## 📚 Document Index

*(Doc-drift correction 2026-08-21: this table's dates and a few descriptions were stale
— this file's own "Updated 2026-04-16" self-reference, Lighting's "9 open bugs" (all
fixed — see LIGHTING_CONTRACT.md, 0 open), and every domain contract's date, now that all
9 non-Power contracts plus POWER_CONTRACT.md got a deep-drift sweep this same day. Also
added GARDEN_CONTRACT.md, which had no row in this table at all.)*

| Document | Purpose | Authority |
|---|---|---|
| `docs/PROJECT_STATE.md` | This file — master context | ✅ Updated 2026-08-21 |
| `docs/CODING_STANDARDS.md` | Code rules and conventions | ✅ Current (updated 2026-08-21) |
| `docs/SESSION_STARTERS.md` | Copy-paste session prompts | ✅ Current |
| `docs/SYSTEM_CONTRACT.md` | Cross-domain dependency matrix + all bugs | ✅ Audit 2026-04-13 (partial — predates context/network/infra contracts; not re-audited 2026-08-21, see OPEN TODO for what was) |
| `docs/domains/SECURITY_CONTRACT.md` | Security — full audit, entity ref, checklist | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/POWER_CONTRACT.md` | Power — full audit, measurement model | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/WATER_CONTRACT.md` | Water — lifecycle contract verification | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/PRESENCE_CONTRACT.md` | Presence — trust model audit | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/ALERTS_CONTRACT.md` | Alerts — pipeline audit | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/NOTIFICATIONS_CONTRACT.md` | Notifications — bypass audit | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/LIGHTING_CONTRACT.md` | Lighting — full audit, scenes | ✅ Authoritative (deep-drift swept 2026-08-21; 0 open bugs, all L01–L19 fixed) |
| `docs/domains/NETWORK_CONTRACT.md` | Network/WAN — entity ref, all bugs fixed | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/CONTEXT_CONTRACT.md` | Context — night mode; CTX01/02/03 all resolved 2026-04-30 | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/INFRA_CONTRACT.md` | Infra — core/backup/office/weather/integrations/HACS | ✅ Authoritative (deep-drift swept 2026-08-21) |
| `docs/domains/GARDEN_CONTRACT.md` | Garden — pond pump alert pipeline | ✅ Authoritative (deep-drift swept 2026-08-21; was missing from this table entirely) |
| `docs/domains/SMART_CLEANING_CONTRACT.md` | Smart cleaning — currently just the Ecovacs Deebot (alerts, water/detergent estimator); named generically so a future 2nd device has a home | ✅ Created 2026-08-31 (also missing until user asked directly whether it existed), renamed from VACUUM_CONTRACT.md same day |
| `docs/domains/UTILITIES_CONTRACT.md` | Utilities — recurring consumable-delivery subsystems: Water Cooler (Section 2-7) + Gas Bottles (Section 8) | ✅ Authoritative (found missing from this table entirely during a 2026-09-02 `/update-docs` sweep — existed since 2026-08-31 with zero row here, same "missing entirely" pattern as GARDEN_CONTRACT.md and SMART_CLEANING_CONTRACT.md above; caught by step 3a, not by a user asking this time) |

> **The `*_CONTRACT.md` files are authoritative.** They were produced by reading actual config files.
> The older `*_CONTEXT.md` files are quick-reference summaries — use contracts for real work.

---

## ⚠️ Known Time Wasters (Avoid Repeating)

- Using `input_boolean.low_trust_present` in automations — always use `binary_sensor.`
- Storing structured data in `input_text` (255 char limit)
- Triggers without `from:` constraints (fires on unavailable→on)
- Calling `notify.*` directly instead of `script.notify_*_event`
- Editing `automations.yaml` — always use package files
- Building UI before backend is stable
- Missing `continue_on_error: true` on the **calling** `script.notify_*_event` action — a Telegram `ConnectTimeout` is an "Unexpected error" that propagates past `continue_on_error` inside the script and kills the caller. Always add `continue_on_error: true` to the calling action, not just inside the script.
- `custom:plotly-graph` cards: `statistic:"sum"` is the raw cumulative long-term-statistics column, NOT a period delta — never point it at a resetting sensor expecting a per-period total, use `statistic:"state"` on a sensor that itself resets on the target cadence instead. Also never put a static `layout.xaxis.range` in card config if buttons/pan/zoom need to persist — the card merges `config.layout` in twice, last, so it permanently overrides live interaction every render (`uirevision` does not help here). See POWER_CONTRACT.md's "`custom:plotly-graph` Card Gotchas" subsection (Section 16) for the full writeup — cost several restart cycles to root-cause, 2026-09-03.

---

## ⚠️ Known Integration Issues (Non-Config)

| Integration | Issue | Severity | Action |
|---|---|---|---|
| `hikvision_next` v1.1.1 | Sets invalid entity IDs with uppercase serial number (`DS-7116HGHI-F1...`). Deprecation warning — breaks in HA 2027.2 | Low | File upstream bug at maciej-or/hikvision_next |
| `bellows` (Zigbee) | `RESET_SOFTWARE: 11` on startup | None | Normal coordinator reset during init — ignore |
| `ids_hyyp` v1.9.0 | Zero automations in automations.yaml — integration not yet wired at HA level | Medium | Stub created (IMP-IDS01 ✅). Migrate entity interface when IDS is live. |
| `localtuya` v5.2.3 | False reconnect event can trigger spurious water tank abort | Medium | See WATER_CONTRACT.md — input_boolean.water_refill_aborted_due_to_safety can get stuck |
| Multiple weather integrations | OWM + OWM History + Met.no + Met Nowcast all installed — canonical source unclear | Low | Document which is used for what in INFRA_CONTRACT.md |
| `ecovacs` (built-in, Deebot T80S Omni) | ~~`deebot_client`'s `getMapSet`/`getCleanInfo` calls fail with `code: 20003, msg: "rcp not support"`~~ — **RESOLVED, confirmed live 2026-08-31 (user screenshot: full house map rendering with live room colors + position dot).** Was broken continuously 2026-08-29 12:20 through the 2026-08-30 evening session (confirmed via `system_log/list`); self-resolved with no config change on our side within ~24h — most likely a cloud-side/device-side fix given the timing, not a `deebot_client` version bump (versions weren't tracked precisely enough to confirm which). Caveat card removed from the dashboard 2026-08-31. Known upstream limitation class still real and affects other newer Omni models (T50/T90 Pro Omni, N10 MAX+) — home-assistant/core issue #173158 is the still-open discussion for T80S Omni specifically, kept here for reference in case it regresses. | Low (was Medium) | Closed. Re-open this row if the map breaks again — same diagnostic path (`system_log/list` over the WebSocket API) applies. |
| `icloud` (built-in) | **Removed end of 2025.** Monthly 2FA session expiry requires manual `.storage/icloud` delete + HA restart + code entry. Non-primary Apple ID (family members) unsupported. Broke entirely on HA 2025.11.0 (`dict has no attribute user_info`, Issue #155933 — **confirmed fixed live on GitHub 2026-08-24**, PR #156485, shipped HA 2025.11.3; this instance runs 2026.8.3, past the fix). Value lost: GPS away-from-home location for all family devices. ~~+ Apple device battery/charging state~~ — **not actually lost, doc-drift correction 2026-08-21:** battery/charging state comes from the HA Companion App (`mobile_app` integration), not iCloud — unaffected by this breakage, and `alerts_device_batteries.yaml` (2026-08-21) already covers it live. Only GPS/location is genuinely blocked on this integration. | High | **DEFERRED — confirmed still correct 2026-08-24.** The specific crash (#155933) is fixed, but re-checked live against GitHub and the underlying reauth pain is a separate, still-open, ongoing bug class through mid-2026 (#160536, #167608, #170959 — filed May 2026 against HA 2026.5.2, still open; community reports through 2026.5.4; an unofficial third-party patched fork exists because the official integration still isn't reliable). Re-add only once one of those settles — see Group U for the full citation trail. Recovery automation (U1/U2) still planned; the battery pipeline (U3) is done and was never actually blocked on this. |

---

*2026-06-16 session (security — dogs inside prompt suppression fix): `security_automations.yaml` dogs_inside_prompt `if` condition tightened. Three suppressions added/corrected: (1) `home_count <= 1` guard — `who_home_now` (AP snapshot captured at T=0 before departure) split and counted; prompt suppressed when 2+ family members home at trigger time (someone stays to supervise dogs). (2) `binary_sensor.staff_on_site` replaced with `binary_sensor.low_trust_present` — `staff_on_site` only covers maid + gardener; `low_trust_present` additionally covers `contractor_on_site`. (3) `input_boolean.entertaining_mode` added — suppresses prompt when guests are over for a party. Full suppression set is now: dogs_inside ON / guest_mode ON / entertaining_mode ON / low_trust_present ON / home_count ≥ 2. No new entities. SECURITY_CONTRACT.md dogs_inside_prompt suppression line updated.*

*2026-06-17 session (grid_risk_level false-critical fix): sensor.grid_risk_level was classifying as "critical" whenever battery_runtime_severity == "critical" (runtime < 3h to SOC floor), regardless of whether the grid was online. With 48kWh battery at 51% discharging to 40% floor at 2.35kW load, runtime = 2h17min → severity=critical → grid_risk_level=critical → dashboard showed "🔴 Grid required soon" even with grid actively providing 75W. Fix: added grid-online branch to grid_risk_level template. When group.inverter_grid = on: < 30min = warning, < 120min = moderate, else = safe. Critical reserved for genuine grid-offline scenarios. No new entities — grid_risk.yaml template condition change only. Reload Template Entities. POWER_CONTRACT grid_risk section updated.*

*2026-06-17 session (airfryer false-cut fix): airfryer_critical_cut automation fired falsely three times (2026-06-15, 06-16, 06-17 12:20) — logbook confirmed trigger was "numeric state of Inverter Battery SOC." Root cause: during inverter restart, Solarman poll gap, or battery swap/bypass, sensor.inverter_battery_soc reads 0%, satisfying SOC < critical_threshold (20%) while group.inverter_grid also shows "off" (inverters in bypass). Both conditions met momentarily → cut fired despite battery being healthy. Fix: added `condition numeric_state SOC above 5` to airfryer_critical_cut. BMS physically shuts down at 10%, so any reading ≤5% is a sensor glitch/startup artifact. Real cuts still fire at 6–20% during genuine grid-off + low-battery. power_automations.yaml changed only — Reload Automations.*

*2026-06-17 session (geyser adaptive evening + SOC recalibration): [Continuation of battery-swap session] (1) All SOC thresholds recalibrated from 30 kWh → 48 kWh bank keeping same absolute kWh targets: orchestrator emergency 15→12%, critical 25→20%, conserve 50→30%, pre_shed 80→65%, load_first 65→45%, conserve_degraded 60→40%; energy_saving threshold 25→20%, recovery 40→30%; grid_offline pool 60→40%, borehole 50→30%, geyser 45→25%, geyser_critical 35→20%; last_sun summer 80→70%, winter 90→80%; pool_pre_shed 80→55%; water_battery_soc_sufficient 50→30%. Applied via one-shot pyscript (pyscript/apply_48kwh_thresholds.py — deleted after use). (2) Geyser hard-off times shifted: normal winter 22:00→21:00, non-winter 21:30→20:30, sports winter 22:30→22:00, sports non-winter 22:00→21:30. (3) Geyser daily minimum check system added: input_boolean.geyser_reached_temp_today (set/reset by geyser_reached_temp_tracker), input_number.geyser_min_daily_energy_kwh (2.0 kWh), input_number.geyser_energy_at_morning_end, input_number.geyser_energy_at_midday_end (captured by geyser_period_energy_snapshot), sensor.geyser_daily_status (power_state.yaml — 4-state + per-period kWh attributes), automation.geyser_daily_minimum_check (20:00 daily — starts geyser if energy < min AND not at temp). (4) Adaptive evening start replaces fixed 19:30/20:00: input_number.geyser_adequate_daily_energy_by_midday (1.5 kWh threshold); 17:30 trigger fires if NOT at_temp AND midday energy < threshold (poor-day early start, accepts cooking-peak overlap); 18:30 trigger fires if NOT at_temp AND switch off (all-seasons fallback). (5) statistics sensors solar_forecast_accuracy_7d and inverter_production_7d_stdev fixed — entity_ids were wrong (7_day not 7d, std_dev not stdev); power_statistics.yaml names corrected; user renamed entity_ids in Settings → Entities. (6) Inverter programme SOC updated manually: P1 40→30%, P3 100→50%, P4 100→90%, P4 Grid→Disabled (HA manages P4 dynamically). force_inverter_sync run to push to INV2. POWER_CONTRACT geyser section fully rewritten. MANUAL DONE: all 16 input_number live values applied.*

*2026-06-17 session (battery swap — 2× Freedom Won Lite 15/13 → 3× Greenrich AF1600): Hardware replaced. New battery specs: 3× Greenrich AF1600, GFM052-314-N00-A, 51.2V nominal, 314Ah each, 16kWh nominal each, 16S1P cell config. Total bank: 3 × 314Ah × 51.2V = 48,230 Wh (48.2 kWh). YAML changes: (1) packages/power/battery_runtime.yaml — total_wh constant 31776 → 48230; header comment updated. (2) packages/power/energy_state.yaml — battery_energy_available sensor 31776 → 48230. (3) packages/power/power_helpers.yaml — battery_capacity_kwh initial 30 → 48; orchestrator_solar_gap_threshold initial 2 → 5 (P4 grid charge gap proportionally scaled). (4) packages/power/power_automations.yaml — P4 automation header comment updated. MANUAL ACTIONS REQUIRED: In HA UI or Developer Tools → Services, set input_number.battery_capacity_kwh to 48 and input_number.orchestrator_solar_gap_threshold to 5 (YAML initial only applies to new entities — stored values still reflect old settings). POWER_CONTRACT + PROJECT_STATE docs updated. Inverter screen settings: Lithium/CAN/BMS_Err_Stop; Batt capacity 900Ah (note: should be 942Ah = 3×314Ah); Charge 80A, Discharge 100A; Float/Absorption/Equalization all 55.0V (BMS controls via CAN — voltage params likely overridden); Shutdown 10% / Low Batt 15% / Restart 20%. System Mode: P1 01:00–05:00 40%; P2 05:00–09:00 40% Grid; P3 09:00–14:00 100%; P4 14:00–17:00 100% Grid (HA-managed dynamically); P5 17:00–21:00 40%; P6 21:00–01:00 40%. Consider reviewing P4 SOC target (currently 100% — may want 80–90% in winter). NOTE: ha core check + reloads required. NOTE: orchestrator SOC thresholds (emergency 15%, critical 25%, conserve 50%) may warrant recalibration for 48kWh bank — 50% conserve = 24kWh which is very conservative.*

*2026-06-15 session (geyser schedule + heat-up tracking): (1) Geyser morning schedule shifted — weekday winter 03:30 → 04:00, weekday non-winter 04:00 → 04:30 (summer needs less heating time; 03:30 was too early). geyser_automations.yaml: trigger times updated; geyser_morning_start_weekday initial updated 4.0 → 4.5 in power_helpers.yaml. ⚠️ ACTION REQUIRED: set input_number.geyser_morning_start_weekday to 4.5 via HA UI (Settings → Helpers) — YAML initial only applies if entity is recreated from scratch; running state not auto-updated. (2) Heat-up duration capture — input_number.geyser_last_heat_up_minutes added (power_helpers.yaml); automation geyser_heat_up_duration_capture added (geyser_automations.yaml) — fires on binary_sensor.geyser_at_temperature → on, calculates elapsed time from switch.last_changed minus 5-min debounce, stores result + logs season/SOC/solar context. Build trend over 1–2 weeks; add statistics sensor for rolling average once enough data. Dashboard: add geyser_last_heat_up_minutes + geyser_at_temperature to Geyser Control view in Lovelace (HA UI — dashboard is in .storage). PENDING FUTURE: Option B polling trigger — replace 4 fixed morning time triggers with time_pattern /15min; add window-guard condition so polling only evaluates during a configured window (e.g. 03:30–06:00) — triggers become fully responsive to input_number values; orchestrator can then auto-adjust start based on heat-up trend (target_ready_hour minus avg_heat_up). Also: all geyser timing input_numbers should eventually be driven by heuristics (heat-up trend + energy state + season). POWER_CONTRACT geyser section updated.*

*2026-06-15 session (entity rename cleanup — load shedding + P6 stats fix): Four files changed (swept into Sprint 4 docs commit). (1) automations.yaml: 15 refs of stale `sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark` → `sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c` in legacy HA UI automations (notification message templates). Resolves Session F KNOWN OUTSTANDING ITEM (a) — was causing WARNING in HA logs on start. (2) packages/load_shedding/load_shedding_automations.yaml: 8 refs same rename — trigger entities and condition checks. (3) packages/power/configuration.yaml: `solar_vs_forecast_ratio_today` moved back into recorder (was excluded — needs to be recorded as source for `sensor.solar_forecast_accuracy_7d` statistics platform sensor). (4) packages/power/power_statistics.yaml: `unit_of_measurement` removed from 6 statistics platform sensors (inverter_production_7d_mean/30d_mean/7d_stdev, house_load_24h_mean/7d_mean, solar_forecast_accuracy_7d) — HA statistics platform inherits UoM from source sensor; explicit override was causing sensors to silently not register. Likely resolves Session F KNOWN OUTSTANDING ITEM (b). POWER_CONTRACT.md updated: load shedding entity name corrected in Source Integration Entities list + External Entity table. No new entities.*

*2026-06-15 session (prepaid dashboard fix — Sprint 3 closure): Dashboard-only fix. Two plotly graph cards in the prepaid dashboard (Prepaid Usage & Cost + Solar Savings vs Cost) were referencing non-existent `sensor.prepaid_total_spent` — sensor was commented out in prepaid_core.yaml; `input_number.prepaid_total_spent` is the correct entity. Both entity lines corrected to `input_number.prepaid_total_spent`. POWER_CONTRACT Sprint 3 now 4/4 complete. No YAML package changes.*

*2026-06-14 session (doc cleanup — Power Contract checklist + Lighting BUG-L09 + solcast fix): Maintenance session — no new features. (1) POWER_CONTRACT.md Sprint 1–4 checklist audited against live code: Sprint 1 (5/5 ✅ — battery_night_survival uses battery_energy_available + average_night_consumption; grid_charging_while_solar uses house_grid_power + grid_to_battery_power; prepaid_depletion_date uses prepaid_units_left_safe; buy_decision_notify condition checks buy_now/buy_soon not 'hold'; solar_surplus_available duplicate resolved via _power suffix rename). Sprint 2 (8/8 ✅ — inverter_1_pv_voltage elif already correct; inverter_today_energy_import ×2 confirmed acknowledged workaround; house_security_power group exists correctly; all solar helpers defined and wired; Solcast entity fixed — see (3); inverter_device_1_since_last_update rename confirmed; load_shedding unique_id typo confirmed fixed; grid_risk.yaml has no stale inverter_power refs). Sprint 3 (3/4 ✅ — grid_status_monitoring + inverter_pwer_monitoring not in automations.yaml; ssa_* refs absent; prepaid_total_spent dashboard refs unverifiable in .storage). Sprint 4 (3/5 ✅ — prepaid_balance_confidence + net_position_this_month + buy_score all implemented; auto-reconciliation + pyscript health check still open). (2) LIGHTING_CONTRACT.md BUG-L09: marked DONE — was incorrectly showing OPEN since 2026-04-29 when M1/M2/M3 completed it (lighting_entertainment.yaml populated, entertaining_mode guard added to lighting_bedtime.yaml). (3) Code fix: packages/power/solar_forecast.yaml:37 — sensor.solar_forecast_available_conservative was reading stale entity `solcast_forecast_forecast_today` (always returned float(0) default); corrected to `solcast_pv_forecast_forecast_today`. Sensor is unused externally — no automation impact, but now reads correct Solcast forecast. No new entities. No entity renames.*

*2026-06-14 session (dashboard Session F — Full Dashboard Pass, Orchestrator Series Complete): Dashboard-only session. No package YAML changed. Pre-flight confirmed all E1–E8 and P6 sensors live (sensor.energy_orchestrator_state: conserve; solar_vs_forecast_ratio_today: 60.3%; solar_weather_correlation: poor). Entity name corrections discovered and applied: session spec used house_known_load_power/house_unknown_load_power/house_load_visibility_percent — actual entities are sensor.known_load_power / sensor.unknown_load_power / sensor.load_visibility_score (power_templates.yaml); water_tank_level_percent → sensor.water_tank_level; load_shedding_area_jhbcitypower3_11_weltevredenpark → sensor.load_shedding_area_za_gt_jhb_weltevredenpark_pa5c; inverter_1/2_today_generation → inverter_1/2_today_production. P6 statistics sensors (inverter_production_7d_stdev, house_load_7d_mean, solar_forecast_accuracy_7d): defined in power_statistics.yaml, YAML parses OK, but NOT appearing in HA entity registry — NOT_FOUND after 2× restart. Pre-existing statistics sensors (7d_mean, 30d_mean, 24h_mean) load as 'unavailable' (no data). Open investigation: P6 stdev/7d_mean/forecast_accuracy sensors silently not registering — HA logs show no error, entity registry shows only 5 statistics platform sensors (pre-P6 plus average_night_consumption/water_daily). Dashboard changes (all via JSON storage file edits while HA stopped): (1) power-control view: added solar_weather_correlation + season to Orchestrator Inputs; added inverter_programme_auto_enabled + battery_capacity_kwh to Orchestrator Thresholds; added Load Visibility card (known/unknown load + visibility score); added pool_winter_start_hour to Pool Controls; added water_tank_level + water_predictive_fill_enabled to Borehole section; added tier4 + airfryer + unknown draw params to Energy Saving section; fixed load shedding entity name; added P6 stats to Forecast + Season card; added P4 programme context to Inverter Pattern card. (2) geyser-control view: extended logbook to 48h; added confirmation dialog to Run Geyser Now button. (3) inverter-control view: added Solar Performance Stats card and Season Calibration card to section 2. (4) power-history view (in debug dashboard): added Solar Performance section (apexcharts H1/H2/H3 + H4 season entities). (5) operations-debug dashboard: added Simulator + Inverter Sync Debug + Orchestrator Debug sections. Navbar audit: all views have standard custom:navbar-card with Home/Operations/Debug/System/Alerts routes — consistent, no changes needed. F-3 merge: energy-control and inverter-details were already merged into Operations dashboard views (power-control, inverter-control) in a prior session — no duplicate files to remove. HA restart performed (stop → edit → start) to persist storage file changes. KNOWN OUTSTANDING ITEMS after F: (a) sensor.load_shedding_area_jhbcitypower3_11_weltevredenpark wrong name also in automations.yaml triggers (Load Shedding: Warning 15 Minutes + 2 Hours) — causes WARNING in HA logs on start; fix requires automations.yaml edit (legacy UI file — do in next automation session); (b) 3 P6 statistics sensors not registering — needs investigation; (c) battery_capacity_kwh = 30 kWh — update to 45 after battery swap; (d) season efficiency factors need calibration after 3+ months data; (e) legacy solar scenes still present (remove after E5 validated 2+ weeks).*

*2026-06-14 session (lighting BUG-L14 — wrong entity in arrival automation): All three arrival scenarios used switch.stw_3gang_garage_switch_3; correct entity is switch.garage_light (controlled by lighting_garage.yaml presence detection). Fixed: replaced across all 3 scenarios in lighting_arrival_night.yaml. No new entities. Confirmed garage presence detection correct: door-assist fires on door open at night before presence kicks in; garage_occupied drives stable on/off state.*

*2026-06-14 session (lighting BUG-L12+L13 — arrival lights never fired + nobody-home front security missing): BUG-L12: arrival lighting was always blocked when presence_boundary set arrival_detected. Root cause: presence_boundary_resolver sets last_arrival_time=now() one action step BEFORE setting arrival_detected=on. The lighting automation's 60s cooldown evaluated (now - last_arrival_time) = ~0s > 60 → FALSE → always blocked. Only security Stage 1 arrivals worked (Stage 1 never touches last_arrival_time). When ipcam01 missed the approaching vehicle (Stage 1 condition fails), the sole path through presence_boundary was permanently gated. Fixed: removed cooldown condition entirely from lighting_arrival_night. The from:"off" to:"on" trigger + 5-min auto-clear (presence_clear_arrival_flag) already prevent invalid re-fires. BUG-L13: Scenario 1 (nobody home) missing switch.front_house_security_light — only Scenario 2 (someone home) had it; back_house_security was always-on but front was dark on empty-house arrivals. Fixed: added front_house_security_light to Scenario 1 switch list. One file changed: packages/lighting/lighting_arrival_night.yaml. No new entities.*

*2026-06-14 session (lighting BUG-L11 — morning wake no-op upper bound): cam14 lounge motion at 23:09 triggered the morning wake routine (morning_routine_ran_today was off; condition checked only now() >= morning_start which is true all day). Fixed: added noon ceiling to time condition — both weekday and weekend paths now require today_at(morning_start) <= now() < today_at('12:00:00'). One file changed: packages/lighting/lighting_morning.yaml condition block (4 lines, 3 deletions). No new entities. LIGHTING_CONTRACT.md BUG-L11 added and marked fixed. Domain status updated to L01–L11.*

*2026-06-14 session (power Session P6 — Solar Statistics + Weather Correlation): Standalone session, no E-series dependency. Pre-flight: power_statistics.yaml existed with 5 sensors (3 statistics, 2 templates) but was missing stdev, today ratio, 7d accuracy, season factor, and had only 2-state weather correlation. Recorder 90-day history confirmed. ha.core check passed before and after. Changes made: (1) power_statistics.yaml fully rewritten — added sensor.inverter_production_7d_stdev (7d stdev, CV signal for weather correlation), sensor.house_load_7d_mean (7d load mean kWh), sensor.solar_forecast_accuracy_7d (7d statistics mean of ratio_today), sensor.solar_season_efficiency_factor (JHB season multipliers: summer 1.0/autumn 0.75/winter 0.55/spring 0.85, UI-adjustable); upgraded sensor.solar_weather_correlation from 2-state (normal/degraded) to 4-state (excellent/good/poor/degraded — worst-match-wins); added max_age to existing stats sensors; retained solar_vs_forecast_ratio_7d and backward-compat 'ratio' attribute (energy_state.yaml reads it). (2) solar_state.yaml: added sensor.solar_vs_forecast_ratio_today — today's actual/forecast with 10am neutral guard (returns 100 before 10am). (3) solar_clipping.yaml: added sensor.solar_production_vs_capacity — PV output as % of system capacity (input_number.system_solar_capacity_w default 10800W). (4) power_helpers.yaml: added input_number.system_solar_capacity_w + 4 solar season factor helpers (solar_factor_summer/autumn/winter/spring). (5) power_automations.yaml: P4 evaluation variables block updated with season_factor + expected_daily context variables (context-only — not used in enable/disable condition, only in logbook message). (6) pyscript/power_snapshot.py: STATISTICS dict added with 10 new sensors; renders as separate section in snapshot output. (7) configuration.yaml: 6 new sensors added to recorder excludes. Backward compat verified: energy_state.yaml 'degraded' check preserved; 'ratio' attribute kept. HA restart required (new platform:statistics sensors + configuration.yaml recorder exclude change). ha core check passed. No errors in HA logs after restart. NOTE: statistics sensors (7d_mean, stdev, 7d accuracy) will show 'unknown' for first 7 days — insufficient history samples; this is expected and normal. Season factors need calibration after 3+ months of winter+summer data. Dashboard additions flagged for next dashboard session: solar_forecast_accuracy_7d, solar_season_efficiency_factor, solar_production_vs_capacity to Solar Performance panel; 7d bar chart for solar_vs_forecast_ratio_today.*

*2026-06-14 session (dashboard re-enable — home overview): Three cards in lovelace.dashboard_overview that had been placeholder-disabled "for testing" were restored from the backup (lovelace.dashboard_overview.bak). Re-enabled: (1) custom:power-flow-card-plus at sections[2].cards[2] — animated solar/battery/grid/load flow diagram with pool/geyser/water/borehole individual consumers; (2) load shedding vertical-stack at sections[2].cards[5] — custom:html-template-card 2-day slot timeline + stage chips + API quota display, visibility-gated to load shedding active; (3) borehole history vertical-stack at sections[2].cards[10] — water stats horizontal panels + history-graph for switch.borehole_pump. Restoration method: Python script replaced placeholder markdown cards with backup originals; WebSocket API (lovelace/config/save, url_path=dashboard-overview) pushed to HA in-memory state (no HA restart required). Dashboard change only — .storage/ is gitignored, no package YAML changed. Confirmed working in Chrome.*

*2026-06-14 session (power/water Session E6 — Borehole orchestrator integration): Pre-flight confirmed all emergency bypass branches (Safety, Critical+power, Critical limited) do NOT reference binary_sensor.water_refill_allowed — emergency paths are clean and untouched. E6-1 no-op: orchestrator gate from E1-4 already correctly applied (orch not in [critical, loadshedding, loadshedding_critical] at water_templates.yaml:320). E6-2 no-op: load_control_borehole_enabled already gated — Branch 2 (critical+power) checks it directly; Branch 4 (demand) checks it via water_refill_allowed. Pump entity confirmed as switch.borehole_pump (NOT switch.borehole_pump_switch). Added to water_helpers.yaml: input_boolean.water_predictive_fill_enabled (default: false, disabled pending water_usage_today implementation), input_number.water_predictive_fill_threshold_percent (50%), input_number.water_max_fill_hours_per_day (2h). Added sensor.borehole_control_status to water_state_extensions.yaml (9-state priority display: Disabled / Emergency fill / Blocked-critical / Blocked-grid+SOC / Filling / Tank full / Waiting-solar / Ready / Monitoring). E6-5 SKIP: sensor.water_usage_today does not exist — water_daily_usage_mean cannot be created; flagged as future water session item (implement water_usage_today first). Orchestrator gate test: with orch='critical', Would_refill_allowed_be_on=False ✅. ha core check passed. WATER_CONTRACT Section 5 updated: borehole_control_status documented, predictive fill helpers documented, future session item flagged.*

*2026-06-14 session (power Session E8 — Energy Scenario Simulator): Read-only scenario simulator evaluating orchestrator and appliance decisions using simulated inputs — no live entity changes. pyscript/energy_simulator.py created; mirrors energy_state.yaml orchestrator logic, geyser window logic, pool_pump_solar_control conditions, water_refill_allowed gate, airfryer_critical_cut conditions, inverter_energy_pattern_control + P4 evaluation. Safety gate: input_boolean.simulator_active must be ON (default OFF). 8 named scenario presets (Normal sunny / Poor solar / Stage 2+4 load shedding / Critical battery morning / Tennis night / Post-battery-upgrade surplus) + Custom mode. All orchestrator thresholds read live from input_number.* — threshold changes instantly affect simulation. Output: persistent_notification.energy_simulator + event energy_simulator_result. New helpers: input_boolean.simulator_active (false), input_select.simulator_scenario (8 presets + Custom), input_number.simulator_soc_override/solar_override/load_override/hour_override/shed_stage_override. Two YAML section errors fixed during session (numbers placed in input_datetime section, moved to input_number — same class of error as E7). ha core check passed. POWER_CONTRACT Section 8 simulator section added. E8 closes the orchestrator design arc: E1 core → E2 geyser migration → E3 pool retrofit → E4 geyser upgrade → E5 inverter programme → E6 borehole → E7 unknown draw → E8 simulator.*

*2026-06-14 session (power Session E7 — Unknown draw detection, air fryer control): Pre-flight found all three E7 target sensors already exist: sensor.known_load_power (kW), sensor.unknown_load_power (kW), sensor.load_visibility_score (%) — all in power_templates.yaml using group.known_power_loads. E7-2 was a no-op. Entity name corrections: air fryer switch = switch.philips_airfryer_plug (spec said switch.air_fryer_switch), power = sensor.philips_airfryer_plug_power. Known appliance inventory documented: 13 sensors in group.known_power_loads; switch entities confirmed for iron, air fryer, dishwasher, two washers, pool pump, pond pump, geyser, borehole. Honor tablets confirmed: already in all STD_* notify groups via mobile_app_honor10_dash / mobile_app_honorx7_dash — no separate chime mechanism exists, push notifications via script.notify_power_event are the chime. New helpers in power_helpers.yaml: load_control_airfryer_enabled (bool, true), tier4_warnings_enabled (bool, true), unknown_draw_warning_threshold (1500W), unknown_draw_critical_threshold (3000W), unknown_draw_duration_trigger (3min), input_datetime.unknown_draw_last_alert (30-min cooldown gate). New automations in power_automations.yaml: automation.airfryer_critical_cut (turns off switch.philips_airfryer_plug on orchestrator critical/loadshedding_critical; conditions: load_control_airfryer_enabled + switch on + grid off + SOC below critical threshold — corrected after initial implementation: only cuts when grid off AND SOC low, NOT on every critical state), automation.airfryer_restore_on_recovery (info notify on recovery, no auto-restart), automation.unknown_draw_warning (dual-branch: critical branch severity:critical, warning branch severity:warning; 30-min cooldown via input_datetime; only fires when orchestrator under pressure). ha core check passed. Fixed YAML error: booleans accidentally placed in input_number section, moved to input_boolean. Functional checks: load_control_airfryer_enabled=on, tier4_warnings_enabled=on, warning_threshold=1500W, known=0.28kW, unknown=0.43kW, visibility=39.3% (correct for evening baseline).*

*2026-06-14 session (power Session E5 — Inverter programme automation): Three new automations added to power_automations.yaml. (1) inverter_energy_pattern_control (E5-2): Battery First / Load First crossover. Triggers: time_pattern /5min + orchestrator state change + homeassistant start + time 16:30. Four branches: Branch 1 (orchestrator in [loadshedding, loadshedding_critical, critical, conserve] → Battery First, severity warning/info); Branch 2 (morning, hour < 10, SOC < 65% threshold → Battery First, silent); Branch 3 (orchestrator normal/surplus, SOC ≥ 65%, 09:00–16:00 → Load First); Branch 4 (16:30 evening return → Battery First). All pattern changes call force_inverter_sync after 5s delay. (2) inverter_p4_grid_charge_control (E5-3): Dynamic P4 grid charge at 14:00–17:00 window. Evaluates at 14:00/14:30/15:00: if SOC gap > 5% AND kwh_shortfall > orchestrator_solar_gap_threshold (2 kWh) → enable P4 Grid on both inverters directly; else if P4 is Grid and on track → disable. 17:00 trigger always restores P4 to Disabled. Uses sensor.house_load_24h_mean (24h mean W) for load estimate. (3) solar_forecast.yaml gated by input_boolean.use_legacy_solar_scenes (default: false) — scenes no longer fire; E5 owns inverter control. Rollback: set use_legacy_solar_scenes to on. inverter_sync_check expanded: now triggers on all 6 program_N_charging changes + energy_pattern; compares all 7 entities; details mismatch. force_inverter_sync expanded: copies energy_pattern + all 6 charging selects + all 6 SOC numbers (INV1→INV2). New helpers: inverter_programme_auto_enabled (bool, true), use_legacy_solar_scenes (bool, false), orchestrator_p4_charge_trigger_soc (70%), orchestrator_solar_gap_threshold (2 kWh), battery_capacity_kwh (30 kWh — update to 45 after Tuesday swap). Pre-flight revealed: battery = 30 kWh (not 17.3); P4 window = 14:00–17:00 (confirmed from Developer Tools); all inverter_2_program entities confirmed via scenes.yaml. ha core check passed. Functional check: orchestrator=conserve → energy_pattern changed Load First → Battery First immediately on reload. P4=Disabled (17:00 already closed). POWER_CONTRACT Inverter Sync section updated (removed PARTIAL flag), Sprint 5 marked IMPLEMENTED, E5-2/E5-3 automation logic documented.*

*2026-06-14 session (power Session E4 — Geyser orchestrator retrofit + midday solar gate): geyser_automations.yaml fully rewritten (E2 baseline → E4). geyser_turn_on: 5 branches. Morning non-negotiable — only loadshedding_critical blocks (hot water essential); morning_override bypasses load_control_geyser_enabled; 4 fixed triggers (03:30/04:00/04:30/05:00) match weekday+weekend×winter start time helpers; branch condition encodes season+weekday logic. Midday upgraded from unconditional (E2) to solar-gated: orchestrator in [surplus,normal] + solar_available_surplus > geyser_midday_surplus_threshold (300W) + geyser_at_temperature off + before 15:00. Evening standard 20:00 non-winter / winter 19:30 (30 min earlier); sports night 21:00. geyser_turn_off: 7 branches + default, mode:queued. Morning hard-off added (07:30/08:00/08:30/09:00 weekday/weekend×winter). Midday hard-off at 15:00. Evening standard 21:30/winter 22:00/sports 23:00/winter sports 23:30. Orchestrator emergency off: loadshedding_critical → turn off outside morning window only (morning sacred). AM battery protection preserved (grid off + SOC < prog2 + shedding — fires even in morning window). New sensors added to power_state.yaml: sensor.geyser_control_status (13-state priority display) + binary_sensor.geyser_at_temperature (power proxy: <50W sustained 5min = at temp). New helpers in power_helpers.yaml: geyser_morning_start_weekday (4h), geyser_morning_start_weekend (5h), geyser_morning_end_weekday (7.5h), geyser_morning_end_weekend (8.5h), geyser_winter_start_offset (30min), geyser_midday_surplus_threshold (300W). Entity name correction: canonical solar surplus is sensor.solar_available_surplus (Watts) NOT sensor.solar_surplus_available (boolean string). ha core check passed. Functional check: geyser_control_status=Outside active windows (correct — between 15:00 and evening start at ~19:00), geyser_at_temperature=off (drawing full power), switch=on, helpers loaded. POWER_CONTRACT geyser section rewritten (E4).*

*2026-06-14 session (power Session E3 — Pool pump orchestrator retrofit): pool_pump_solar_control in power_automations.yaml retrofitted to use sensor.energy_orchestrator_state as the primary gate — this is the reference pattern all appliance automations will follow. Turn-on (Branch 5): replaced raw battery_state_health + load_shedding_stage + upcoming-shed SOC checks with `orchestrator in ['surplus', 'normal']` as the single gate; local conditions preserved: grid-offline SOC floor (grid_offline_soc_min_pool 60%), last-sun-slot (last_sun_soc_target 80%/90%), winter morning hold (pool_winter_start_hour 10h). Turn-off: Branch 2 replaced raw load_shedding_stage >= 2 / upcoming-shed logic with `orchestrator in [loadshedding, loadshedding_critical, critical]`; loadshedding_critical overrides minimum run time protection (emergency turn-off); other states respect min run time, notify at warning. Branch 3 replaced raw battery_state_health + headroom < 800W with `orchestrator == conserve OR headroom < 800W`. sensor.pool_pump_control_status (power_state.yaml) fully rewritten to read orchestrator state — 11-state priority display: Disabled / load-shedding blocked / critical battery / target reached / running (surplus/min-time) / conserve / last sun slot / winter morning hold / outside hours / solar insufficient / waiting. Entity name fix: sensor.solar_surplus_available (boolean True/False string in solar_core.yaml) replaced with sensor.solar_available_surplus (canonical Watts metric in power_state.yaml) in lovelace.dashboard_operations line 5212. ha core check passed. Functional check: orchestrator=conserve, pool status=Off—target reached (100 min / 90 min winter target), solar_available_surplus=-4412W (evening, correct), switch=off.*

*2026-06-14 session (power Session E2 — Geyser automation migration): automations.yaml IDs 1742224800650 (turn on) + 1744130174080 (turn off) migrated to packages/power/geyser_automations.yaml. Migration fixes: device_id → switch.geyser_heat_pump_switch (confirmed); sensor.inverter_power → sensor.inverter_load_power; notify.STD_Information → script.notify_power_event; load shedding check re-enabled (was blocked on string comparison against unavailable sensor — now uses state_attr | int(0) < 4). New automations: geyser_turn_on (4 branches: morning standard 04:30 / morning winter 04:00 / midday 12:05–14:05 unconditional / evening standard 20:05 / evening sports 21:05), geyser_turn_off (4 branches: AM battery protection 05:30–08:30 / winter evening 22:00 / standard evening 21:05 / sports night 23:05), geyser_sports_night_scheduler (Tue+Thu 17:00 → on; 00:01 daily → off), geyser_manual_run (timed run via input_select.geyser_manual_run_duration 30/60min). script.geyser_manual_run dashboard entry point. Seasonal trigger design: dual triggers with IDs (04:00+04:30; 21:05+22:00); both fire daily, branch condition filters by trigger.id + sensor.season — single automation, no code duplication. New helpers in power_helpers.yaml: geyser_sports_night (bool), geyser_morning_override (bool), geyser_manual_run_active (bool), geyser_manual_run_duration (select 30/60). ha core check passed. Functional check: switch.geyser_heat_pump_switch=on, geyser_sports_night=off, geyser_manual_run_active=off, geyser_manual_run_duration=30, geyser_enabled=on. All 4 automations confirmed live in Settings → Automations. POWER_CONTRACT Section 2 (file count 25→26) + Section 8 (geyser helpers + automation logic) updated. AUTOMATIONS_AUDIT.md geyser section marked MIGRATED.*

*2026-06-14 session (security — dogs_inside departure notification tap): Two files changed. (1) security_automations.yaml: automation.dogs_inside_from_notification added — handles mobile_app_notification_action DOGS_INSIDE_ON event; turns on input_boolean.dogs_inside + logs to logbook. No auto-off — must be cleared manually on return (arming logic prevents false alerts even if left on). (2) notify_security_events.yaml: departure Stage 1 push notification updated to include inline action button "DOGS INSIDE 🐕" (action DOGS_INSIDE_ON). Suppressed when dogs_inside already ON, guest_mode ON, or staff_on_site ON. Fires at Stage 1 (gate opened / ipcam01 departure trigger) so leaving family member can tap before driving off. SECURITY_CONTRACT sprint 9h+ section added. No new entities — dogs_inside and dogs_out already locked in PROJECT_STATE.md (added S9h 2026-05-20).*

*2026-06-14 session (power Session E1 — Energy Orchestrator core): sensor.energy_orchestrator_state rebuilt from basic 5-state sensor (run_heavy_loads/run_medium_loads/reduce_load/conserve_energy/normal) into 6-state priority ladder: loadshedding_critical / loadshedding / critical / conserve / surplus / normal. Priority logic: loadshedding_critical = load shedding active AND SOC < emergency threshold; loadshedding = active OR upcoming within pre_shed_hours_warning window AND SOC < pre_shed target; critical = battery_state_health == 'critical' OR SOC < critical threshold; conserve = health == 'low' OR SOC < conserve threshold OR solar degraded AND SOC < degraded floor; surplus = solar export > threshold AND battery strong/healthy; normal = default. Gated by input_boolean.orchestrator_enabled (when OFF returns normal always). sensor.orchestrator_decision_reason added — human-readable explanation for dashboard/logbook. 12 new input helpers in power_helpers.yaml: orchestrator_enabled (bool), inverter_sync_status (bool), plus 9 SOC/threshold input_numbers (emergency 15% / critical 25% / conserve 50% / pre_shed 80% / conserve_degraded_soc 60% / surplus_export 1000W / target_soc_by_sunset 90% / load_first 65% / pre_shed_hours 3h) + pool_winter_start_hour (10h). binary_sensor.water_refill_allowed updated: added orchestrator gate — blocks normal refills when state in [critical, loadshedding, loadshedding_critical]; conserve does NOT block. All three emergency branches (safety, critical+grid, critical+limited) in water_tank_refill_control.yaml confirmed separate and unaffected. Inverter sync check: automation.inverter_sync_check (90s delay after energy_pattern change, compares and sets inverter_sync_status, notifies on mismatch) + script.force_inverter_sync (copies INV1 energy_pattern to INV2) added. Currently partial — only select.inverter_1/2_energy_pattern confirmed; work_mode/program_N_charging/inv2_soc entities need verification in Developer Tools. Spec correction: battery_state_health states are strong/healthy/low/critical — no 'warning' state (spec said ['low','warning'] for conserve; corrected to ['low']). Degraded SOC floor parameterised (was hardcoded 60% — moved to input_number.orchestrator_conserve_degraded_soc_threshold). ha core check passed. Functional check: orchestrator = normal (SOC 64%, battery healthy, no surplus, no load shedding — correct). water_refill_allowed orchestrator_state attribute confirmed. POWER_CONTRACT Section 7 (orchestrator states) + Section 8 (helpers + inverter sync) updated. WATER_CONTRACT Section 5 cross-domain table + refill_allowed definition updated.*

*2026-05-28 session (network: Synology NAS graceful shutdown + WoL restore + HA Pi shutdown): network_nas.yaml created (3rd network/ file). Three-stage shutdown pipeline: Stage 1 warn at <15 min remaining (critical notification, grace countdown); Stage 2 grace period (5 min, cancellable by toggling ups_nas_auto_shutdown_enabled OFF or grid restore); Stage 3 NAS shutdown (button.guardians_shutdown) → 4 min wait → hassio.host_shutdown (Pi). binary_sensor.ups_nas_shutdown_imminent = ON when on battery AND runtime < warn_threshold. WoL restore: HA startup trigger → if ups_nas_was_shutdown flag ON AND NAS offline → 3 min delay → button.wol_synology_nas_00_11_32_ad_af_a5 → 5 min wait → confirm/warn → clear flag. WoL broadcast_address bug fixed in .storage/core.config_entries: was 192.168.1.6 (unicast to powered-off IP, never works) → 192.168.1.255 (subnet broadcast). NAS on 192.168.1.x, Pi on 10.10.1.x (cross-subnet WoL via routing). DSM WoL confirmed enabled on LAN 1 + LAN 2 physical NICs; auto-restart on power failure also enabled in DSM. WoL targets physical NIC MAC (00:11:32:AD:AF:A5), not bond interface (Bond 1 = LAN1+LAN2, WoL is per-NIC only). 3 automations + binary sensor + 5 input helpers. Restart required (new helpers). See NETWORK_CONTRACT.md Section 10.*

*2026-05-28 session (dashboard battery alerts + Mobile Devices view): alerts_batteries.yaml created (15th alerts/ file) — canonical battery alert pipeline for Honor 10 Dash (HEY3-W00, 10100 mAh, 39.1 Wh) and Honor X7 Dash (JMS-W09, 7020 mAh, 27.2 Wh). Full pipeline: per-device low binary sensors (delay_on 1 min, < 30% AND not charging) + per-device overcharge binary sensors (delay_on 2 h, ≥ 95% while charging) → binary_sensor.dash_battery_alert_active → sensor.dash_battery_alert_context (critical/warning/normal, devices[] with SOC + runtime) → alert.dash_battery_alert (30/60 min repeat, STD_Alerts) → global aggregator. Runtime estimate sensors: honor10_dash_battery_time_remaining_est / honorx7_dash_battery_time_remaining_est (minutes; -1 when charging; 0.5W doze fallback when power < 0.1W). Screen brightness management added to packages/admin/tablets.yaml (new admin/ package, first file): 4 automations — night_dim (night_mode ON → night brightness), morning_restore (night_mode OFF → day or away), away_dim (nobody_home ON, not night → away brightness), return_restore (nobody_home OFF, not night → day brightness). Three input_number helpers for day/night/away brightness levels (0–255 Android scale). ⚠️ Restart required (alert: entity). Mobile Devices dashboard view (lovelace.dashboard_operations, mobile-devices path): 3-column sections layout built — col 1: summary tiles + alert status card + brightness sliders + alert settings; col 2: Honor Pad 10 detail (battery/state/runtime, health/temp/power, cycles/low/overcharge, screen/interactive/doze, per-device brightness buttons); col 3: Honor Pad X7 (same structure). Heading badges fixed: type: entity (type: template not supported in heading card badges). 16 new locked entities — see Dashboard Battery Alert Entities section. Network: sensor.ups_accessories_power (sum USB1/2/3/TypeC/DC) and sensor.ups_visibility_score (accessories/total × 100) added to network_ups.yaml.*

*2026-05-27 session (security S15 — RUNG 5 three-way split + gate_activity rung): sensor.security_event_classification now has 12 output states (was 10). RUNG 5 split into three: RUNG 5a visitor (entrance_valid + allhome OR nobody home), RUNG 5b gate_activity (entrance_valid + some family home — ambiguous arrival/visitor, neutral notification with gate control), RUNG 5c perimeter_front (no entrance_valid — street/passing activity, always fires, dynamic severity: warning daytime+family home / critical night or away). entrance_valid = binary_sensor.ipcam01_street_driveway_up_entrance_valid (AcuSense regionentrance, Higher validity precision zone at gate). BUG-S47 fixed (two paths): (1) security_image_inside_main slot never written during RUNG 2.5 events — security_capture_best_snapshot skips low/none confidence; inside cameras excluded from confidence scoring; added zone-slot write to security_capture_each_camera_motion for cam14/cam15/cam05. (2) Visitor/perimeter_front notification image stale (timing race — router reads slot before security_capture_best_snapshot writes it); added 4s delay + local image re-read in visitor and perimeter_front branches (same pattern as critical_intrusion 3s delay). BUG-S44 fixed: cam14/cam15 NVR false motion from go2rtc reconnect replay — delay_on: 3s added to both debounce sensors (cameras_processing.yaml). BUG-S45 partial fix: pool alarm deactivation — ipcam04_deactivate_alarm REST command added + security_dogs_out_cancel_alarm automation. BUG-S46 fixed: RUNG 8 missing not arriving guard (cam15 false critical during family AP transition). ipcam03 dog AcuSense misclassification noted (camera fix required — Minimum Target Size in Region Exiting rule). No new helper entities. Files: security_logic.yaml, security_automations.yaml, cameras_processing.yaml, security_helpers.yaml, SECURITY_CONTRACT.md.*

*2026-05-23 session (security S13 — BUG-S37 through BUG-S43 implemented): All 7 bugs fixed. BUG-S37: ipcam03_driveway_exit_valid now includes linedetection (departure line crossing). BUG-S38: ipcam01_recent window 120s→180s. BUG-S39: RUNG 2.5 hard 5am floor added (now().hour < 5); 90s cooldown on critical_intrusion branch (input_datetime.last_critical_event stamped before delay); 3s delay + re-eval image in critical branch (snapshot race fixed). BUG-S40: visitor cooldown 1800s when staff_on_site ON (was flat 30s). BUG-S41: security_image_arrival_locked added (locked at Stage 1 T+5s before cam04 overwrites grounds_front); Stage 2 reads locked slot for both arrival confirmed and Stage 1 notification. BUG-S42: Stage 1 title softened to "Gate opened — vehicle entering"; Stage 2 Unknown → "Nobody newly detected — visitor, delivery, or same-person return." BUG-S43: security_lighting_required entity reference fixed (security_visibility_low → security_weather_low_light); boundary_security_on switch.turn_on calls both get continue_on_error: true. New entities: input_datetime.last_critical_event, input_text.security_image_arrival_locked. Files: cameras_processing.yaml, security_core.yaml, security_logic.yaml, security_helpers.yaml, security_automations.yaml, lighting_boundary.yaml.*
*2026-05-23 session (security S12 — diagnosis + gitupdate.sh fix): Diagnosis-only session. Investigated real failures from 2026-05-23 field events. 7 new bugs identified (BUG-S37 through BUG-S43) — no code shipped, pending user approval. Summary: (1) BUG-S39 CRITICAL — RUNG 2.5 false critical ×6 at 05:03am when family rose early: is_late_night (h<6) still true, all_family_in_bedrooms still true (AP delay), ext_recent true from passing car; zero cooldown on critical_intrusion → 5 queued router dispatches all fired; first snapshot blank (2s race), subsequent 5 stale same-image. (2) BUG-S37 HIGH — ipcam03_driveway_exit_valid watches only regionexiting; S11b set line crossing to departure direction but linedetection signal ignored by exit_valid → missed departures. (3) BUG-S40 — staff visitor spam: S10 removed `not staff` from RUNG 5 (correct for maid-at-gate), but Saturday gardener cleans all morning at front gate → 30s cooldown gives repeated visitor notifications. (4) BUG-S41 — Stage 2 shows carport NVR image: cam04 ranks above ipcam03 in trigger_camera priority; car drives in → cam04 overwrites grounds_front image slot; Stage 2 fires 3.5min later, reads stale carport dark frame. (5) BUG-S38 — ipcam01 120s window too narrow: two gate opens showed first arrival missed because ipcam01 hadn't fired within 120s. (6) BUG-S42 — delivery person opening gate = false "Arrival — vehicle entering" (structural limitation of Stage 1 gate+ipcam01 logic). (7) BUG-S43 — security_lighting_required references non-existent binary_sensor.security_visibility_low (correct name: security_weather_low_light) → weather condition never triggers boundary lights. Also fixed: gitupdate.sh — sync_docs.sh failure was silent (no error check on call). Fixed to surface sync_docs.sh exit code with warning. sync_docs.sh confirmed working: ha-biesie-docs in sync with local docs (SSH deploy key at ~/.ssh/id_ed25519 is correctly scoped to ha-biesie-docs; main config repo uses /config/.ssh/id_rsa via core.sshcommand). Files changed: gitupdate.sh (sync_docs.sh error propagation).*
*2026-05-22 session (security_external_motion_recent moved to security_logic.yaml): Entity was defined in security_zones.yaml but did not create after reload (entity not found). Moved to security_logic.yaml alongside binary_sensor.security_intruder_active which uses the same delay_off pattern and is confirmed working. No functional change — same entity_id, same logic, same delay_off: 5min.*
*2026-05-22 session (security S11c): ipcam03_driveway_entrance_valid: regionentrance → fielddetection. regionentrance disabled in camera UI after user removed Region Entrance zone from ipcam03. fielddetection restores confirmed_human signal for threat_level sensor. entrance_valid no longer used as Stage 1 trigger. NVR cam05 sensitivity raised 20→40 (NVR web UI, not code). ipcam04 Target Validity confirmed already High. File: cameras_processing.yaml.*
*2026-05-22 session (security S11b): Stage 1 rewired for new ipcam03 camera config. User disabled ipcam03 Region Entrance, set line crossing to driveway→gate (departure) direction only. Stage 1 arrival trigger changed from ipcam03_driveway_entrance_valid → binary_sensor.main_gate_sensor (from:off, to:on). Condition: gate trigger proceeds only if ipcam01_recent (<120s) = vehicle from street. exit_valid trigger: suppressed if gate opened <120s ago AND ipcam01_recent (arriving car in exit zone). cam_dir simplified: gate=arrival, exit=departure. File: security_automations.yaml.*
*2026-05-22 session (security S11 — false intruder fixes, outdoor corroboration, visitor immediate): anyone_connected_home: delay_off 2min added — was instantly OFF on AP disconnect → departing car triggered CRITICAL intruder. RUNG 7 + 8d: `not departing` guard added. New binary_sensor.security_external_motion_recent (security_zones.yaml, delay_off 5min) — stays ON for 5min after any outdoor camera fires. RUNG 2.5 (stay-mode lounge at night): added ext_recent requirement — NVR cam14 headlights/shadows no longer escalate without outdoor corroboration. RUNG 8 (nobody home critical): added (ext_recent OR ip_cam OR high_conf) — NVR cam14/15 alone without outdoor context falls silently to RUNG 8d (family_movement). zone_label: grounds split into 'grounds front'/'grounds rear' based on trigger camera (was always 'grounds' → router always used carport image for rear cameras). Router zone_img map: 'grounds rear' → security_image_grounds_rear added. Router reason/zone: changed from live state_attr to trigger.to_state.attributes (snapshot at trigger time, not stale re-evaluation). Visitor branch: 45s grace removed → immediate notification; cooldown 60s→30s. ipcam02 left unchanged (admin access pending, BUG-S28 still open from HA perspective). Files changed: presence_core.yaml, security_zones.yaml, security_logic.yaml, security_automations.yaml. New locked entity: binary_sensor.security_external_motion_recent.*
*2026-05-21 session (security S10 — direction fix, staff muting, perimeter always active): BUG-S36 — departure misclassified as arrival (confirmed Telegram 06:55): departing car backs through gate-mouth entrance zone → entrance_valid fires → Stage 1 classified as arrival → Stage 2 "Unknown home". Root: entrance_valid always treated as arrival. Fix: cam_dir in Stage 1 now checks ipcam01 (street-up) recency — if no street approach within 120s, entrance_valid = departure (car was already inside property backing out). exit_valid always = departure. Mutual 120s suppression condition added: if entrance_valid fires within 120s of exit_valid (or vice versa), the second sensor is blocked — prevents an arriving car crossing the exit zone on its way to garage from generating a spurious second Stage 1. direction variable simplified: cam_dir if cam_dir != 'unknown' else ap_dir (was cam_dir if not both_active else ap_dir). RUNG 4 (service_person): removed `perim` — perimeter (street cameras ipcam01/02) was being swallowed by service_person rung during maid hours, muting visitor/perimeter alerts at gate. Fix: RUNG 4 now only catches `grounds or inside_any`. RUNG 5 (visitor): removed `and not staff` — visitor at gate must fire even when maid is on site. Router service_person branch: changed from 10-min-cooldown push notification to silent logbook always (no push ever for staff movement). Stage 2 departure: suppress "Unknown departed" notification when staff_on_site=on (maid leaving triggers departure via ipcam03, Stage 2 gets "Unknown" since maid not in person list). Zone display format in reason attribute: human-readable names (`Grounds` instead of `-G---`). Files changed: security_logic.yaml (RUNG 4/5 + zones format), security_automations.yaml (Stage 1 condition + cam_dir + service_person router + Stage 2 departure suppress). SECURITY_CONTRACT Sprint 10 added + BUG-S36 documented. No entity additions/renames. Reload required: template entities + automations.*
*2026-05-08 session: Security camera system restructure — cam06 removed (uninstalled); cam01/cam10 deprecated; cam05 moved inside garage 2026-05-07 (was placed in grounds_front 2026-05-08 as false-critical workaround; reverted to inside_garage zone 2026-05-17 once S2 arming gates made the workaround obsolete); all 5 IP cameras fully wired and entity naming standardised to ipcam01–ipcam05 prefix (was mixed camip/ipcam — single-pass rename across 7 config files); perimeter_rear_motion fixed (was always OFF — cam03 never existed, now uses ipcam05_back_boundary); security_event_router elevated branch added (Root Cause 1 — single-camera daytime events were silently dropped); cam07 added to confidence front tier (Root Cause 2 — intruder never fired for cam07-only events); AcuSense entrance/exit debounce sensors added (ipcam03 entrance/exit, ipcam01 entrance, ipcam02 entrance stub); vehicle_approaching correlation fixed (street cameras only — gate is closed structure, driveway = inside boundary); vehicle_in_driveway new correlation added; security_visitor trigger corrected to street cameras only (ipcam01/02 + entrance sensors); security_arrival_detected uses ipcam03_entrance_valid as primary with motiondetection fallback; notification spam fixed (event_router now sole notifier, individual automations retain state/logbook only); critical message fixed (contextual — checks inside motion at fire time vs hardcoded "inside house"); docs updated (SECURITY_CONTRACT.md + PROJECT_STATE.md).*
*2026-05-06 session: Water notification clarity — 4 files changed to fix misleading "aborted due to fault" messaging on normal pump stops. Root cause: water_refill_aborted_due_to_safety was set for (1) tank reaching max depth (NORMAL completion) and (2) permission blocks (override off, low SOC, load control) — neither is a genuine hardware fault. Fix: (1) water_safety.yaml water_stop_refill_at_max_depth no longer sets the abort flag; lifecycle now shows "Completed (Full)" instead of "Aborted (Safety)"; notification changed to "Tank Full" title with clear success language. (2) water_tank_refill_control.yaml water_block_refill_when_not_allowed no longer sets the abort flag; added notification with specific reason detection (master switch off / load control override / battery too low / grid offline). (3) water_refill_capture.yaml start/end notifications: removed [Code: ...] prefixes, added descriptive titles ("Refill Started"/"Refill Complete"), added depth rise delta to end message. (4) water_templates.yaml: sensor.water_refill_outcome_display now shows "Completed (Full)" when end depth >= 1.95m; sensor.water_refill_blocked_reason priority 1 reworded from "pump halted due to fault" to "Last refill stopped by fault protection — check borehole or pump" (now only fires for genuine no-rise stops). Abort flag now has single clear meaning: borehole no-rise protection fired. ha core check passed. Reload: automations + template entities.*
*2026-04-29 session (consolidated): Recovery mode fix — YAML/Jinja2 Rule 6 added to CODING_STANDARDS (notify_power_event + notify_security_events had {% if %} inside YAML keys → split to choose: branches). ha-biesie-docs public docs repo created + sync_docs.sh auto-sync wired into every gitupdate.sh commit. FETCH_URLS.md (24 URLs for Claude chat context seeding). AUTOMATIONS_AUDIT.md created — 9 active automations remain in automations.yaml (7 power session, 2 geyser DO NOT TOUCH). alerts_garden.yaml + GARDEN_CONTRACT.md (pond pump unscheduled-run alert pipeline). M1 entertainment mode: lighting_entertainment.yaml populated, entertaining_mode guard added to both kids_bedtime automations. M2/M3 energy saving: helpers + lighting_energy_saving.yaml populated; morning clear added. 3 automations deleted/migrated from automations.yaml (pond pump, 3 load-shedding warning blocks). IDS stub created (security_alarm.yaml). CTX01/02/03 context cleanup: context_presence.yaml → presence_trust.yaml, context_schedules.yaml deleted, home_context rewritten to remove security/ import.*
*2026-04-30 session: Doc sweep — verified live against actual files: BUG-NET01/02/03 confirmed fixed in network_helpers.yaml (availability references corrected, packet loss removed from health score formula). alerts_presence.yaml 158 lines fully implemented (not 17-line stub — contract was stale). borehole gate already marked done. Infra domain table updated (all 4 bugs fixed). Network sensor reliability (unifi_cpu_5m_max, unifi_memory_5m_max, wan_health_score) — formula fixes confirmed in code; live value reliability (whether UniFi integration feeds valid data) is a runtime check, not code — verify in Developer Tools → States.*
*2026-04-30 session: average_night_consumption sampling bug fixed — energy_state.yaml: removed sampling_size: 20 (at 5s solarman poll rate it gave ~100s effective window, silently overriding max_age: 24h because HA statistics platform uses whichever limit is hit first). Now uses max_age: hours: 24 only → true 24h rolling mean. ALERTS_CONTRACT updated: alerts_presence.yaml line count corrected 17→~158 (was fully implemented 2026-04-16, line count never updated). Infra domain table updated: all 4 bugs marked fixed (were already confirmed fixed in session log but table still said "4 bugs found"). Water borehole gate confirmed ✅ done (already in Load Control section line 236, was just stale in old TODO text). BUG-CTX01/CTX03 fixed (see below).*
*2026-04-30 session: BUG-CTX01 fixed — context_presence.yaml content moved to packages/presence/presence_trust.yaml; context_presence.yaml deleted. Entity IDs unchanged. BUG-CTX03 fixed — sensor.home_context in context_global.yaml no longer imports sensor.security_mode or sensor.security_trust_mode from security/; now derives from binary_sensor.security_nobody_home (self-contained in context_global) + binary_sensor.night_confirmed; trust attribute simplified to low_trust/normal via binary_sensor.low_trust_present. ha core check passed. context/ package: 4→2 files. presence/ package: 5→6 files. All three context bugs confirmed closed.*
*2026-04-30 session: Telegram backslash escape fix — all 6 notify scripts had 18-character MarkdownV2 escape chains applied (BUG-N09 fix 2026-04-13), but the Telegram bot integration config has `options.parse_mode: markdown` (old Markdown, not MarkdownV2). Old Markdown does not recognise backslash escapes for `.`, `(`, `)`, `+`, `|` etc., so they appeared literally in all messages. Reduced escape chain to 4 characters in all 6 scripts: `\\`, `\*`, `\_`, `` \` `` (only characters special in old Markdown). NOTIFICATIONS_CONTRACT.md BUG-N09 updated with revised history. Rule: do NOT expand escape chains beyond these 4 unless the Telegram integration is switched to parse_mode: markdownv2.*
*2026-04-30 session: WAN outage 04:30–05:00 caused cascade of Telegram ConnectTimeout errors (geyser AM/PM at 04:30, bar via presence at 04:45, git push at 05:00). Root issue: `continue_on_error: true` was on the Telegram send INSIDE notify scripts but NOT on the calling `script.notify_lighting_event` action — ConnectTimeout propagates as "Unexpected error" which bypasses the inner guard. Fixed: 23 call sites across 7 lighting files (lighting_bar_presence +6, lighting_evening +3, lighting_morning +4, lighting_arrival_night +3, lighting_boundary +2, lighting_garage +2, lighting_office_presence +3). gitupdate.sh hardened: now skips commit+push if `git diff --cached --quiet` (nothing staged) → exits 0 cleanly instead of propagating git's exit 128 when there's nothing to push. github.yaml backup failure message: added `| default('no stderr')` guard on backup_result['stderr'] (was None, causing Jinja2 error in the failure notification). Prepaid Repairs error (notify.std_warning in automations.yaml legacy automation) — confirmed already documented in session notes; fix via HA UI only (DO NOT TOUCH automations.yaml rule).*
*2026-05-18 session (security classifier hardening — BUG-S18/S19/S20 + FMT-01): BUG-S18 — false CRITICAL INTRUDER at night with `home=all`: root cause was RUNG 3 excluding `inside_armed_active`, so cam14 motion after 23:00 (inside_cameras_armed=on) bypassed family_movement and hit RUNG 8 simultaneously with NVR cam04 headlight triggers outside. Fix: (1) removed `and not inside_armed_active` from RUNG 3 — when family is home, ALL inside motion → family_movement regardless of arming state; (2) added `and not anyhome and not staff` guard to RUNG 8 — critical_intrusion only fires when property is genuinely unoccupied. BUG-S19 — staff on site spam (maid/gardener all-day cam09 motion): added 10-min cooldown to service_person branch in security_event_router using `input_datetime.last_visitor_event`; sets datetime after each notification fires, suppresses if within 600s. BUG-S20 — arrival notification spam (3-5 per arrival event): event_router was firing for every camera trigger during arrival sequence as classifier flipped idle→arrival repeatedly. Fix: arrival and departure branches in security_event_router now log to logbook only; stage1/stage2 automations handle all arrival/departure notifications. OPT-8.4 — cam14_lounge_motion_valid and cam15_passage_motion_valid added to TOP of security_trigger_camera priority list (was causing trig=none when inside cameras fired alone). FMT-01 — reason attribute reformatted: changed from flat `zones=PG---` format (YAML `>` folding collapsed all to one line) to 3-line output using `'\n'` in Jinja2 string concatenation: line 1 = zones/gate/home, line 2 = arriving/departing/staff, line 3 = conf/cam. Also changed all `=` separators to `:`. Extended RUNG 4 (service_person) to include `inside_any` zones — staff legitimately enter the house; inside camera triggers during scheduled staff hours → service_person not critical_intrusion. SECURITY_CONTRACT Sprint 7 section added with all 5 items documented. Files changed: security_logic.yaml (RUNG 3/4/8 + trigger camera priority + reason attribute), security_automations.yaml (service_person cooldown + arrival/departure logbook-only). Reload: template entities + automations.*
*2026-05-17 session (camera fleet cleanup): cam01_street_driveway, cam06_front_entrance, cam10_pool_bar deprecated/uninstalled entities purged from packages/security/ (input_text helpers, history_cleanup entries, pyscript list). cam05 renamed cam05_front_driveway → cam05_inside_garage across cameras_processing, security_zones, security_logic, security_automations, security_helpers, history_cleanup, pyscript. NVR hardware entity cam05_front_driveway_motiondetection unchanged (integration-provided). Active fleet locked: 7 NVR (cam04/05_inside_garage/07/09/12/14/15) + 5 IP (ipcam01-05). SECURITY_CONTRACT Section 3 rewritten as authoritative zone map. Entity registry orphans for cam01/06/10 (image.*, sensor.*, input_text.*) need manual Settings → Entities cleanup. Dashboard: cam01 has 13 refs in lovelace.dashboard_operations — needs user attention.*
*2026-05-17 session (security S3 — single notify path): Triple-fire dedup gate (2026-04-16, via binary_sensor.security_intruder_active 30s delay_off) was a Band-Aid — the trio still sent notifications (correction: the trio was ALREADY modified to NOT send notifications by some later session; at S3 pre-flight, trio was found to only manage security_event_active state + logbook + last_security_event). Architectural fix: security_event_router rebuilt to trigger on sensor.security_event_classification (S2 presence-first classifier), dispatch by classifier output, mode: queued. Deleted: security_grounds_motion, security_rear_grounds_motion, security_house_motion, security_visitor, security_arrival_detected. Added: security_arrival_stage1_vehicle / _stage2_confirm, security_departure_stage1_vehicle / _stage2_confirm (two-stage arrival/departure with AP confirmation at 3.5min). Visitor classification → critical severity (session decision). DSC alarm hook stubs in intruder + critical_intrusion branches. notify_security_event severity confirmed: must use 'information' not 'info'. SECURITY_CONTRACT Issues 2 (triple-fire), 7 (from: constraints partial), 8 (trust filtering via classifier) marked resolved/deferred. Reload: automations.*
*2026-05-17 session (workflow hardening): CODING_STANDARDS Rule 7 strengthened — docs-with-code is now blocking, not advisory. SESSION_CHECKLIST.md created at repo root — visible session-close enforcement mechanism. NVR placeholder slot devices (cam02/03/11/13/16-future) removed; NVR roadmap documented: all future additions will be IP-based, existing NVR fleet progressively replaced.*
*2026-05-17 session (cam05 placement correction): cam05 reverted from grounds_front to inside_garage zone. cam05 watches garage interior, not driveway approach (ipcam03 = driveway). The 2026-05-08 grounds_front placement was a workaround for false critical_intrusion alerts in the old single-zone model. With S2 zone arming gates (inside_garage_armed defaults OFF), the workaround is no longer needed and was causing real classification bugs (garage interior motion routing through intruder path). Removed from security_grounds_front_cameras group in cameras_core.yaml. Docs corrected: SECURITY_CONTRACT cam05 table + garage design block.*
*2026-05-17 session (inter-sprint doc reconciliation — S1+S2): P01/P02 confirmed already fixed 2026-04-15 (PROJECT_STATE was accurate; PRESENCE_CONTRACT bug list was stale). P10 was N/A — two staff vars in different sensors, not duplicates. Startup sync confirmed in presence_trust.yaml not presence_boundary.yaml. BUG-P03 body corrected — actual pre-fix state was maid-Monday-only logic (not the deprecated input_datetime situation described). Section 2 file inventory updated: presence_trust.yaml documented as startup sync owner; orphan helpers (input_boolean.low_trust_present / staff_on_site) noted as harmless. zone aggregation table updated. Architecture Rule 7 added (zone arming separates from zone motion). Locked entity names added for all S2 entities. [NOTE: cam05 entry in this reconciliation was wrong — corrected in same-day follow-up commit above.]*
*2026-05-17 session (S2 — Presence-First Classifier + Three-Zone Inside Split): New sensors added: binary_sensor.security_inside_garage_motion (cam05), security_inside_main_motion (cam14), security_inside_bedrooms_motion (cam15), inside_garage_armed / _main_armed / _bedrooms_armed (DSC-ready stubs backed by input_boolean.*_armed_manual), binary_sensor.family_arriving / family_departing / all_family_home (AP-location recency, 600s window, room names from unifi_ap_room_map). sensor.security_event_classification replaced: 5-state simple → 9-rung presence-first ladder (idle/arrival/departure/family_movement/service_person/visitor/perimeter_threat/intruder/critical_intrusion) with reason + zone_label attributes. security_inside_house_motion updated to raw union of three zone sensors (arming gate moved to classifier). SECURITY_CONTRACT Section 11 status: Design → Implemented. BUG-S14/S15/S16/S17 and BUG-P13 closed. ha core check passed. Reload: template entities + helpers.*
*2026-05-17 session (S1 — Trust Layer Rewire + Startup Sync): Pre-flight verified trust chain swap (P01/P02) already applied — zero reads of input_boolean trust entities in code. S1.3: boundary_permissive_window rebuilt (security_core.yaml) — old maid-Monday-only logic replaced with binary_sensor.low_trust_present OR guest_mode OR boundary_permissive_override; gate_open_too_long_permissive now fires for all staff/contractor/guest scenarios. S1.4: arrival_detected wired — both Pass 1 and Pass 2 arrival branches in presence_boundary_resolver now set input_boolean.arrival_detected; presence_clear_arrival_flag auto-clear automation added (5 min, mode: restart); lighting_arrival_night and security_arrival_detected now fire on real arrivals. S1.6: startup sync extended — gardener_on_site now restored on HA restart if inside gardener window and low_trust_enabled; contractor excluded (manual-only, HA state restore sufficient). BUG-P10 confirmed N/A — two staff vars are in separate sensors, both correct. GROUP A correction: A2 was only partially fixed 2026-04-15; fully corrected today. PRESENCE_CONTRACT updated: P01/P02/P03/P06/P10/P11/P12 all closed. Reload: template entities + automations.*
*2026-04-15 session (Group A audit): A1 confirmed superseded — low_trust_start/end never needed; complete trust chain runs through maid_start/end + gardener_start/end datetimes → schedule automations → maid_on_site/gardener_on_site booleans → binary_sensor.staff_on_site → binary_sensor.low_trust_present. A2 confirmed done — boundary_permissive_window reads maid_on_site + weekday()==0 + override, no longer always false. Group A fully complete.*
*2026-05-10 session: security_pool_alarm_trigger — tiered threat gate. Before: required `security_night_mode` ON + threat in ['warning','critical']. After: arms from 20:00 (night_mode OR hour>=20); critical always fires; warning fires when nobody home OR (evening + not night_mode); warning suppressed when family asleep (night_mode ON + home). Rationale: old logic missed the 20:00–22:30 window when people are still awake outside; family-asleep suppression prevents nuisance wakeups for low-confidence rear motion (dogs/wind). SECURITY_CONTRACT.md updated with pool alarm behaviour.*
*Last updated: 2026-05-08 — Security camera system restructure + entity rename complete (ipcam01–ipcam05 standardised — see SECURITY_CONTRACT.md for full detail)*
*2026-04-28 session (pool pump bugs): pool_pump_solar_control in power_automations.yaml — Bug 1: no 16:00 hard-stop trigger. Pump turned on at 15:00 by Branch 5 (solar surplus), ran until manually turned off at 18:05 (confirmed via DB: switch state change had empty context = physical/manual, not automation). Fix: added "16:00:00" time trigger (id: end_of_day); added Branch 0 (unconditional 16:00 turn-off, no minimum run time guard); removed before:"16:00:00" from global conditions; added before:"16:00:00" to Branch 5 turn-on conditions only. Bug 2: input_datetime.pool_pump_last_on never updated — mode:single caused Branch 1 (records pump start time) to be silently dropped (max_exceeded:silent) when it queued behind Branch 5 that had just turned the pump on. Result: pool_pump_continuous_run_minutes always showed ~1085 min (today_at('00:00:00') = midnight), so 45-min minimum run time guard was always trivially met. Fix: mode:single → mode:queued.*
*2026-04-29 session: HA Repairs "Prepaid Strategic Top-Up uses notify.std_warning" — confirmed transient error, no code change needed. automation.prepaid_strategic_top_up_suggestion is only in prepaid_core.yaml (correctly calls script.notify_power_event); no duplicate in automations.yaml. Error occurred at 22:29:11 during script reload window when notify services were briefly unavailable. API query confirmed all 4 notify groups live: std_information, std_critical, std_alerts, std_warning. HA Repairs notification dismissed. automations.yaml lines 1660+1708 (grid usage warning, legacy) use notify.std_warning directly — intentional, DO NOT TOUCH.*
*2026-04-29 session: Notify script Telegram audit — disable_notification removed from all 6 scripts (notify_light_events, notify_system_event, notify_water_events, notify_presence_events, notify_power_event, notify_security_events). Key was invalid in notify.send_message schema: top-level in light script, nested data.data in the rest. Caused runtime errors: garage lighting automation → notify_lighting_event failed; weather_api_recovery → notify_system_event failed. All 6 scripts confirmed clean against CODING_STANDARDS Rule 6 (all {% if %} blocks inside block scalars, none emitting YAML keys) and Telegram schema (message/title at data root; inline_keyboard under data.data in critical branches only; continue_on_error: true on all send_message calls). EcoFlow sensor.river_pro_ups_main_remain_capacity unit mismatch (mAh vs mA/A for device_class current) — third-party custom component bug in ecoflow_cloud; no HA config change possible, report to github.com/snell-evan-itt/hassio-ecoflow-cloud-US/issues.*
*2026-04-29 session: Garden alert pipeline: alerts_garden.yaml created — binary_sensor.garden_alert_active (pond pump after 11:00, 1min/5min delay), sensor.garden_alert_context, alert.garden_alert (60min repeat, STD_Alerts), garden_alert_ack_turn_off_pond_pump automation (mobile_app_notification_action handler — fixes broken wait_for_trigger-in-notify-data from original). alert.garden_alert added to alerts_summary.yaml aggregator trigger list. Pond pump automation 1742999668407 deleted from automations.yaml (replaced with migration comment). update_device_uptimes_group + device_restart_info_alert_active confirmed already absent from automations.yaml (HA reformatted file on last reload). Both superseded by alerts_network.yaml pipeline. automations.yaml now 1,968 lines, 9 active automations. GARDEN_CONTRACT.md created. ALERTS_CONTRACT.md updated: 12→13 files, garden domain pipeline audit added, aggregator trigger list corrected. ⚠️ RESTART REQUIRED — alert.garden_alert is an alert: entity; alert: entities cannot be reloaded (CODING_STANDARDS Reload vs Restart Rules). Binary sensor + context sensor will load on template reload; alert entity only activates after full HA restart.*
*2026-04-29 session: TASK 1 M1 entertainment mode — lighting_entertainment.yaml populated (4 automations: button→boolean, boolean ON→scene, boolean OFF→evening restore gated on civil_night, 06:00 daily clear). All wired to input_boolean.entertaining_mode (context_presence.yaml) — same concept as entertaining guests; duplicate input_boolean.entertainment_mode removed from lighting_helpers.yaml after user clarification. kids_bedtime_week + kids_bedtime_weekend: entertaining_mode guard added via choose block — if ON, skip pool light; turn off other 4 lights individually. TASK 2 energy saving — M2 helpers fixed (threshold 25%, icon mdi:lightning-bolt-outline); M3 already done in prior micro-session; morning_wake_lights_on now clears energy_saving_mode (lighting_morning.yaml). TASK 3 IDS stub: security_alarm.yaml created — zero IDS refs found in automations.yaml; provisional entity interface documented; IMP-IDS01 closed. TASK 4 water borehole gate: load_control_borehole_enabled added to binary_sensor.water_refill_allowed condition (water_templates.yaml:299); borehole_enabled attribute added. TASK 5 load shedding migration: announce_load_shedding_stage + load_shedding_warning_15 + load_shedding_warning_2hr migrated to packages/load_shedding/load_shedding_automations.yaml (fixes: notify.STD_* → script.notify_system_event, whatsapp removed, dead notify.mobile_app_ap_0223_1001 removed); blocks commented out in automations.yaml with migration banner; ha core check passed. Inverter Scene Switcher deferred to power session. TASK 6 automations.yaml full audit: AUTOMATIONS_AUDIT.md created in docs/. 12 active automations remaining (was 3,836 line file, now 2,915 lines). Power session owns 7 automations; geyser session owns 2; 3 ready to migrate now. Dead entity flags: 8 automations still reference sensor.inverter_power.*
*2026-04-29 session: Camera health alert implemented — alerts_camera_health.yaml created (⚠️ requires HA restart). Monitors video loss (binary_sensor.camXX_videoloss) and motion staleness (sensor.camXX_last_seen_seconds > 8h outdoor / 24h indoor, 08:00-20:00 only, only when security_system_enabled ON). One domain alert.camera_health, per-camera fault detail in sensor attributes and notification. Telegram mirror via automation → script.notify_security_event. alert.camera_health added to alerts_summary.yaml aggregator trigger list. Active faults: cam01 (NO VIDEO), cam04 (NO VIDEO), cam12 (video loss — cam11_back_pond entities; binary_sensor.cam11_back_pond_videoloss ON). cam12 entity naming confirmed: NVR channel is Cam12-Back-Pond but HA config uses cam11_back_pond IDs. Fix for all three: draw motion detection areas + check NVR channel signal in Hikvision NVR UI.*
*2026-04-29 session: BUG-L10 FIXED — kids_bedtime_week + kids_bedtime_weekend: added continue_on_error: true to both script.notify_lighting_event calls (pre-notification and post-scene). Root cause of bedtime failure on 2026-04-28 night: T2 fix introduced illegal {% if %} Jinja2 in notify_power_event.yaml + notify_security_events.yaml; those files caused script load error at reload time; when notify_lighting_event was called (no continue_on_error), HA may have been in a partial recovery state causing the call to fail, stopping the automation before scene.turn_on. NOTE: DB query earlier in session incorrectly showed switch entities as unavailable — this was historical data from April 24-25 outage; recorder does not write new entries while state unchanged after recovery. Sonoff integration IS working (confirmed via UI screenshot). BUG-L10 entry corrected from Sonoff-outage to notification-hardening issue.*
*2026-04-29 session: BUG-L10 found — Sonoff integration offline since ~2026-04-24. DB query confirmed switch.front_house_security_light, back_house_security_light, entrance_down_lights, dining_room_light, pool_light_switch all showing only "unavailable" state since April 24 (no on/off transitions recorded). Kids bedtime automation (kids_bedtime_week/weekend) condition: or check uses state:"on" — all unavailable switches evaluate to false, condition fails after setting bedtime_mode, scene never fires. Physical lights stayed on in last relay state. Root cause: Sonoff (EWElink) cloud integration down — check Settings → Integrations → Sonoff. Both switches are sonoff platform, config_entry 01JHTXM2, device 1001e1f931. LIGHTING_CONTRACT.md updated: L05/L07/L08 marked fixed; scene_kids_bedtime inventory corrected to 5 lights; helper inventory pruned (kids_bedtime_enabled + patio_second_wake_time removed); entertaining_mode entity name clarified (input_boolean.entertaining_mode exists in context_presence.yaml; button name mismatch "entertainment" vs "entertaining"). M1 planning note in GROUP M corrected — no new boolean needed, wire button to existing input_boolean.entertaining_mode.*
*2026-04-29 session: Recovery mode caused by illegal % token in notify_power_event.yaml line 198 and notify_security_events.yaml — both had {% if sev == 'critical' %} blocks used to conditionally include the inline_keyboard YAML key inside a data: mapping. HA's YAML parser sees {% at structural YAML level (where a key would appear) as an illegal % token and enters recovery mode. Fixed both files by splitting the Telegram notify.send_message into a choose block: critical branch includes inline_keyboard, default branch uses disable_notification. Both files verified correct: critical branch → inline_keyboard present, default branch → disable_notification present, no {% %} at key level. CODING_STANDARDS Rule 6 added: never use Jinja2 block tags to conditionally emit YAML keys — use choose: branches instead. Rule also added to pre-commit checklist.*
*2026-04-29 planning: BUG-L09 gap analysis completed. Entertainment mode: scene.scene_entertainment_mode already exists (pool light + patio + entrance_down + dining); missing are input_boolean.entertainment_mode, a button→boolean automation in lighting_entertainment.yaml (currently empty), and an entertainment-mode guard in both kids_bedtime automations which unconditionally turn off switch.pool_light_switch today. Energy saving mode: input_boolean.energy_saving_mode missing entirely; lighting_energy_saving.yaml empty; architecture decision — power system owns the SOC/orchestrator trigger (power_automations.yaml); lighting side only consumes the boolean. Manual override buttons exist (energy_saving_mode_on/off). Implementation captured in Group M (M1/M2/M3). Not yet implemented.*
*2026-04-28 session 3: T1 fixed — telegram_bot.send_photo added to notify_security_events.yaml after Telegram text message, guarded on img not none. T2 fixed — inline_keyboard added for critical severity in security (→ /ack_security_alert → alert.security_alert off) and power (→ /ack_power_alert → alert.power_alert off); callback automations added to respective notify files. T3 fixed — disable_notification: sev==information added to all 6 Telegram send_message calls. T4 fixed — push.interruption-level: critical (iOS) + channel: alarm + ttl/priority (Android) added to all 6 STD_Critical notify calls. BUG-WEA01 confirmed already fixed (checkbox not updated). BUG-CORE01 fixed — ha_events_per_second removed (was returning total sensor count not rate). BUG-CTX02 fixed — bedtime_mode moved to lighting_helpers.yaml, bedtime_time deleted, context_schedules.yaml deleted. BUG-L07/L08 fixed — kids_bedtime_enabled + patio_second_wake_time removed (no consumers). Only BUG-L09 remains open.*
*2026-04-28 session 2: BUG-INF01 fixed — removed dangling }} from binary_sensor.printer_cartridge_low. BUG-NET03 fixed — packet loss variables removed from WAN health score (ping_sum_5min is latency sum not pass count; formula was subtracting ms from integer). BUG-NET01 fixed — unifi_cpu_5m_max availability was referencing non-existent sensor.unifi_cpu → corrected to unifi_gateway_cpu_utilization. BUG-NET02 fixed — unifi_memory_5m_max availability was self-referencing → corrected to unifi_gateway_memory_utilization. BUG-L02 fixed (correctly this time) — main_entrance_light removed from scene_night_away (it stays ON overnight as deterrence with boundary lights), explicit switch.turn_off added to morning_wake_lights_on action so it turns off with the morning routine. BUG-L01/L03/L04/L05/L06 confirmed already fixed in code — checkboxes updated. BUG-BKP01 fixed — github.yaml failure notification routed through script.notify_system_event (warning severity bypasses quiet hours); success notification removed (5AM noise). IMP-IDS01 noted as intentionally deferred.*
*2026-04-28 session: number.inverter_battery_capacity TemplateError fixed — |int with no default throws when either solarman entity is unavailable at startup. Fixed |int → |int(0) for both inverter_1 and inverter_2 capacity in the UI template config entry (entry_id: 01JKJES2XR88FXQ8TZHST4YA46). Note: this fix was documented as done in the 2026-04-22 session C notes but did NOT persist — storage entry still showed modified_at: 2026-01-15 when re-examined. Root cause likely: HA rewrote the storage file after the edit (config entry was reloaded via API which triggers HA to overwrite the file from its in-memory state). Fix re-applied and verified — entity reads 600.0 Ah immediately. POWER_CONTRACT updated with UI-created template number section and re-application warning.*
*2026-04-23 session (performance): Recorder excludes expanded in configuration.yaml — added sensor.wan_*_5min_max, sensor.udm_*_5min_*, sensor.average_night_consumption, sensor.house_outdoor_power_clean, sensor.prepaid_drift_rate_per_day. Root cause: statistics/filter platform sensors recalculate on every source entity update (no minimum interval configurable); unifi_*_stats and *_ping_* were already excluded. Recorder excludes only stop DB writes — recalculation continues (negligible CPU for arithmetic templates). OPEN BUG: sensor.average_night_consumption has sampling_size: 20 bug — at solarman 5s poll rate, effective window is ~100s not 24h. Fix: throttled intermediary sensor (5-min trigger) + sampling_size: 288 → true 24h mean. Requires configuration.yaml reload → ⚠️ full HA restart required.*
*2026-04-22 session (power Session E): E1 — geyser switch rename confirmed done; one missed reference (load1_switch in dashboard_operations line 827) fixed. E2/P4-1 — sensor.prepaid_net_position_this_month is always available (no availability guard, all inputs use float(0)); returns 0 at month start before utility meters accumulate; no broken dependencies. P4-3 — sensor.prepaid_balance_confidence added to prepaid_core.yaml: 0-100% graduated score on drift magnitude + -2%/day penalty >30 days since last meter verification (max 30% penalty). P4-4 — input_number.prepaid_drift_threshold (default 2%) added to prepaid_helpers.yaml; prepaid_auto_reconcile automation added to prepaid_core.yaml: triggers on prepaid_meter_lifetime_import change, condition drift>threshold, calls script.prepaid_realign_offset + script.notify_power_event.*
*2026-04-22 session (power Session D — fixes): sensor.pool_pump_run_hours_today history_stats never loaded — history_stats platform sensors in packages silently fail when the source entity (switch.pool_pump_switch) has no recorder history at load time. Removed history_stats definition from power_state.yaml; must be created as UI Helper (Settings → Helpers → History Stats). sensor.pool_pump_solar_headroom orphaned entity removed from core.entity_registry (stale entry from before rename to solar_available_surplus, was showing cached -295W). POWER_CONTRACT updated to document UI-only history_stats requirement.*
*2026-04-22 session (power Session D — continued): Solar surplus metric corrected — solar_export_potential reads near-zero while battery fills (subtracts battery charging). Replaced with sensor.solar_available_surplus = pv_power - load_power (single shared metric for all optional loads; each appliance checks its own threshold). Pool pump thresholds: turn on >1200W, turn off <800W (hysteresis). sensor.pool_pump_control_status added — human-readable reason string for dashboard. All pool pump automation notifications now route through script.notify_power_event (HA app + Telegram mirror). Default branch fires hourly when solar active + pump off — Telegram trace of why pump didn't run. Entity renames: switch.pool_pump_switch_1 → switch.pool_pump_switch and switch.geyser_heat_pump_switch_1 → switch.geyser_heat_pump_switch across all YAML + 3 dashboard storage files (operations/overview/testing).*
*2026-04-22 session: BUG-A09 fixed — sensor.critical_sensor_health_alert_context devices attribute had inline Jinja2 `# comment` on same line as {{ ns.items[:20] }} inside a YAML > block scalar. YAML does not treat # as a comment inside block scalars; Jinja2 appended the literal text to the list output, making it unparseable as a list. alert_device_entities aggregator check `devs is not string` silently failed — critical sensor health never appeared in global alert summary. Fixed by converting to {# Jinja2 comment #}. Same fix applied to sensor_health_overview_context. Prepaid Strategic Top-Up Suggestion stale HA Repairs error: automation YAML already uses script.notify_power_event (correct) — stale Repairs notification, dismissed via UI. Note: automations.yaml lines 2318+2368 (grid usage warning) still use notify.std_warning (lowercase) — DO NOT TOUCH (legacy automations.yaml).*
*2026-04-22 session (power Session D — pool pump migration): D0 pre-flight: sensor.load_shedding_stage → actual entity is sensor.load_shedding_stage_eskom (attr stage: int); sensor.load_shedding_next_start → use sensor.load_shedding_minutes_remaining; sensor.battery_runtime_estimate → use sensor.ss_soc_battery_time_left. D1: 5 pool helpers added to power_helpers.yaml (pool_minimum_run_minutes/pre_shed_battery_threshold/target_hours_summer/_winter/pool_pump_last_on). D2/D3: pool_pump_run_hours_today (history_stats) + pool_pump_continuous_run_minutes + pool_target_run_hours_today template sensors added to power_state.yaml. D4: pool_pump_solar_control added to power_automations.yaml — solar-surplus-aware, load-shedding-gated, minimum-run-time-protected, season-aware target hours. D5: 4 legacy pool pump automations (IDs 1742227789739/1742294033785/1742294477609/1742295582190) deleted from automations.yaml; migration comment left at deletion point. D6: POWER_CONTRACT pool pump section added.*
*2026-04-22 session (power Session C): sync_power_groups.py: (1) @time_trigger("startup") added so load groups populate on HA start without manual service call; (2) from homeassistant.helpers import entity_registry blocked (not in pyscript ALLOWED_IMPORTS) — replaced er.async_get(hass) with hass.data.get("entity_registry") (HassKey is str subclass, plain string lookup works). battery_runtime.yaml: 6× |int → |int(0) on program_N_soc lookups (unavailable at startup caused TemplateError). energy_state.yaml: removed device_class: energy from battery_energy_available (incompatible with state_class: measurement — device_class: energy requires total/total_increasing; this is a point-in-time Wh value). .storage/core.config_entries: number.inverter_battery_capacity UI-template fixed |int → |int(0) for both inverter capacity entities (cannot edit via YAML — lives in config entries storage). C3 pool pump research: 4 automation blocks found in automations.yaml lines 2222–3174 — IDs 1742227789739/1742294033785/1742294477609/1742295582190 (Turn Off PM / Turn On Early AM / Turn On AM 11-1 / Turn On PM 1-4). All 4 use dead sensor.inverter_power, notify.STD_Information, device_id-based control, and indirect solar mode via input_select.inverter_solar_appliance_mode_helper. Load shedding conditions partially disabled (enabled:false). Research only — no changes.*
*2026-04-22 session: BUG-N11 fixed — notify_power_event.yaml all 3 severity branches used {{ title }} with no default guard; HA logged "title is undefined" on every render. Added | default('Notification') to match notify_water_events.yaml pattern.*
*2026-04-21 session (power + load control): energy_core.yaml — grid_to_house_power now subtracts grid_to_battery_power (was assigning all grid import to house); solar_to_battery_energy_today and grid_to_battery_energy_today template sensors removed — both already exist as UI utility meters (created 2026-03-20, would have created _2 duplicates); grid_used_by_house_today now uses grid_energy_import_today minus grid_to_battery_energy_today (UI meter). CODING_STANDARDS.md Rule 4 added: always check .storage/core.entity_registry before creating any entity in YAML. load_control.yaml created: input_boolean.load_control_geyser_enabled/pool_enabled/borehole_enabled + 3 automations that turn off running devices when disabled. recorder purge_keep_days set to 90. pyscript/power_snapshot.py added.*
*2026-04-21 session: Removed 3 UI-created storage duplicates causing startup errors. counter.water_borehole_faults_today and counter.water_borehole_faults_this_week removed from .storage/counter (both already defined in water_helpers.yaml since 2026-04-16). input_boolean.notifications_enabled removed from .storage/input_boolean (already defined in notifications_helpers.yaml since 2026-04-15 as BUG-N03 fix). Requires HA restart to take effect.*
*2026-04-21 session (cont): BUG-NET04 fixed — sensor.network_device_down_alert_severity and sensor.network_alert_context devices attribute both only filtered state=='off', missing 'unavailable'. group.network_devices (all:true) goes 'off' when any member is unavailable, correctly firing the alert binary sensor, but severity evaluated down=0 → none → context stayed normal → global summary showed nothing. Fixed both selectors to ['off','unavailable']; unavailable devices now labelled "Unavailable" in alert detail.*
*2026-04-21 session: UniFi AP MAC addresses updated in presence_core.yaml for new/replaced hardware (Bar, Office, Garage, Lounge, Bedroom Zone). AP monitoring migrated from ping platform (UI-configured binary_sensor.ap_*_ping) to UniFi controller state (binary_sensor.ap_*_connected wrapping sensor.ap_*_state == 'connected') in network_helpers.yaml. group.network_devices updated. Old ping entries for APs can now be removed via UI. Kitchen AP not yet adopted — excluded.*
*2026-04-20 session: Disabled two enabled notify.whatsapp steps in Load Shedding Inverter Scene Switcher (automations.yaml Stage 4+6 branches). Decided to drop WhatsApp entirely — no viable free replacement. Added Group T (Telegram enhancements): camera snapshot on security events, inline keyboard ack buttons, disable_notification for info level, mobile alarm sound for critical.*
*Previous: Three-batch session — BUG-A06 door unification, water alerts expansion, presence pipeline + triple-fire dedup*
*2026-04-15 session (Group D+E): D1 fixed — security_lighting_allowed entity typos corrected; D2 fixed — cam06 last event now reads cam06 not cam09; D3 fixed — cam10_pool_bar_motion_images renamed to cam10_pool_bar_images; D4 fixed — unique_id added to security_correlation; D5 fixed — presence_test_arrival removed; E1 fixed — from: constraints added to water_borehole_no_rise_protection + water_reconcile_cycle_state; E2 fixed — for: stability window added to water_refill_visibility_guard; E3 verified — water_refill_allowed gate confirmed correct.*
*2026-04-13 session: BUG-N08 fixed (escaped_title/message defaults); BUG-N09 fixed (MarkdownV2 . and ! missing from escape chains — caused Telegram BadRequest on all float values); alerts_power template fix; camera snapshot continue_on_error; automations max_exceeded:silent; Lovelace template errors; prepaid weekday trigger fixed (sunday→sun list); BUG-N10 fixed (Group F — 8 missing from:/not_from: guards across notifications, lighting, power)*
*2026-04-14 session: BUG-N01 fixed (continue_on_error on Telegram in notify_water_events); sev default guard fixed in notify_water_events; prepaid_days_since_verification none guard fixed; Lovelace: config.entity replaced with config.get()/static styles on all alert+power cards; door sensor last_changed guarded against None at boot (6 cards); continue_on_error added to all 10 direct Telegram calls across 5 files (solar_forecast x3, water_reporting x1, water_tank_refill_control x2, github x2, lighting_bar_presence x2) — fixes ServiceNotFound on restart*
*2026-04-15 session: sensor.load_visibility_score ZeroDivisionError fixed — float(1) default didn't guard against actual 0 value; added if total > 0 else 100 guard (power_templates.yaml); sensor.printer_cartridge_state UndefinedError fixed — min on empty sequence when printer is off; added rejectattr+list+none guard (office_helpers.yaml); Lovelace dashboard_operations: 6 door/gate card_mod CSS templates still had unguarded states.binary_sensor.X.last_changed — secondary text was fixed in prior session but style template was a separate code path; fixed with entity-is-not-none fallback to now() on all 6 (main_gate, garage, front_security_gate, front_door, lounge_door, bar_door)*
*2026-04-15 session (Group C): C3 fixed — notifications_helpers.yaml created, notifications_enabled now in YAML (BUG-N03); C4 fixed — 4 temperature route automations migrated from direct notify.STD_* calls to script.notify_system_event with quiet hours + Telegram (BUG-N04); C5 fixed — 4 presence unknown AP automations migrated from direct STD_Information to script.notify_presence_event with quiet hours (BUG-N05); C6 verified clean — route_device_power_alert already removed (BUG-A04). C2 verified — water_notifications.yaml:158 already uses counter.water_borehole_faults_this_week (correct name). A3/A4 verified — security_core.yaml, security_logic.yaml, lighting_departure.yaml already use binary_sensor.low_trust_present and binary_sensor.staff_on_site throughout. All were already applied; docs updated to reflect.*
*2026-04-16 session (lighting audit): 5 bugs fixed — (1) scene_night_away missing entrance_down_lights (added); (2) night departure now applies scene_morning_routine_off as second scene to catch evening routine lights; (3) departure trigger got from:"off" restart guard; (4) arrival scenario-1 binary_sensor.anyone_home→anyone_connected_home (entity didn't exist); (5) morning_wake_lights_on got time+cam14 fallback triggers with trigger-id weekday guard; scene_kids_bedtime got entrance_down_lights+dining_room_light; kids bedtime week+weekend got 30s confirmation notify + cancel button (input_button.kids_bedtime_cancel added to helpers).*
*2026-04-16 session: BUG-A06 fixed — sensor.doors_open_alert_severity deleted; sensor.door_alert_context now unified single source with full tiered logic (tier 1: gate/front security gate; tier 2: front door/garage; tier 3: lounge/bar). Two new input_numbers added: tier3_door_evening_start (21h) + entry_door_night_escalation_minutes (5 min). Water: water_tank_full_notification dead trigger fixed (binary_sensor.water_tank_full_depth). Borehole fault counters defined in YAML (were undefined). Fault escalation pipeline added: L1 notification, L2 alert (>=3), L3 critical (>=5). sensor.water_refill_blocked_reason added (priority-ordered block reason). Presence B1: full alerts_presence.yaml pipeline implemented (unknown AP + occupancy anomaly). Triple-fire dedup: binary_sensor.security_intruder_active + zone conditions added to all three intruder automations. alerts_summary.yaml trigger list updated for water borehole faults + presence_alert.*
*2026-04-21 session B (power entity fixes + migration): Session A: battery_energy_available + average_night_consumption (statistics platform, mean of inverter_load_power, 24h/20 samples) added to energy_state.yaml; battery_night_survival formula fixed (×10 for 10h Joburg night); solar_forecast.yaml syntax error line 61 fixed + 3× sensor.inverter_power→sensor.inverter_load_power; grid_risk.yaml + alerts_system_health.yaml sensor.inverter_power fixed; power_core.yaml inverter_today_energy_import doubling workaround documented with full deprecation comment; pyscript/power_snapshot.py: hass.bus.async_fire→event.fire. Session B: power_automations.yaml created as power domain automation home; grid_status_monitoring + inverter_pwer_monitoring commented out in automations.yaml (fully superseded by alert.power_alert in alerts_power.yaml); inverter_alert_battery_soc_critical migrated as power_battery_soc_critical_alert — ssa_*→ss_* sensor fix + notify.std_critical→script.notify_power_event; alerts_power.yaml: 3× sensor.inverter_power→sensor.inverter_load_power; load_shedding migrated to new packages/load_shedding/ package (⚠️ restart required); load_shedding unique_id typo fixed (load_hedding_inutes_emaining→load_shedding_minutes_remaining); sensor.inverter_device_1_since_last_update renamed (unique_id fix); house_security_power group reference fixed (security_power_sensors→house_security_power_sensors); dashboard prog1_capacity fixed (number.inverter_1_program_1_charging→number.inverter_1_program_1_soc which exists in entity registry).*
*2026-04-15 session (door alerts): Gate notification split — alert.door_alert now skip_first:true (first fire at 5 min sustained open); notify_gate_opened automation sends warning severity immediately on open (bypasses quiet hours, befitting tier 1 perimeter); notify_gate_closed sends information severity on close (suppressed in quiet hours). Removes generic "list all doors" fallback from initial low-severity trigger.*
*2026-04-14 session (security): Inside camera arming schedule added (security_inside_cameras_arming automation — arms cam14/cam15 based on time+occupancy). New booleans: inside_cameras_armed (managed), inside_cameras_schedule_override (dashboard override). Master security_system_enabled now also gates camera capture automations (was only gating notifications). Motion debounce tuned: delay_on 1–2s on all outdoor cameras to filter rain/wind; delay_off increased on rear cameras to 45s. Dashboard cards recommended: security_system_enabled + inside_cameras_schedule_override on security control card. cam01/cam04/cam07 snapshots still blank — root cause: no motion detection areas drawn in Hikvision NVR. Fix is Hikvision UI only, no HA changes needed.*
*2026-04-15 session (security classification audit): warning threshold raised to 75% (from 50%) in sensor.security_threat_level; security_event_router — added 5-min cooldown (checks last_security_event >300s), added to: "on" guard to suppress off-transition fires, now updates last_security_event after each notification; cam04 carport added to sensor.security_movement_path (was returning none for carport-only events). Architecture — all classification state names, entity names, and notification interface preserved intact for new AI camera integration. Known remaining: triple-fire from security_grounds_motion + security_rear_grounds_motion + security_house_motion all triggering on correlation→intruder simultaneously (low priority — 5-min cooldown on event_router is the primary anti-spam gate).*
