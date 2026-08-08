---
name: reference_db_surgery_lessons
description: "Hard-won Postgres lessons on otto-q-core: NEVER DROP a function before capturing its source (recovery = supabase_migrations.schema_migrations + replay later splices); enum ::text casts silently defeat partial indexes (2,449ms -> 0.83ms)."
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-30T00:51:06.344Z
---

**⚠️ RULE -1 — TWO GRANT/SEARCH_PATH TRAPS THAT MAKE A MIGRATION LOOK SUCCESSFUL WHILE DOING NOTHING (learned 2026-07-29, both hit live).**

**(a) `REVOKE ... FROM anon` is a NO-OP while PUBLIC holds the grant.** Postgres grants `EXECUTE` to `PUBLIC` on every new function by default. The ACL reads `=X/postgres | postgres=X/postgres | service_role=X/postgres` — **the empty grantee IS `PUBLIC`**. So `anon` holds EXECUTE *through* PUBLIC; revoking from `anon` by name removes nothing, the statement succeeds, and `has_function_privilege('anon',…)` still returns true. Must be `REVOKE ... FROM PUBLIC` then `GRANT ... TO service_role`. This concealed a live exposure: **all 48 SECURITY DEFINER world-state writers were callable with the publishable anon key**, including `ottoq_sim_seed_fleet` (blanket-updates every vehicle, clears every stall) and `ottoq_purge_prior_runs` (deletes the black box). Any plan saying "revoke cross-schema grants" hits this trap.
⚠️ Also: the writer count is **48, not 47** — `ottoq_sim_seed_fleet` has TWO overloads sharing one name. Dedupe on full signature, never on `proname`, or the most destructive routine in the set silently stays granted.

**(b) A routine with a `SET` clause CANNOT `COMMIT`.** `ALTER ROUTINE … SET search_path` on a procedure that uses transaction control fails at runtime with `invalid transaction termination`. Blanket-pinning all 396 routines broke `ottoq_demo_metronome` (the Start button's tick engine) and both `ottoq_retention_purge_worker` overloads; cron job 12 failed two consecutive beats before I corrected it. **For routines that commit, use no SET clause and set the path inside the body instead:** `PERFORM set_config('search_path','twin, ottoq, public, extensions', false);` — legal alongside COMMIT and survives each commit.

**FORWARD-COMPATIBLE search_path pattern (applied to all 396 routines 2026-07-29):** pin to `twin, ottoq, public, extensions` BEFORE moving anything. Non-existent schemas in a search_path are silently ignored, so today everything resolves in `public` exactly as before; after a table moves to `twin` it is found there first with **no function edit**; and `public` must stay on the path because PostGIS lives there (`stalls.absolute_point`, `depots.geofence`, `origin_point`). Move day then changes no function bodies at all. See [[project_ottoq_twin_boundary]].

**⚠️ RULE -2 — PER-ITEM `prosrc LIKE` SCANS IN A MIGRATION ARE QUADRATIC AND CAN FLATTEN THE BOX (learned 2026-07-30, self-inflicted).** The twin-schema move ran one `SELECT count(*) FROM pg_proc WHERE prosrc LIKE '%public.<name>%'` PER candidate (~79 of them). Each was a ~21 s seq scan over every function body (log-captured plan) → ~25 min of saturated compute, 137-second checkpoint writes, pg_cron "job startup timeout" on every job, statement timeouts everywhere, MCP connector dead for many minutes. The migration itself COMPLETED correctly — the damage was the safety check, not the change. **Pattern: one pass, not N** — build all reference checks into a single scan (e.g. unnest the candidate list and join once against pg_proc), or precompute matches into a temp table. Also: on this box, treat ANY migration expected to exceed ~15 s as an outage risk — pause the run first and check `get_logs(postgres)` afterwards, not just the success flag.

**⚠️ RULE 0 — NEVER TRUST A SINGLE BEFORE/AFTER TIMING ON THIS INSTANCE (learned 2026-07-26).**
I measured `ottoq_record_event` at **7.155 ms** before dropping 8 unused indexes and **8.029 ms** after — i.e. it appeared to get SLOWER, which is causally impossible (dropping indexes cannot slow an insert). Both were single samples. Six consecutive identical probes then returned **5.835 / 5.345 / 4.954 / 3.497 / 3.499 / 4.198 ms — a 67% spread, monotonically warming.** The noise floor exceeds most effect sizes here because the working set does not fit in `shared_buffers` (224 MB), so timings are dominated by cache-miss luck — the same reason the tick itself varies 3×.
**PROTOCOL: ≥6 warm rounds per arm, compare MEDIANS, and if the arms' ranges overlap, report "below the noise floor" rather than a number.** Self-rolling-back probe pattern (leaves no junk rows):
```sql
DO $$ DECLARE t0 timestamptz; t1 timestamptz; n int := 200; BEGIN
  FOR i IN 1..20 LOOP PERFORM <fn>(...); END LOOP;          -- warm, untimed
  t0 := clock_timestamp();
  FOR i IN 1..n LOOP PERFORM <fn>(...); END LOOP;
  t1 := clock_timestamp();
  RAISE EXCEPTION 'per_call_ms=%', round(extract(epoch from (t1-t0))*1000/n, 3);  -- aborts => rollback
END $$;
```
**Corollary: `pg_stat_database.stats_reset IS NULL` means `idx_scan` counts cover the instance's ENTIRE lifetime — that IS strong enough evidence to drop an index, even when you cannot measure the speed win.**

**⚠️ RULE 1 — NEVER `DROP FUNCTION` BEFORE CAPTURING ITS SOURCE.**
I dropped `ottoq_sim_decide_and_dispatch(uuid)` (the decider on BOTH the demo and production tick paths) intending to recreate it with an extra parameter, without first saving `pg_get_functiondef`. `pg_proc` is the only live copy — once dropped, the source is gone.

**RECOVERY PROCEDURE THAT WORKED (2026-07-25):**
1. `supabase_migrations.schema_migrations` stores every migration's SQL in `statements` (text[]). Find the last FULL definition:
   ```sql
   SELECT version, name,
          (array_to_string(statements,E'\n') ~* 'CREATE OR REPLACE FUNCTION[^;]*<fn>') AS has_full_create,
          (array_to_string(statements,E'\n') ~* 'pg_get_functiondef')                  AS is_splice
     FROM supabase_migrations.schema_migrations
    WHERE array_to_string(statements,E'\n') ILIKE '%<fn>%' ORDER BY version;
   ```
2. Extract JUST that function's statement (don't replay the whole migration — it would revert OTHER functions to that date):
   `substring(s FROM position('CREATE OR REPLACE FUNCTION public.<fn>' in s) FOR position('$function$;' in substring(s FROM st)) + 10)` then `EXECUTE`.
