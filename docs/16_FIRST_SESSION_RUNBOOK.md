# 16 — First-session runbook

**Concrete commands for your first hour.** Written after auditing the rest of this package and
finding that several practical things a fresh agent needs were assumed rather than stated.

---

## 0a. ⭐ The working-copy rule — read this before you clone anything

> **GitHub is the only source of truth. Clone fresh, work, push a branch, delete the clone.**

There are **no persistent local working copies** on this project any more. The founder removed them
on 2026-08-08 after stale clones (up to **77 commits behind**) caused real errors — including in the
first draft of this package. Full account in `docs/17_LOVABLE_AND_SYNC.md` §7.

**Never leave a branch unpushed at the end of a session.** Six branches were nearly lost that way.

## 0. Two more facts that will trip you immediately

### `OTTOYARD` on GitHub is a **personal account, not an organization**

```
GET /users/OTTOYARD   →  { "login": "OTTOYARD", "type": "User" }
GET /orgs/OTTOYARD/repos  →  404
```

Elsewhere in this package (and in most conversation about the project) it is loosely called "the
org." **For prose that is harmless; for an API call it is a 404.** Use:

```
GET  https://api.github.com/user/repos?per_page=100&affiliation=owner
POST https://api.github.com/user/repos            # to create a repo
```
not the `/orgs/OTTOYARD/...` equivalents. Repo URLs are still
`https://github.com/OTTOYARD/<repo>`.

### Every `supabase/config.toml` in every repo points at the WRONG project

Measured across all the repos on disk:

| `project_id` found in a `config.toml` | Reality |
|---|---|
| `hfjaofyfxsyniohdfacg` | **Does not exist.** Dead reference. |
| `odhpbdhnpcrjeaxvbrzd` | **Dead project.** |
| `ycsisvozzgmisboumfqc` | Real, but that is **OrchestrAV's legacy database** — not the core. |
| `OTTO-Q_V1` | A placeholder string, not a ref. |

**`gxdrcyphqjzjsuhxuqtg` — the real core — appears in NONE of them.** It is hardcoded in client
code instead (`src/lib/supabase.ts`, `src/lib/ottoTwin.ts`, `src/lib/otto-q-api.ts`).

> 🚨 **Consequence: `supabase link` / `supabase db push` / `supabase functions deploy` run from any
> of these repos will target the wrong project.** Never run a Supabase CLI command that writes,
> without first checking `--project-ref` explicitly. Do not "fix" the config files without telling
> Chase — the edge functions genuinely deploy to `gxdrc`, and changing a linked ref has deployment
> consequences.

---

## 1. Get a working database connection

**This is the first thing to sort out, and it is the most likely thing to be missing.** Claude Code
reached the database through a Supabase MCP server. **You may not have that.** In rough order of
preference:

**Option A — a Supabase MCP server** (what Claude used). If your runtime supports MCP, this is the
cleanest: it gives `execute_sql`, `list_tables`, `apply_migration`, `get_logs`, `get_advisors`,
`create_branch`, `list_edge_functions`. Ask Chase for a Supabase **personal access token** scoped to
the project.

**Option B — the Supabase CLI.** ⚠️ **It is not installed on the founder's machine** — install it
yourself in your own environment.
```bash
npm i -g supabase            # or brew install supabase/tap/supabase
supabase login               # needs a Supabase access token
supabase projects list
supabase db dump --project-ref gxdrcyphqjzjsuhxuqtg --schema public -f schema.sql
```
**Always pass `--project-ref gxdrcyphqjzjsuhxuqtg` explicitly.** Do not rely on `link`.

**Option C — direct Postgres.** Needs the database password, which only Chase has.
```
postgresql://postgres:<PASSWORD>@db.gxdrcyphqjzjsuhxuqtg.supabase.co:5432/postgres
```
⚠️ Claude never had this — `pg_dump` was unavailable for that reason, which is why schema baselines
were captured via `ottoq_schema_snapshots` inside the database instead.

