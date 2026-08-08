---
name: project_db_outage_2026_08_05
description: "🚨🚨LIVE OUTAGE (2026-08-05, UNRESOLVED, awaiting Chase's decision): otto-q-core is resource-starved — won't hold a connection with ZERO runs active. 14 GB DB, ottoq_events 11 GB / 4M rows, 7 GB of it live TOAST. Purge only 2% done. Deleting rows will NOT shrink the file."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-05T12:33:44.814Z
---

# 🚨 STATE: OUTAGE, PAUSED BY CHASE. DO NOT RESUME WORK WITHOUT HIS INSTRUCTION.

He was offered 4 options (upgrade compute / grind the purge / pause / discuss) and **dismissed the
question**. Treat that as: **stop, wait, do not act.**

## THE SITUATION (all MEASURED)
- Supabase reports `ACTIVE_HEALTHY` at the platform level, but the DB **cannot reliably accept a
  connection with ZERO sim runs active**. Trivial indexed reads and even `get_advisors` time out.
- `public.ottoq_events`: **~4.05-4.33M rows · 11 GB** = heap 3,869 MB + indexes 751 MB +
  **TOAST 7,056 MB (~60%)**. Database total **14 GB**.
- ⭐ **The TOAST is LIVE jsonb payload, NOT bloat** — dead tuples were only 1.6%. That is ~1.7 KB of
  JSON per event across 4M events. **The root cause is that events are far too fat**, not that
  cleanup was skipped.
- Metronome wall-clock: healthy window **0.1-1.2 s**; degraded window **10-84 s**, every beat ending
  in a job-startup failure. `advance_tick_world` was measured at **48-65 s** (normal ~1 s).

## WHAT WAS DONE BEFORE THE PAUSE
1. **`trg_ottoq_auto_incident_report` DISABLED** (`tgenabled='D'`) — it ran a synchronous
   incident-report scan of the 11 GB table inside `decide_tick` on **every** `severity='safety_critical'`
   insert (**63 in 15 minutes**). Leave disabled; a queued replacement is future work.
2. Frozen run `6e087038` emergency-stopped cleanly (archived, blackbox_ready, depot reset).
3. **50,000 rows deleted (~2% of ~2.9M deletable).** Purge is NOT complete.
4. `CREATE INDEX idx_site_energy_snapshots_created_at` (it was seq-scanning 27k rows on every
   cockpit poll). `ANALYZE ottoq_events`.

## ⭐ THE ESCAPE HATCH — why retention NEVER worked (answers task #16)
`ottoq_events_block_mutation()` contains:
`IF TG_OP='DELETE' AND current_setting('ottoq.retention', true)='on' THEN RETURN OLD; END IF;`
⇒ **DELETE is allowed when the GUC is set.** `ottoq_purge_prior_runs` never sets it, so every purge
was refused, the error swallowed, and the job reported success. Use
`PERFORM set_config('ottoq.retention','on', true)` **inside a DO block** (`SET LOCAL` in autocommit
dies with its own implicit transaction).

## ⚠️ TWO HARD TRUTHS TO CARRY FORWARD
1. **Deleting rows will NOT shrink the 14 GB.** Plain VACUUM frees space *inside* the file; the
   deletable rows are the physically OLDEST = the head of the heap, and VACUUM only truncates
   trailing pages. `pg_database_size` stays ~14 GB. Real reclaim needs `pg_repack` or time
   partitioning + `DROP PARTITION` — and pg_repack needs headroom this DB does not have.
   **NEVER `VACUUM FULL`** (exclusive lock + needs free space equal to the table).
2. **There are NO unused indexes to drop.** All 10 on `ottoq_events` have `idx_scan > 0` (lowest = 7).
   The earlier "8 of 18 never used" finding no longer holds. An agent correctly declined to
   manufacture a target.

## 🔴 A NEAR-MISS WORTH REMEMBERING
The assess agent proposed keeping `event_seq >= 7,346,372`, claiming a clean write gap. **It was
wrong** — rows 7,346,369/70/71 share a timestamp INSIDE the 7-day window; that cutoff would have
deleted ~7,126 rows only 6.6 days old. The reclaim agent probed the boundary, found the true edge
(`event_seq <= 7,339,245`, 8.96 d old), and added a redundant `occurred_at` guard.
**Lesson: probe a deletion boundary directly; never trust a computed gap.**

## 📌 MY OWN CONTRIBUTION — stated plainly
This session added **~3 GB** to that table via repeated full-length certification runs
(session start: events 9.1 GB / DB 11 GB → now 11 GB / 14 GB). The work was real, but I kept filling
a container I already knew was nearly full. **Future certification runs must be budgeted against
remaining headroom, not treated as free.**

## ⏭️ WHEN CHASE RESUMES — recommended order
1. Get a working connection (his call: compute upgrade is the fast, reversible path).
2. Finish the retention purge to a 1-week keep window using the GUC hatch, batched.
3. **Fix the write volume** — 1.7 KB of JSON per event is the actual disease. `previous_state` writes
   were already removed from the state-change triggers earlier this session, so NEW rows are lighter
   (3,163 B vs 4,000 B measured); the remaining payload should be audited next.
4. Then, and only then, resume the forward-scheduling build.

## ⏸️ WORK PARKED MID-FLIGHT (nothing lost, all committed or recorded)
- **Migration 0005 needs a NARROW revert** — see [[project_forward_service_scheduling_doctrine]].
  Keep §3 (inspection wiring, proven correct). Remove §2's unconditional
  `exterior_soil_level := round(soil_index,3)`: it copies a *sensor*-soil counter (structurally capped
  ~0.30) into a *body*-soil field banded at 0.45/0.65/0.85, so it can ONLY ratchet a vehicle toward
  `ok`. It had already laundered **8 of 17** moved vehicles out of the wash bands with no wash bay.
  Also gate the `open_fault_codes := '{}'` / `worst_fault_severity := 99` writes riding the same array.
  Branch `fwd2-inspection-condition-resets` is pushed, **NOT merged**.
- Merged and good: 0002 (approval decider), 0003 (bay work recovery), 0004 (close ledger loop).

Links: [[project_db_capacity_ceiling]], [[project_forward_service_scheduling_doctrine]],
[[project_orchestration_build_2026_08_01]], [[project_demo_loop_ops]], [[reference_db_surgery_lessons]].