3. **Replay, in version order, every later migration that SPLICED that function.** Anchored splices carry "already wired" skip-guards so they are safe to re-run. Missing this step silently loses features — here it would have dropped the ORCH-3 forecast attach and the reservation re-optimizer.
4. Verify by feature flags, not just length: `prosrc ~ '<symbol>'` for each layer, then a live smoke run.

**Corollary:** to add a parameter to a hot function, DON'T drop it. Either pass state another way (I stamped `ottoq_sim_runs.payload->>'tick_minutes_actual'` and had the callee prefer it), or capture the source first and rebuild in one transaction.

**⚠️ RULE 2 — `enum::text = 'literal'` SILENTLY DEFEATS PARTIAL INDEXES.**
`ottoq_sim_compute_charger_load_kw` had `cs.status::text = 'active'`. The partial index `idx_ocpp_sessions_active_run (depot_id, sim_run_id) WHERE status='active'::ocpp_session_status` could not match the cast predicate, so the planner fell back to `idx_ocpp_sessions_depot` and **filtered 28,963 rows on every call**. Called ~33×/tick by the EN.001 shield rule ⇒ ticks blew the statement timeout.
**Measured: 2,449 ms / 14,356 buffers → 0.83 ms / 11 buffers (~2,900×).** EN.001 max 12,266 ms → 173 ms; 45/45 ticks clean where runs previously died at tick 6.
Same cast fixed in `ottoq_sim_advance_site_energy` (runs every tick), `ottoq_reconcile_charger_states`, `ottoq_count_enacted_breaches`.
**Always `EXPLAIN (ANALYZE, BUFFERS)` a suspect function before blaming "a slow rule" — and grep for `::text` on enum columns first.**

**RULE 3 — don't trust a sub-agent's "could not find the code" claim.** An investigation reported EN.001's evaluator "lives in a TypeScript edge function outside scope, never found". It is an ordinary SQL function (`ottoq_eval_en_001_grid_capacity`), one `pg_proc` query away. Verify load-bearing claims directly.

