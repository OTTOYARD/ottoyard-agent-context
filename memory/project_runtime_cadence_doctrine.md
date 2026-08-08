---
name: project_runtime_cadence_doctrine
description: "⭐⭐CORE DOCTRINE (Chase 2026-07-30): the TICK is not OTTO-Q's decision rate. Twin = constant motion at 1:1; OTTO-Q = slow background scan + push wake; run lifecycle START/PAUSE/STOP; temp staging is the pressure-relief valve when a decision lands late."
metadata: 
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-07-31T03:27:59.815Z
---

**⭐⭐ THE DISTINCTION CHASE DREW (2026-07-30) — I was conflating two things and he corrected it.**
A "tick" had been doing two jobs: advancing the twin's world AND running OTTO-Q's decision pass.
**They are separate concerns with separate requirements.** He is right, and the code already supports
it — `ottoq_sim_advance_tick_world` and `ottoq_sim_decide_and_dispatch` are already distinct
functions (the 2026-07-13 de-fuse), and I ran them independently 5× to measure them.

**MEASURED WARM, SEPARATELY (2026-07-30, 5 samples each):**
| half | median | range |
|---|---|---|
| twin world step (physics) | **~914 ms** | 853-1,026 |
| OTTO-Q decision (thinking) | **~181 ms** | 165-324 |
⇒ **OTTO-Q already answers in ~0.18 s, well inside Chase's stated 0.5-1 s tolerance.**
⇒ **The twin's physics is ~5× the cost of the brain.** So smooth 1:1 motion is a TWIN-side
engineering problem, NOT an OTTO-Q speed problem. (This reversed my working assumption.)

---

## Chase's three answers — verbatim intent, treat as doctrine

**1. WHAT WAKES OTTO-Q → both.**
- A **slow, constant background scan** — *"especially analysing energy/grid/battery triage."*
- **PLUS push/spontaneous activation** on a need — *"a random flag for immediate vehicle return for
  cleaning or sensor, etc."*

**⭐ RUN LIFECYCLE IS PART OF THIS ANSWER AND IS A HARD CONTRACT:**
- **NOTHING RUNS UNTIL "START".** Neither OTTO-Q nor the twin. On START the twin *"comes alive and
  begins selecting variables for OTTO-Q to begin sorting."* (Reinforces
  [[project_arrival_and_tick_doctrine]] — depot ticks ONLY on explicit start.)
- **PAUSE freezes everything in time** within the twin.
- **STOP** (from the controls tab *or* the black box tab) **completely resets and wipes the twin
  clean** for a new run, **while saving/storing that previous run's data log.**
  ⇒ STOP is destructive-to-world but preserving-to-record. Both halves are required.

**2. SPEED-UP LATENCY → acceptable; do NOT scale the brain's clock.**
*"OTTO-Q will make decisions that won't play out for several minutes at a time. Even a last-minute
stall assignment might decide instantly, within a second or two max, but still takes minutes to
actually move through the depot."* So OTTO-Q keeps **real-time** latency while the twin runs 2-3×;
it must *"still safely and efficiently orchestrate while the twin simulation is sped up."*
⇒ Simplifies the build: no brain-clock scaling, no sim/real latency conversion.

**3. TWIN NEVER FREEZES FOR A DECISION.**
*"Twin should be in an absolute state of motion with vehicles, chargers, energy etc always firing and
rendering."* OTTO-Q reacts via the steady scan; anything urgent arrives as a **push**.
**⭐ AND THE PRESSURE-RELIEF VALVE:** *"OTTO-Q can utilise the TEMPORARY STAGING exactly for this
purpose if no exact replacement can be orchestrated quick enough or ideal enough in that moment."*
⇒ This is the answer to "what if the world moved while OTTO-Q was thinking": **do not block, do not
fail — stage temporarily and re-plan.** Consistent with [[project_perimeter_hold_doctrine]] (temp
staging and perimeter hold are different PURPOSES, not overflow tiers).

---

