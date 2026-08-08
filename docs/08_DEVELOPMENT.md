# 08 — Development: run it, test it, migrate it, verify it

## 1. Clone what you need

```bash
git clone https://github.com/OTTOYARD/otto-q-core.git          # the brain, in files
git clone https://github.com/OTTOYARD/ottoyarddepot-sim.git    # OTTO-TWIN cockpit + renderer
git clone https://github.com/OTTOYARD/ottoyard-field-ops.git   # OTTO-PULSE
git clone https://github.com/OTTOYARD/ottoyard-OTTO-Q.git      # OrchestrAV
git clone https://github.com/OTTOYARD/ottoq-intelligence.git   # frontier Python service
git clone https://github.com/OTTOYARD/otto-q-workspace.git     # ~90 design/spec docs
```

⚠️ **If you are working on the same machine as another agent, use `git worktree` or clone into your
own directory.** Two sessions in one repo folder share one working tree and will clobber each other.

## 2. Running the front ends

All three React apps are Vite + React + TypeScript + shadcn/ui + Tailwind. They are Lovable-synced,
so **the repo is the source of truth and Lovable follows it.**

```bash
npm install          # or: bun install  (bun.lock is committed)
npm run dev          # depot-sim serves on :8080
```

| Script | Repos | What |
|---|---|---|
| `dev` | all | Vite dev server |
| `build` | all | production build |
| `test` | all | vitest (`test:run` in OrchestrAV) |
| `typecheck` | depot-sim | `tsc -p tsconfig.app.json --noEmit` |
| **`verify`** | **depot-sim** | **`typecheck && vitest run && vite build` — run this before every PR** |
| `layout:seed` | depot-sim | `exportSitePlan.mjs && buildLayoutSeed.mjs` |
| `layout:check` | depot-sim | `checkLayoutGeometry.mjs` |
| `layout:verify` | depot-sim | rebuild the seed, assert **no git diff**, then run the geometry check |

`ottoyard-field-ops` and `ottoyarddepot-sim` also have Playwright configured
(`playwright.config.ts`, `playwright-fixture.ts`).

⚠️ **Dev-server gotcha:** after npm changes, if the RTX lazy chunk fails to fetch, `rm -rf
node_modules/.vite`. Production builds are unaffected.

⚠️ `ResponsiveGuard` in depot-sim requires **≥1200px** viewport width. Test at 1440×900 or larger.

## 3. Running the frontier service

```bash
cd ottoq-intelligence
pip install -r requirements.txt
uvicorn app.main:app --port 8080
python tests/test_energy_alpha.py     # the energy MPC alpha test
```
Deployment notes: `deploy/AWS_PLAYBOOK.md`, `deploy/DEPLOY_EC2.md`, `Dockerfile`.

## 4. Running a simulation — the thing you will do most

Everything runs against the live database. There is no local Postgres.

```sql
-- START: pick a scenario, playback speed, sim days, and a seed
select public.ottoq_start_demo_run('busy_day', 3.0, 1, 424242);

-- WATCH: the metronome (cron 12, every minute) advances it automatically
select id, status, scenario_code, sim_clock_current, tick_count, random_seed
  from ottoq_sim_runs order by started_at desc limit 3;

-- PAUSE / RESUME
select public.ottoq_pause_run('<run_id>', 'inspecting');
select public.ottoq_resume_run('<run_id>');

-- STEP MANUALLY (instead of waiting for the metronome)
select public.ottoq_sim_advance_tick('<run_id>');

-- JUMP AHEAD
select public.ottoq_sim_jump_forward('<run_id>', 120, 30);   -- 120 sim-min, ≤30 real-sec

-- STOP: archives the run, then wipes the twin
select public.ottoq_sim_stop_and_reset('<run_id>', 'done');
```

**Scenarios:** `normal_day` · `busy_day` · `heat_wave` · `winter_storm` · `dr_event_cascade` ·
`charger_outage_morning_rush` · `grid_brownout_at_peak` ·
`solar_underperformance_partly_cloudy` · `aggressive_fleet_turnover`.

**Rules of the road:**
- ⚠️ **`ottoq_start_demo_run` aborts running non-production runs.** One running run per depot.
- ⚠️ **A run shorter than ~139 sim-minutes certifies nothing.** Two build phases were wasted on
  n=3 and n=6 runs.
- ⚠️ **Capture evidence only AFTER the run is stopped.** A mid-run capture once understated
  assignments 6→10 and overstated cuOpt share 20%→33%.
- ⚠️ **A demo run is not free.** It writes to `ottoq_events` and other tables. Budget against
  remaining headroom — one session added ~3 GB by repeatedly certifying into an almost-full
  container.
- ⚠️ **Do not start a demo run while certification arms are executing.**

**Reading what happened:**

```sql
select * from ottoq_score_run('<run_id>');            -- the scorecard
select * from ottoq_run_blackbox('<run_id>');         -- the flight recorder
select * from ottoq_twin_snapshot('<run_id>');        -- world state, as the renderer sees it
select * from ottoq_depot_cards('11111111-1111-1111-1111-111111111111');
select * from ottoq_nl_status_brief('<run_id>');      -- plain-English status
select * from ottoq_booking_provenance_summary('<run_id>');
select * from ottoq_count_enacted_breaches('<run_id>');
select * from ottoq_clock_fidelity('<run_id>', 20);
```

**Watching the decision layer:**

```sql
select resolved_action_context,
       proposed_action->>'source' as source,
       outcome_status, count(*)
  from ottoq_decisions
 where sim_run_id = '<run_id>'
 group by 1,2,3 order by 4 desc;

select rule_code, verdict, count(*)
  from ottoq_rule_evaluations
 where sim_run_id = '<run_id>'
 group by 1,2 order by 3 desc;
```

