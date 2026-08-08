# 05 — The database

Everything below is verified against the live `gxdrcyphqjzjsuhxuqtg` instance on **2026-08-08**.

## 1. Vital statistics

| | |
|---|---|
| Project ref | `gxdrcyphqjzjsuhxuqtg` ("otto-q-core") |
| Postgres | 17.6 |
| Region | us-east-1 |
| Database size | **652 MB** (was 14 GB in the August 5 outage — recovered) |
| Tables in `public` | **461** ⚠️ (most are scratch/proof tables — see §7) |
| Functions | `public` 1088 (incl. ~744 PostGIS) · `twin` 71 · `ottoq` 55 |
| Migrations applied | **641**, latest `20260808182226` |
| Edge functions | 27 |
| Cron jobs | 5 (4 active) |
| Extensions in use | PostGIS, pg_cron, pg_net, pg_stat_statements |

## 2. Schema layout

| Schema | Contains | Reachable by `anon`? |
|---|---|---|
| `public` | Tables, entry points, the four policy arms (`ottoq_decide_tick`, `_greedy_tick`, `_fifo_tick`, `_manual_tick`), shared primitives (`ottoq_reserve_stall`), the PostgREST surface, PostGIS | Yes (PostgREST) |
| `ottoq` | **The decision layer.** 55 functions: calendar API, the decision halves of the 8 split functions, `ottoq_emit_vehicle_command`, `ottoq_validate_assignment`, `ottoq_react_to_refusals`, `ottoq_place_unplaced_vehicles` | **No — no USAGE grant.** Structurally unreachable from client keys and invisible to PostgREST. |
| `twin` | **The world engine.** 71 functions: `ottoq_sim_*`, `ottoq_world_advance`, the twin halves of the 8 split functions | **No — no USAGE grant.** |

All 396 routines are pinned to `search_path = twin, ottoq, public, extensions`. Non-existent
schemas in a search path are silently ignored, so this was forward-compatible: a table moved to
`twin` later is found there first **with no function edit**.

## 3. Cron jobs — read this before touching any of them

```
jobid  schedule       name                       command                                      active
  2    0 4 * * 0      ottoq-twin-ingest-weekly   SELECT ottoq_twin_ingest_refresh()            true
 10    */2 * * * *    ottoq-depot-tick           SELECT public.ottoq_cron_tick()               true
 11    0 8 * * *      ottoq-retention-nightly    CALL ottoq_retention_purge_worker(90,2000,…)  true
 12    * * * * *      ottoq-demo-metronome       CALL public.ottoq_demo_metronome(90)          true
 13    * * * * *      ottoq-cert-battery         SELECT ottoq_cert_battery_step()              FALSE
```

> 🚨 **NEVER disable job 12.** `ottoq_demo_metronome` **is** the run engine — it is what the START
> button's runs are advanced by. Disabling it silently stops every simulation while the UI stays
> green. This has bitten us.

- **Job 10** (`ottoq_cron_tick`) drives the `production_live` path: `twin.ottoq_world_advance()` plus
  edge functions `ottoq-orchestrate-tick` and `ottoq-wave-admit`. Those hit paid APIs — **it costs
  money every two minutes.**
- **Job 12** alternates DECIDE and FIRE beats. The FIRE beat is the one that fires cuOpt and commits
  so pg_net can actually transmit; the DECIDE beat is structurally incapable of getting a cuOpt
  answer in time and its fire was deliberately stood down.
- **Job 13** is the certification battery, deliberately inactive. Turning it on starts consuming the
  cert queue with `statement_timeout = 0`.

⚠️ **`ottoq_start_demo_run` aborts running non-production runs.** Do not start a demo run while cert
arms are executing.

## 4. The RPC surface

~290 `ottoq_*` / `twin_*` functions. The ones you will actually use:

### Run lifecycle
```sql
ottoq_start_demo_run(p_scenario text, p_speed numeric, p_days int, p_seed bigint)
ottoq_start_busy_run(p_speed numeric, p_days int, p_seed bigint)
ottoq_pause_run(p_sim_run_id uuid, p_reason text)
ottoq_resume_run(p_sim_run_id uuid)
ottoq_sim_stop_and_reset(p_sim_run_id uuid, p_reason text)   -- archives, then wipes
ottoq_archive_run(p_sim_run_id uuid, p_reason text)
ottoq_active_sim_run() / ottoq_depot_running_run(p_depot_id uuid)
ottoq_set_playback(p_sim_run_id uuid, p_mode text, p_speed_x numeric)
ottoq_sim_jump_forward(p_sim_run_id uuid, p_sim_minutes numeric, p_max_seconds numeric)
```