**Option D — PostgREST over HTTPS.** Works for RPCs and table reads with the anon or service-role
key.
```bash
curl -s "https://gxdrcyphqjzjsuhxuqtg.supabase.co/rest/v1/rpc/ottoq_active_sim_run" \
  -H "apikey: $SUPABASE_ANON_KEY" -H "Content-Type: application/json" -d '{}'
```
⚠️ **PostgREST only exposes the `public` schema.** The `ottoq` and `twin` schemas are deliberately
invisible to it — that is a security feature, not a bug.

**If none of these work, ask Chase using the template in `12_CREDENTIALS.md` §3.** Be specific about
which option you want and why. **Do not proceed on guesses about database state** — this package is
full of examples of what that costs.

---

## 2. Confirm you are looking at the real system

Run these four. They take seconds and they anchor everything else.

```sql
-- 1. Am I on the right database?
select current_database(), version(),
       pg_size_pretty(pg_database_size(current_database())) as db_size;
-- expect: postgres, PostgreSQL 17.6, a few hundred MB

-- 2. Is the schema split real?
select nspname, count(*) from pg_proc p
  join pg_namespace n on n.oid = p.pronamespace
 where nspname in ('public','ottoq','twin') group by 1;
-- expect roughly: public ~1088 (incl. ~744 PostGIS), twin ~71, ottoq ~55

-- 3. Is the run engine alive?
select jobid, jobname, schedule, active from cron.job order by jobid;
-- job 12 (ottoq-demo-metronome) MUST be active — it IS the run engine

-- 4. Is anything running right now?
select id, status, run_by, scenario_code, random_seed, sim_clock_current, tick_count
  from ottoq_sim_runs
 where status in ('running','paused')
 order by started_at desc;
```

> ⚠️ **If a run is already going and it is not yours, do NOT start one.**
> `ottoq_start_demo_run` **aborts** running non-production runs. Ask, or work on something else.

---

## 3. Your first honest run

You will learn more from one properly-finished run than from a day of reading.

```sql
-- START (busy_day, 3× speed, 1 sim-day, the pinned seed)
select public.ottoq_start_demo_run('busy_day', 3.0, 1, 424242);
```

Let it run. The metronome (cron 12) advances it every minute. **Give it at least 139 sim-minutes —
anything shorter certifies nothing.** Watch progress:

```sql
select id, tick_count, sim_clock_current, status from ottoq_sim_runs
 order by started_at desc limit 1;
```

```sql
-- STOP (this archives the run, then wipes the twin)
select public.ottoq_sim_stop_and_reset('<run_id>', 'first look');
```

> ⚠️ **Capture evidence only AFTER the run is stopped.** A mid-run capture once understated
> assignments 6→10 and overstated cuOpt share 20%→33%.

Then read it:

```sql
select * from ottoq_score_run('<run_id>');

-- who decided what, and on whose proposal
select resolved_action_context,
       proposed_action->>'source' as source,
       outcome_status, count(*)
  from ottoq_decisions where sim_run_id = '<run_id>'
 group by 1,2,3 order by 4 desc;

-- did the shield actually evaluate anything
select rule_code, verdict, count(*) from ottoq_rule_evaluations
 where sim_run_id = '<run_id>' group by 1,2 order by 3 desc;

-- did the forward calendar hold
select * from ottoq_booking_provenance_summary('<run_id>');
select state, count(*) from ottoq_stall_bookings
 where sim_run_id = '<run_id>' group by 1;

-- the number that must always be zero
select * from ottoq_count_enacted_breaches('<run_id>');
```

**Things worth noticing on that first run**, because they are the live open questions:
- What fraction of stall assignments carry `source='cuopt'`? (It has spanned 2.8% → 42.4% → 18%.)
- Were any `exterior_wash` atoms emitted? (A daytime run should emit **zero** — that is P1-1, the
  night gate.)
- Did any bay booking die `no_show_grace_elapsed`? (That is P1-12.)
- Did the run's seed actually differ from `424242` if you passed a different one? (That is P0-4, the
  pin.)

---

## 4. Get the front ends running

```bash
git clone https://github.com/OTTOYARD/ottoyarddepot-sim.git
cd ottoyarddepot-sim && npm install && npm run dev     # serves on :8080
```