## 5. Making a database change

**The regime, in order. It is not optional.**

```
1. Write the migration as a FILE:  otto-q-core/db/migrations/NNNN_short_name.sql
2. Commit it to a branch.
3. Test it — a Supabase preview branch, or a transaction you roll back.
4. Verify with real queries and real row counts.
5. Open a PR containing the evidence.
6. Chase merges.
7. Apply to production from the merged file; record it in MIGRATION_LOG.md.
```

Template: `otto-q-core/db/migrations/0001_EXAMPLE_template.sql`.
Read `otto-q-core/db/migrations/README.md` and `otto-q-core/README.md` first.

**Rolling back and capturing baselines:**
- `public.ottoq_schema_snapshots` holds labelled DDL snapshots (~6,000 objects). Take one before any
  significant DDL: existing labels include `pre_separation_2026_07_29`,
  `pre_schema_move_2026_07_29`, `pre_twin_move_2026_07_29`, `pre_split_2to8_2026_07_29`.
- `public.ottoq_fn_definition_backups` holds per-change function backups.
- **Always capture `pg_get_functiondef` before touching a function.** `pg_proc` is the only live copy.

**The migration-safety checklist:**

- [ ] Is this a file in `otto-q-core/db/migrations/`? If not, stop.
- [ ] Have I captured `pg_get_functiondef` for every function I am replacing?
- [ ] Have I taken an `ottoq_schema_snapshots` label?
- [ ] Will this run in under ~15 seconds? If not, pause the run first
      (`UPDATE ottoq_sim_runs SET status='paused'`) and use `SET LOCAL lock_timeout='8s'`.
- [ ] Are my safety scans **one pass, not N**? (Per-item `prosrc LIKE` is quadratic and has flattened
      the box.)
- [ ] Does any new table have a `sim_run_id` column? If so, it will be **auto-swept by
      `ottoq_purge_prior_runs`** at the next run start — add it to `c_keep_tables` or rename the
      column.
- [ ] Am I deleting a `stalls` row? `ottoq_stall_bookings.stall_id` is **ON DELETE CASCADE**.
      **Re-home, never retire.** A stall may only be deleted if it has zero references across all
      17 FK columns; otherwise UPDATE it in place.
- [ ] Am I comparing an enum with `::text`? That silently defeats partial indexes and can match zero
      rows forever.
- [ ] Does the routine `COMMIT`? Then it cannot carry a `SET` clause — use
      `PERFORM set_config('search_path', 'twin, ottoq, public, extensions', false);` in the body.
- [ ] Am I creating a function with a `$$`-quoted body? **The Supabase MCP migration runner
      mis-parses those.** Use raw `execute_sql` with `$fn$` quoting, then `apply_migration` for the
      ledger entry.
- [ ] If a migration times out: **poll whether it landed before retrying.** Never blind-retry.
- [ ] Did I check `get_logs(postgres)` afterwards, not just the success flag?

## 6. Verifying front-end work

Text-based verification beats screenshots for correctness; screenshots are for the founder.

1. Start the dev server (depot-sim is configured in `.claude/launch.json` on port 8080).
2. Read console messages and network requests for errors.
3. Read the accessibility tree for content and structure.
4. Drive interactions, then re-read to confirm.
5. Resize for responsive and dark-mode checks.
6. **Then** take a screenshot as evidence for the PR.

For motion work specifically:
- **Replay the captured fixture**, not a live run: `src/engine/__fixtures__/twinRun.busyday.json` +
  `replay.ts`. Live runs make intermittent bugs unfalsifiable.
- **Measure oriented body overlap** (the 4.2 × 10.2 box), never centre distance.
- **Sample during motion**, not after the scene settles.

## 7. Git conventions

- Branch from `main`. Name it `hermes/<short-slug>`.
- ⚠️ **GitHub rejects pushes authored as `chase@ottoyard.com`** (email privacy). Commit as the
  noreply identity. Both cockpit repos are already configured for this — check
  `git config user.email` in any repo you commit from.
- **Never push to `main`. Never merge.** Open a PR; Chase merges.
- A PR description must contain: what changed · **the evidence it works** (real numbers, row counts,
  screenshots) · what you did **not** verify · what could break.

## 8. Where the deeper working agreements live

`ottoyarddepot-sim/docs/`:
- `MISSION-AND-WORKING-AGREEMENT.md`
- `OTTO-Q-WORLD-CONTRACT.md` — the twin/world contract
- `OTTOQ-TWIN-BOUNDARY.md` — the separation plan
  ⚠️ This file was previously **only on an unmerged branch** (`claude/simulation-data-otto-q-4p7450`)
  and one branch deletion from gone. It now appears in `docs/` on the local clone — **confirm it is
  on `main` before relying on that.**

⚠️ **A correction that matters if you read `OTTO-Q-WORLD-CONTRACT.md`:** it says "118 sim/twin
functions." The real figure is **387 non-extension routines** — a scoped number was reused as an
unscoped total. Other corrections found in that doc: the §4 portability list omits `executors.ts`
(which imports `TwinMotionDriver` and is therefore **not** portable); "reservations are TTL'd" is
misleading (**the TTL is decorative — nothing expires them**); and there are **four** twin policies,
not three (`manual` is real, with 56 benchmark rows).

## 9. Cost awareness

- **cron 10** (`ottoq_cron_tick`, every 2 minutes) calls paid edge functions. It costs money
  continuously.
- **cuOpt** is a billable NVIDIA API call per solve. `cuopt_debounce_s` exists for this reason.
- **Nemotron** is a billable NIM call.
- **The AWS g6e box** costs ~$2–4/hour. **Stop it when idle.**
- Certification runs consume database headroom.

If a change would materially increase any of these, say so in the PR.