### Ticking
```sql
ottoq_demo_metronome(p_budget_s int)          -- the run engine (cron 12)
ottoq_cron_tick()                             -- production path (cron 10)
ottoq_sim_advance_tick(p_sim_run_id uuid)     -- world + decide, demo path
twin.ottoq_sim_advance_tick_world(uuid)       -- world physics only
ottoq_sim_decide_and_dispatch(uuid)           -- brain only; routes by policy
ottoq_manual_tick(uuid)
```

### Decision + shield
```sql
ottoq_decide_tick(uuid)                       -- the orchestrator (otto_q arm)
ottoq_build_decision_context(action_context, entity_type, entity_id, depot_id, now)
ottoq_build_decision_frame(depot_id)
ottoq_capture_decision_snapshot(run, tick_seq, depot, sim_clock)
ottoq_assert_snapshot_integrity(snapshot_id)
ottoq_evaluate_rules_for_action(action_context, entity_type, entity_id, context, …)
ottoq_shield_probe(...)                       -- NON-RAISING. Use this to ask "would this pass?"
ottoq_shield_and_log(run, tick, depot, actions, shadow)
ottoq_l1_override_authorized(rule_code, actor_type, actor_id, justification, …)
```

### Reading state (what the cockpits use)
```sql
ottoq_twin_snapshot(run)              ottoq_twin_depot_layout(depot)
ottoq_twin_run_context(run)           ottoq_twin_fleet_condition(run)
ottoq_twin_boot_manifest(run)         ottoq_twin_events_window(run)
ottoq_depot_cards(depot, fleet_operator DEFAULT NULL)
ottoq_vehicle_card(vehicle)
ottoq_nl_status_brief(run)            -- natural-language status; also the source of BESS SoC
ottoq_agent_board(run)                -- what Nemotron sees
ottoq_run_blackbox(run)               ottoq_blackbox_latest_run()
ottoq_approach_zone(vehicle, run)     -- + view ottoq_approach_band
```

### Scoring, certification, forensics
```sql
ottoq_score_run(run)                  ottoq_certify_run(run)
ottoq_ab_stats(metric, baseline, direction, scenario)
ottoq_ab_paired_summary(scenario)
ottoq_tick_invariance_arm / _metrics / _report
ottoq_entity_history(entity_type, entity_id, limit)
ottoq_causation_chain(event_id)       ottoq_replay_window(...)
ottoq_booking_provenance_audit(run)   ottoq_booking_provenance_summary(run)
ottoq_count_enacted_breaches(run)     ottoq_clock_fidelity(run, last_n)
```

### Policy knobs
```sql
ottoq_policy_get(run, key, default)
ottoq_policy_set(scope_type, scope_id, key, value, by)
-- catalog: ottoq_policy_param_catalog ; values: ottoq_policy_params (~800 rows)
```
Known dials: `deploy_peak_fraction`, `deploy_release_per_tick_cap`, `deploy_surge_catchup`,
`forecast_horizon_min`, `energy_reserve_shave`, `cuopt_solve_window_ms`, `cuopt_contention_min`,
`cuopt_debounce_s`, `approach_freeze_minutes`. **`deploy_floor_soc` is never auto-tuned.**

## 5. The tables that matter

**Decision + audit:** `ottoq_decisions` · `ottoq_rule_evaluations` · `ottoq_rules` (52) ·
`ottoq_events` (append-only) · `ottoq_decision_snapshots` · `ottoq_state_transitions` ·
`ottoq_rule_overrides` · `ottoq_incident_reports` · `ottoq_recommendations`

**Orchestration:** `ottoq_stall_bookings` (**the forward calendar**) · `ottoq_tick_reservations` ·
`ottoq_visit_needs` · `ottoq_vehicle_itineraries` · `ottoq_itinerary_legs` ·
`ottoq_vehicle_needs_card` (view, 66 cols) · `ottoq_vehicle_commands` · `ottoq_external_proposals` ·
`ottoq_ops_approvals` · `ottoq_deploy_log`

**World:** `vehicles` · `stalls` · `depots` · `ottoq_sim_runs` · `ottoq_sim_scenarios` ·
`ottoq_telemetry_packets` · `ottoq_ocpp_messages` · `ocpp_sessions` · `ottoq_weather_snapshots` ·
`ottoq_grid_snapshots` · `ottoq_solar_output` · `site_energy_snapshots` · `bess_snapshots` ·
`ottoq_vehicle_dispatches` · `ottoq_vehicle_wear` · `vehicle_need_profile` · `ottoq_site_structures`

**Calibration + variability:** `ottoq_calibration_datasets` / `_distributions` / `_profiles` ·
`ottoq_variability_cards`

