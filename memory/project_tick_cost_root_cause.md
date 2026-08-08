---
name: project-tick-cost-root-cause
description: "2026-07-26 ISOLATED — the 7-22s tick is event-write amplification into a 7.6GB/18-index ottoq_events, NOT the shield; plus the session-scoped profiling technique that found it"
metadata: 
  node_type: memory
  type: project
  originSessionId: 5090c7cf-5d02-4eff-8d79-11286d165280
  modified: 2026-07-26T06:04:06.490Z
---

**Root cause of the 7–22 s tick (isolated 2026-07-26).** Full receipt:
`~/Desktop/OTTO-Q V1/OTTOQ_TICK_COST_ISOLATION_2026-07-26.md`.

The cost is in **no named step** — it is `ottoq_record_event` + siblings
(`ottoq_sim_emit_telemetry`, `ottoq_sim_emit_ocpp`), called 60–120×/tick from *inside*
every step at **28 ms/call** (two independent samples agreed to 0.25 ms). Cause:
`ottoq_events` = 7.6 GB / 2.75 M rows / **18 indexes** vs **224 MB shared_buffers**.
Measured: unindexed insert 0.065 ms → indexed 4.03 ms (**62×**), of which 86% is I/O
wait on cache misses. Idle 4 ms → loaded 28 ms, and the miss count being luck is what
makes it vary ~3×.

**8 of the 18 indexes have never been scanned once** (~110 MB of pure write tax).
`correlation_id` (107 MB) averages 1.04 rows/value — a random UUID per row, the worst
insert pattern. Second regime: `trg_ottoq_auto_incident_report` runs a 6-table
75-minute forensic replay *synchronously inside the insert*; its skip guard only covers
`benchmark%` depots, so it never applies to the flagship.

**Two corrections to earlier beliefs:**
- `pg_stat_statements.track` is `top`, which rolls nested work into the caller. So
  "metronome = 35.7% of DB time" is the *same tick cost seen from outside*, not a second
  problem to reclaim. See [[project_demo_loop_ops]].
- EN.001 / the shield is **not** the tick-timeout culprit (contradicts the note in
  [[project_full_service_visit_doctrine]]).

**Reusable technique — how to profile a tick.** `track_functions` and
`pg_stat_statements.track` are *settable per session* by the `postgres` role even though
it is not superuser (`ALTER DATABASE` is blocked, session `SET` is not). Because the SET
is session-scoped, **only your request collects counters and cron traffic never pollutes
the sample.** Snapshot `pg_stat_user_functions` before/after and diff; `self_time` excludes
callees, which is what makes an invisible per-row helper pop out. Driver scripts kept at
`OTTOQ_TICK_COST_ISOLATION_2026-07-26.md` §1. Watch the Supabase Management API rate
limit — 2 snapshot queries per tick will trip a 429 after ~100 ticks.

Also found: `ottoq_sim_auto_charge_assign_tick` has `cs.status::text = 'in_progress'`
but the `ocpp_session_status` enum only has `active/completed/faulted/cancelled` — the
predicate matches **0 rows always**, so the occupancy guard is vacuous (likely the real
HW.004 "occupied-stall proposals" cause) *and* it forces a 54 k-row scan per vehicle per
tick. Same failure mode as [[reference_db_surgery_lessons]].