## ⭐ Why this unblocks the twin/OTTO-Q split
The rule-1 inversion was rejected (2026-07-30) because the actuation window is **30 sim-minutes** —
orders sat unexecuted while the world moved on. Under this doctrine the window is **~1 second**, and
the "world changed while the order was in flight" case has a *designed* answer (temp staging + the
refusal/re-plan loop that already exists and fired live today). **Chase spotted that these were the
same problem before I did.** Order: clock/cadence → inversion prep → inversion.

## Also settled: the 24-hour concern is void
Chase will *"only ever grab snapshots"*, use 2×/3× to capture hours of depot data in a fraction of
the time, and jump ahead or reset to have OTTO-Q reassess against the twin's new variable spin-up.
⇒ **No need to touch the certification path or keep a separate `fixed` cert mode on his account.**

---

## ✅ BUILT 2026-07-30 — STOP now saves the run log (the missing half of the contract)
**GAP FOUND:** `ottoq_sim_stop_and_reset` returned `'blackbox_ready', true` but **persisted nothing**.
`ottoq_run_blackbox()` builds its bundle ON DEMAND from live rows, and the next
`ottoq_start_demo_run` purges those rows (4,084 deleted this morning). **No `*_archive` table existed
anywhere in the schema** ⇒ the previous run's log survived only if a human manually pulled it between
STOP and the next START. Live data-loss path.

**BUILT:** `public.ottoq_run_archives` (RLS on, service_role only, revoked from PUBLIC/anon/authenticated
**at birth** — deliberately not repeating [[project_rls_public_policy_hole]]) + `ottoq_archive_run(run, reason)`,
called by `ottoq_sim_stop_and_reset` **BEFORE** the wipe and **with no EXCEPTION handler** — if the record
cannot be saved the stop fails loudly rather than silently discarding what it exists to protect.
**Design:** archive the **reproducibility key** (scenario + `random_seed` + policy + depot) **+ outcome
metrics**, NOT the raw bytes. A run replays exactly from its seed, and Chase's need is *"just the data so
it can be either recreated or just analyzed"*. The full black-box bundle is ~24 MB/run — unaffordable
while `ottoq_events` sits at 9 GB. Revisit full-bundle archival after the capacity work.

**🚨 CAUGHT ONLY BY TESTING THE CONTRACT END-TO-END — the first version was silently destroyed.**
`ottoq_purge_prior_runs` sweeps **generically**: every `public.ottoq%` table having a `sim_run_id` column.
`ottoq_run_archives` matched and was wiped on the very first purge (`archive_survived = 0`). Fixed with a
named exclusion list `c_keep_tables = {ottoq_sim_runs, ottoq_run_archives}`. **LESSON: any new table with a
`sim_run_id` column is auto-swept by the purge — add it to the exclusion list or it will vanish.**
**VERIFIED (rolled-back txn, nothing left behind):** `stop_archived=true · archive_before=1 ·
archive_AFTER_purge=1 · run_row_after=0 · seed_replayable=700008 · purged_rows=1131`.

**⚠️ SECOND DEFECT CONFIRMED IN THE SAME READ (tracked, NOT fixed):** the purge deletes children via
`WHERE sim_run_id IN (SELECT … FROM ottoq_sim_runs …)`, so **it can only reach children of runs that still
exist**. Any row outliving its run row is orphaned **permanently** — that is why 252 `ottoq_vehicle_commands`
and 79 `ottoq_events` rows are stranded. It also swallowed every per-table failure with
`EXCEPTION WHEN OTHERS THEN NULL`; now at least `RAISE WARNING`.

## ✅ BUILT 2026-07-30 — PAUSE / RESUME, and the clock leak they exposed
`ottoq_pause_run(run, reason)` + `ottoq_resume_run(run)` (service_role only; both emit
`twin.sim_run_paused` / `twin.sim_run_resumed`). **START · PAUSE · STOP are now all first-class.**