**AI:** `ottoq_cuopt_fire_log` · `cuopt_invocation_log` · `ottoq_cuopt_deferrals` ·
`ottoq_model_routes` · `ottoq_energy_commands`

**Records:** `ottoq_run_archives` · `ottoq_schema_snapshots` (~6k restorable DDL objects) ·
`ottoq_fn_definition_backups`

⚠️ **`ottoq_events` is append-only by trigger** — `ottoq_events_block_mutation()` blocks
UPDATE/DELETE regardless of grants. **The escape hatch is a GUC:** the trigger allows DELETE when
`current_setting('ottoq.retention', true) = 'on'`. Retention purges silently failed for months
because `ottoq_purge_prior_runs` never set it and the error was swallowed. Set it with
`PERFORM set_config('ottoq.retention','on', true)` **inside a DO block** — `SET LOCAL` in autocommit
dies with its own implicit transaction.

⚠️ **`ottoq_purge_prior_runs` sweeps generically:** every `public.ottoq%` table having a
`sim_run_id` column. **Any new table with a `sim_run_id` column will be wiped at the next run start
unless you add it to the `c_keep_tables` exclusion list.** `ottoq_run_archives` was destroyed on its
very first purge this way. It also scans `information_schema.columns`, which includes **views** —
that is why `ottoq_vehicle_needs_card` uses `run_id` rather than `sim_run_id`.

⚠️ **`ottoq_stall_bookings.stall_id` is `ON DELETE CASCADE`.** Deleting a stall silently vaporises
ledger rows with no error. This is what halted migration 0010.

## 6. Security posture (deliberately partial, honestly stated)

**Closed:**
- All 31 blanket `"Service role full access"` policies — which were actually
  `FOR ALL TO PUBLIC USING(true) WITH CHECK(true)` — have been rescoped `TO service_role`.
  `remaining_blanket_policies = 0`.
- `anon` INSERT/UPDATE/DELETE revoked on `vehicles`/`stalls`/`depots`.
- `anon` has **no USAGE on `ottoq` or `twin`**.
- Destructive control-plane functions (`ottoq_sim_seed_fleet` ×2 overloads,
  `ottoq_purge_prior_runs`, `ottoq_benchmark_reset`, `ottoq_sim_run_scenario`) revoked **FROM
  PUBLIC**.
- `ottoq_vehicle_commands` hardened: RLS on, service-role writes, client read; anon can no longer
  forge acks (verified false).

**Open, knowingly:**
- Full multi-tenant RLS is **not** implemented. The user → fleet-operator binding in OrchestrAV is
  client-side only and spoofable.
- ~48 SECURITY DEFINER world-state writers are owned by `postgres` (`rolbypassrls = true`), so they
  ignore RLS and any REVOKE by design.
- Real auth/tenancy is deferred until post-funding (founder decision).

> ⚠️ **THE TRAP THAT HID ALL OF THIS: read `pg_policies.roles`, NEVER `policyname`.** A policy
> *named* "Service role full access" was `TO PUBLIC`. Thirty-one tables were world-writable while
> every audit that read names said they were fine.
>
> ⚠️ **`REVOKE … FROM anon` is a NO-OP while PUBLIC holds the grant.** Postgres grants EXECUTE to
> PUBLIC on every new function. The ACL entry with an empty grantee **is** PUBLIC. `anon` holds
> EXECUTE *through* PUBLIC, so revoking by name removes nothing, the statement succeeds, and
> `has_function_privilege('anon', …)` still returns true. You must
> `REVOKE … FROM PUBLIC` then `GRANT … TO service_role`.

## 7. Technical debt you will trip over

**461 tables in `public`, and roughly 300 of them are scratch.** Certification and proof runs write
evidence into ad-hoc tables — `phase9_*`, `p0019_*`, `fwd2_*`, `build3_*`, `cuopt_supply_*`,
`preimport_*`, `smoke*`, `p12_contaminated_run904_*`. They were named without the `ottoq` prefix
**on purpose**, so the generic purge would not sweep them. That was correct at the time; the result
is a schema you cannot navigate.

**This is real, safe, valuable cleanup work** — but it requires care: some are the only surviving
evidence for a published certification. Inventory before deleting, and get sign-off.

## 8. Rules for doing surgery on this database

These were each learned by breaking something. Follow them.

1. **NEVER `DROP FUNCTION` before capturing `pg_get_functiondef`.** `pg_proc` is the only live copy.
   Recovery is possible via `supabase_migrations.schema_migrations.statements` but it is
   archaeology, and you must replay every later migration that *spliced* that function or you
   silently lose features.
   *Corollary:* to add a parameter to a hot function, don't drop it — pass state another way.
