---
name: reference_live_data_model
description: "How real LIVE data flows to the cockpits on otto-q-core — the production_live run + cron model, the read endpoints/tables, and why the energy series is empty when idle"
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

**How live cockpit data works on otto-q-core (`gxdrcyphqjzjsuhxuqtg`), reverse-engineered 2026-06-26.**

- **The heartbeat:** pg_cron job `ottoq-depot-tick` (jobid 1, `*/2 * * * *`, active) runs `ottoq_cron_tick()` → calls `ottoq_world_advance()` (twin physics) + edge fns `ottoq-orchestrate-tick` + `ottoq-wave-admit` (these hit cuOpt etc. = real cost each tick).
- **`ottoq_world_advance()`** advances the SINGLE most-recent run with `run_by='production_live' AND status='running'`, using REAL elapsed minutes (`sim_clock_current = now()`, clamped 0.25–10 min). So a production_live run's snapshots are stamped at real wall-clock → they show up in real-time `/energy/history` and the `site_energy_snapshots` reads. It tracks real time-of-day (start it in the evening to demo the overnight wave + peak-shave; midday = low charge).
- **No production_live run = dark cockpits.** If none is running, world_advance no-ops (`RAISE WARNING 'no running production run'`), no fresh snapshots, and `/energy/history` returns `series:[]` (empty). As of 2026-06-26 the feed was idle (last snapshot ~04:18). To light it up: `ottoq_sim_start_run('normal_day', now(), 1, <seed>, 'production_live')` on the flagship (one-running-run-per-depot constraint; complete any existing running run first). This re-seeds the flagship fleet. It's an ongoing-cost decision (cron hits paid edge fns every 2 min) → Chase's call, not unilateral.

**What the cockpits read (all anon-readable today; OQ-7 RLS would change this):**
- Real tables via PostgREST: `site_energy_snapshots` (grid_import_kw, total_ev_charging_kw, building/lighting_load_kw, bess_output_kw, billing_period_peak_kw, current_rate_per_kwh, current_tariff_label, timestamp), `ottoq_energy_commands` (charge_cap_kw setpoint + reason.demand_target), `waves` (wave_code, scheduled_date, vehicle_count, status, windows). NOTE: `ottoq_bess_units` is RLS-BLOCKED to anon (returns `[]`) — get battery SoC from the `ottoq_nl_status_brief` RPC instead.
- Real gateway endpoints (`/functions/v1/otto-q-api/api/v1/...`): `/fleet/summary`, `/ai/depot-projection?depot_id=`, `/energy/history?range=24h&depot_id=`, `/fleet/schedule-intelligence?horizon=`. The gateway `ottoQFetch` unwraps `{data,meta}`.
- Frontier RPC/tables (Pulse + Orchestra `use-ottoq-frontier.ts`): `ottoq_nl_status_brief`, `ottoq_policy_params`/`ottoq_policy_set` (energy dial), `ottoq_cil_adoptions`.

**Cockpit real-data cleanup (2026-06-26, PRs):** Pulse [field-ops#4] (energy tab: real grid series + snapshot fields + wired emergency-shutdown; waves: real `waves` table, dropped fake $47.20/180kW summary). Orchestra [a4359174#10] (rebuilt fake EnergyDashboard on `/energy/history`; deleted dead mock-agent layer tool-executor/predictive-engine/automation-rules; Index energy tiles real). Both build clean. Remaining follow-ups (need telemetry the API doesn't expose): vehicle GPS (Orchestra jitters around city center), per-vehicle odometer; Pulse's other hooks (vehicles/stalls/incidents) still fall back to mock arrays on API error. See [[feedback_ui_audit_branding]], [[project_energy_e4_charging_fix]].