Open it, start a scenario from the Operator Console Run Control, and watch the depot. Toggle
**2D / 3D / RTX**.
⚠️ The viewport guard requires **≥1200px** — use 1440×900 or larger.
⚠️ The **RTX** tab needs the AWS Isaac box running, and its IP override lives in
`localStorage('ottoq_omniverse_ip')`. It will be blank if the box is stopped, which is normal and
correct.

Before any PR from a front-end repo:
```bash
npm run verify        # depot-sim: typecheck && vitest run && vite build
```

---

## 5. Lovable — check before you assume

All three front ends were scaffolded in **Lovable** and keep **GitHub sync**. The platform decision
(2026-06-27) was **stay on Lovable, no migration**; complex visual code is hand-authored in the repo
via that sync, because Lovable's generation degrades above ~300-line components.

**What Claude did not verify, and you should, before your first front-end PR:**
- Whether Lovable pushes its own commits to `main` (which would mean a merge race with your PRs).
- Whether merging a PR triggers a Lovable rebuild or deploy.
- Whether the deployed Lovable site tracks `main` automatically.

**Ask Chase.** One question up front is much cheaper than discovering a race after the fact.

One known historical symptom worth keeping in mind: an error Chase reported **only reproduced on the
deployed Lovable site**, never locally.

---

## 6. Where to look when something is wrong

| Symptom | Look here first |
|---|---|
| The simulation is not advancing | `select * from cron.job where jobid = 12;` — is it active? Then `select * from cron.job_run_details order by start_time desc limit 10;` |
| Cron says `succeeded` but nothing happens | **`get_logs(postgres)` — this is the one that catches swallowed exceptions.** A `RAISE WARNING` inside a swallowed transaction is invisible everywhere else. This is exactly how the `leg_type` abort was found, on Chase's instruction. |
| Zero decisions this tick | Same as above. One unmapped vocabulary value can roll back an entire tick while reporting success. |
| A function seems slow | `EXPLAIN (ANALYZE, BUFFERS)` it. **Grep for `::text` on enum columns first** — that has been the answer more than once. |
| A guard "passes" suspiciously | Check whether it can ever fail. Multiple guards here were **vacuous**. Prove it has fired at least once. |
| A number looks too good | Read `memory/reference_ottoq_real_edge.md`, then interrogate the baseline. It was invalid twice. |
| The cockpit shows nothing | Is a run actually running? An idle depot legitimately looks empty. Also check which project ref the client points at. |
| An RPC 404s over PostgREST | It is probably in the `ottoq` or `twin` schema — **deliberately invisible to PostgREST.** |

---

## 7. Your first PR — a suggestion

**Fix the 3D car scale.**

`src/components/canvas/three/Vehicle3D.tsx` uses `BoxGeometry(2.2, 0.85, 4.9)` — **authored in
metres and dropped into unit-space**, where 1 unit = 0.4785 m. It therefore renders at roughly 48%
of correct size: a toy car parked in a correctly-sized stall. The 2D body is already correct at
`4.2 × 10.2` units.

Why it is a good first PR: it is visually obvious, it is genuinely wrong, the fix is small (a uniform
group scale of about 2.0), it is low-blast-radius, and it teaches you the whole verification loop —
read the file, understand the yardstick, change it, run `npm run verify`, screenshot before and
after, write the PR with evidence.

⚠️ **Do not, in the same PR, touch `rightOffset` or `traffic.ts CAR_LENGTH`.** Those move routed
motion and need a certified pass. Note them in the PR as follow-ups.

---

## 8. The habits that will keep you out of trouble here

1. **Search before you build.** The prior is "it already exists, half-wired."
2. **Verify a stored fact before building on it**, including facts in this package.
3. **Make every vocabulary mapping a total function** with a safe, loud `ELSE`.
4. **Prove a guard has fired.** A guard that cannot fail is worse than none.
5. **Prefer a runtime marker over source inspection.**
6. **Always state your denominator.**
7. **Finish the run.** An unfinished run means every number reads NOT MEASURED.
8. **Say what you did not verify.** Every time.