**🔴 THE BUG THIS EXPOSED — raw `UPDATE status='paused'` does NOT freeze time in live mode.**
`ottoq_sim_advance_tick_world:25` derives the advance from `clock_timestamp() - last_tick_at`, and
`:79` anchors `last_tick_at` at **tick START** (deliberate: *"so the tick's own compute is inside the
next elapsed"*). A paused run keeps its stale anchor ⇒ **the first tick after resume advances the sim
clock by the ENTIRE paused interval**, capped only by the 10-minute anti-teleport ceiling on line 24.
Pause for 8 minutes, come back, and the depot silently jumps 8 minutes. Only bites in `live` mode.
**FIX: RESUME re-anchors `last_tick_at = clock_timestamp()`.**
**MEASURED PROOF:** anchor was stale by **49.4 s**; sim advance on resume = **0.03 s**
(`paused_real=37.24s`, `clock_reanchored=true`). Without the fix that would have been a 49-second jump.

**❌ AND A CORRECTION I OWE:** I earlier called the first tick's "over-advance" a bug (run started
22:57:52, tick 1 advanced 43 sim-sec over 26 real-sec). **It is NOT a bug** — `last_tick_at` is set at
run creation, so the first tick correctly bills the real time between pressing START and the first
beat. Withdrawn.
## ✅ RESOLVED — THE 1:1 CLOCK IS EXACT. The 1.246 was MY MEASUREMENT, not the engine.
**MEASURED BOUNDARY-ALIGNED: ratio = 1.0000 over 43 s (1 tick) and 1.00017 over 62 s (2 ticks).**
Sim-seconds 42.98 vs real-seconds 42.98. **`live` mode delivers exact 1 real sec = 1 sim sec.**

**Why my earlier 1.246 was wrong:** I compared `sim_clock_current` (which advances in ~10-30 s STEPS
at tick boundaries) against a continuously-running `now()`, sampled at arbitrary moments. That gives up
to a full tick of phase error at **each** end — easily 25% over a 67 s window.
**⭐ THE CORRECT INSTRUMENT: compare `sim_clock_current` against `last_tick_at`.** Both are written by
the SAME statement (`advance_tick_world:79`), so they are phase-aligned by construction and the ratio
is exact. Never sample a step-advancing clock against a continuous one.

**Three hypotheses raised and ALL REFUTED before the real answer:** (1) a 15-second dt floor — line 24
is `GREATEST(0.0, …)`, no floor; (2) a stale/mis-set anchor — line 79 anchors at tick START, correct
by design; (3) a second writer double-advancing — `twin.ottoq_world_advance:10` is scoped
`WHERE run_by='production_live'` so it never touches demo runs, and `ottoq_cron_tick:5` skips when a
run is running. *(Full writer list of `sim_clock_current`: `ottoq_sim_advance_tick_world`,
`ottoq_start_demo_run`, `twin.ottoq_world_advance`, and the DEAD `twin.ottoq_sim_advance_clock`.)*
**Lesson: I guessed twice and was wrong twice; the answer came from picking the right instrument.**

**⇒ PHASE A (clock ratio) IS DONE AND CORRECT. The remaining gap is CADENCE, not ratio.**
Measured **31.1 real-sec between ticks** on this run (and ~8.4 s on an earlier one) — irregular and far
too coarse for smooth motion, while tick COMPUTE is only ~1.15 s. So smoothness is bounded by the
**pg_cron metronome's firing cadence**, not by engine speed. That is the next piece of work.

~~**STILL MISSING for the lifecycle contract: PAUSE has no first-class function**~~ — only a raw
`UPDATE ottoq_sim_runs SET status='paused'` (which does work; the metronome only ticks `status='running'`).
START (`ottoq_start_demo_run`) and STOP (`ottoq_sim_stop_and_reset`) both exist.

Links: [[project_twin_realtime_clock]], [[project_ottoq_twin_boundary]],
[[project_perimeter_hold_doctrine]], [[project_arrival_and_tick_doctrine]], [[project_demo_loop_ops]].