**✅ TICK COST — ISOLATED (audit session local_f1c3569b, 2026-07-26). Receipt: `OTTOQ_TICK_COST_ISOLATION_2026-07-26.md`.**
**`ottoq_record_event` = 28ms/call × 60-120 calls/tick = the entire tick.** It is called from INSIDE every step, so each step only absorbs a slice and nothing looks slow individually — which is exactly why my per-function profiling found nothing. Two independent samples agreed to 0.25ms/call (archived 12.9s tick: 120 calls @ 28.04ms, self-times summed 13,626ms vs 12,920ms measured — accounts for the whole tick).
**ROOT CAUSE — index maintenance on a table that no longer fits in cache.** Controlled test in an aborted txn: insert into `ottoq_events` (**18 indexes**) = **4.03ms**; identical insert into an unindexed clone = **0.065ms** — **62×**. EXPLAIN: 3.54 of 4.13ms is I/O wait on 2 cache misses, because `ottoq_events` is **7.6 GB with 585 MB of indexes against 224 MB shared_buffers**, three indexes >100MB keyed on random UUIDs. Idle 4ms → under load 28ms. **Miss-count luck is what makes the tick vary 3×.** Reproduced on a fenced run with only ~21 active vehicles: 19.4/15.9/22.9/19.3/20.2/21.7s — so it is NOT a fleet-size problem.
**FIX (not yet A/B'd): 8 of the 18 indexes have NEVER been scanned once** (~110MB of pure write tax; one is a strict duplicate of a partial index that takes all the traffic). Restore DDL captured in the receipt.
**🚨 MY "metronome = 35.7% of DB time" WAS A MEASUREMENT ARTIFACT — RETRACTED.** `pg_stat_statements.track` is `top`, which rolls ALL nested work into the caller. That 35.7% IS the tick cost seen from outside, **not a second 35% to reclaim.** Lesson: with `track=top`, never read a top-level CALL's total as its own cost.
**ANOTHER VACUOUS ENUM GUARD (same family as Rule 2):** `ottoq_sim_auto_charge_assign_tick` filters `cs.status::text = 'in_progress'` — the enum has only `active/completed/faulted/cancelled`, so it matches **ZERO rows, always**. The occupancy check never fires (strong candidate for the HW.004 occupied-stall symptom) AND the `::text` cast defeats the partial index → 54k-row scan per vehicle per tick, **1.47 BILLION lifetime tuples read.**

**⏱️ (superseded) TICK-COST PROFILE (2026-07-26) — my three refuted hypotheses, retained because they are still worth not re-testing:**
Tick = **7-22s, ~3x variance**. `ottoq_sim_decide_and_dispatch` **27.6s** in one sample vs **<5s for the ENTIRE world sim** (telemetry 2.6s, energy 2.1s, comms 838ms, service_flow 242ms, visit_atoms 236ms, everything else <100ms).
**REFUTED — do not re-test:** (1) *the shield* — ALL rule evals summed to **2.7s across a 12-MINUTE window**; one 17.7s tick had **ZERO** evals; EN.001 now **2.6ms avg** (was 12,266ms — the enum-cast fix holds). (2) *the 20s-timeout `net.http_post` to ottoq-orchestrator-agent on the hot path* — pg_net is ASYNC, measured **119ms**; cuopt_refresh 241ms; orchestrator_trigger 226ms. (3) *metronome lock contention* — fencing (`run_by='cert_harness'`) changed nothing: 19.3s unfenced vs 7.0/10.3/22.1s fenced.
**Every named sub-part of decide_and_dispatch measures <300ms on settled state (sum 259ms).** Components fast, composite slow ⇒ something UNMEASURED dominates. **Top untested suspect: per-row TRIGGERS on vehicles/stalls writes** (`log_vehicle_state_change`, `sync_stall_occupancy`, `ottoq_vehicles_state_change`, `ottoq_stalls_state_change`, `ottoq_sign_event`, `ottoq_compute_event_hash`) — 116 vehicles × per-row event signing/hashing is invisible to function-level timing. Next probe: instrument `ottoq_sim_advance_tick_world` with `clock_timestamp()` around EVERY `PERFORM` (gated by a policy flag = real observability), and diff `pg_stat_statements` (ENABLED) immediately before/after ONE tick for statement-level attribution.
**⚠️ SEPARATE REAL COST: `CALL ottoq_demo_metronome` = 35.7% of ALL DB execution time** (17,291 calls, 3.6s mean, ~17h cumulative) + `ottoq_cron_tick` 7.5% ⇒ **~43% of total DB time is the two cron tickers**, much of it on runs nobody is watching. Verify idle-exit is cheap; consider lowering cadence/budget.
**`track_functions` is 'none'** so `pg_stat_user_functions` is empty and a session-level `SET` does not capture it; `pg_stat_reset()` is permission-denied. Use explicit `clock_timestamp()` instrumentation instead.

Links: [[feedback_confirm_every_logic_point]], [[project_full_service_visit_doctrine]], [[feedback_gap_ownership]], [[project_twin_realtime_clock]].