2. **`enum::text = 'literal'` silently defeats partial indexes.** One such cast made a shield rule
   scan 28,963 rows on every call: **2,449 ms → 0.83 ms** once fixed (~2,900×). Another made an
   occupancy check match **zero rows forever** — that is how the A/B baseline ended up with
   unlimited chargers. **Grep for `::text` on enum columns before blaming "a slow rule."**
3. **Never trust a single before/after timing here.** The working set does not fit in
   `shared_buffers` (224 MB), so timings are dominated by cache-miss luck. Six consecutive identical
   probes once spread 67%. **Protocol: ≥6 warm rounds per arm, compare medians, and if ranges
   overlap report "below the noise floor" rather than a number.**
4. **Per-item `prosrc LIKE` scans in a migration are quadratic and can flatten the box.** One
   migration ran ~79 of them at ~21 s each — 25 minutes of saturated compute, 137-second
   checkpoints, cron job-startup timeouts, MCP connector dead. **One pass, not N.**
5. **Treat any migration expected to exceed ~15 s as an outage risk.** The working pattern:
   *surgical pause* (`UPDATE ottoq_sim_runs SET status='paused'` — instant, the metronome
   self-quiets, black box intact) → migrate with `SET LOCAL lock_timeout='8s'` → resume
   (`status='running', next_tick_due_at=now()`).
6. **Never blind-retry a timed-out migration — poll whether it landed first.** Three timeouts once
   occurred here; two had not landed, one had, and a pattern check prevented a double-apply.
7. **A routine with a `SET` clause cannot `COMMIT`.** `ALTER ROUTINE … SET search_path` on a
   procedure that uses transaction control fails at runtime with `invalid transaction termination`.
   Blanket-pinning all 396 routines broke the metronome. For committing routines, set the path
   **inside the body**: `PERFORM set_config('search_path','twin, ottoq, public, extensions', false);`
8. **The Supabase MCP migration runner mis-parses `$$`-quoted function bodies** ("syntax error at or
   near FROM"). Create functions via raw `execute_sql` with `$fn$` quoting, then `apply_migration`
   for the ledger entry.
9. **Deleting rows will not shrink the database file.** VACUUM frees space *inside* the file and
   only truncates trailing pages; the deletable rows are the oldest = the head of the heap. Real
   reclaim needs `pg_repack` or time partitioning + `DROP PARTITION`. **Never `VACUUM FULL`** —
   exclusive lock, and it needs free space equal to the table.
10. **Probe a deletion boundary directly; never trust a computed gap.** A proposed cutoff based on a
    "clean write gap" would have deleted ~7,126 rows that were only 6.6 days old.
11. **Don't trust a sub-agent's "could not find the code" claim.** One investigation reported a rule
    evaluator "lives in a TypeScript edge function, never found." It was an ordinary SQL function,
    one `pg_proc` query away.
12. **Dedupe functions on full signature, never on `proname`.** `ottoq_sim_seed_fleet` has two
    overloads; deduping by name once left the most destructive routine in the set still granted.

## 9. The tick-cost story (so you don't re-derive it)

The slow tick was **not** the shield and **not** cuOpt. It was **event-write amplification**:
`ottoq_record_event` cost ~28 ms per call × 60–120 calls per tick = the entire tick. Root cause was
index maintenance on a table that no longer fit in cache — an insert into `ottoq_events` (18
indexes) took 4.03 ms versus 0.065 ms into an unindexed clone (**62×**).

**Refuted hypotheses, retained so nobody re-tests them:** the safety shield (all rule evals summed
to 2.7 s across a *12-minute* window; one 17.7 s tick had **zero** evals) · the pg_net HTTP post on
the hot path (pg_net is async, measured 119 ms) · metronome lock contention (fencing changed
nothing).

⚠️ **`pg_stat_statements.track` is `top`**, which rolls all nested work into the caller. A claim that
"the metronome is 35.7% of DB time" was a measurement artifact — that 35.7% **is** the tick cost
seen from outside, not a second 35% to reclaim. `track_functions` is `none`, so
`pg_stat_user_functions` is empty; use explicit `clock_timestamp()` instrumentation.

## 10. How to make a change, end to end

```
1. Write the migration as a FILE in otto-q-core/db/migrations/NNNN_name.sql
2. Commit it to a branch.
3. Test it — Supabase preview branch, or a transaction you roll back.
4. Verify with real queries and real row counts.
5. Open a PR with the evidence.
6. Chase merges.
7. Apply to production from the merged file, and record it in MIGRATION_LOG.md.
```

**The repo's own rule, in bold in its README:** *If it isn't a committed file, it didn't happen.*
This exists because the live DB once had 621 applied migrations while the working folder had 80
files with **zero overlap**.
