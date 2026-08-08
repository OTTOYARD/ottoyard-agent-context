---
name: project_orchestration_build_2026_08_01
description: "⭐⭐THE BUILD (2026-08-01/02): needs card → atomic enactment → space-agnostic assignment → armed overlap guard. CERTIFIED at full length: throughput +85.8%, charging +76.8%, first bay enactments EVER, calendar 0%→100%, zero real double-bookings. Open: cuOpt share regressed 37.3%→12.6% because availability counts stale calendar rows."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-03T12:57:16.135Z
---

# The build Chase asked for: "tell vehicles what to do, when, in which space, and why"

## ✅ PHASE 1 — the INPUT (rich per-vehicle need cards)
`public.ottoq_vehicle_needs_card` — **66 columns, one row per vehicle**. Chase's own idea and it was
the right foundation. Key columns: `must_do_now text[]`, `deferrable_now text[]`, `must_do_legs`,
`overall_urgency`, `next_deploy_at`, `minutes_to_deploy`, `est_charge_min`, `open_must_do_min`,
**`fits_window bool`** (does the required work fit before redeploy?), `need_statement jsonb`.
Ten per-dimension urgency ladders (energy/wash/cabin/calib/pm/fault/tire/brake/software/item) graded
`ok|due_soon|due|overdue|critical` from cadence ratios (elapsed/interval; ≥1.0 due, ≥1.25 overdue).
⚠️ **Its run column is `run_id`, NOT `sim_run_id` — deliberate.** `ottoq_purge_prior_runs` scans
`information_schema.columns` (which includes VIEWS) for `sim_run_id` on `table_name LIKE 'ottoq%'`
and would issue a failing DELETE against a view every run start. **Do not rename it.**

**Cadence escalation (CTO decision, replacing "everything deferrable"):** a service is deferrable
until overdue, then escalates to `must_do`. Calibrated — per-service must_do share **2.6%–23.3%**
(nothing escalates the whole fleet, nothing escalates none); must_do/vehicle 2.39→1.59;
**49.1% of the fleet needs no bay at all**; service_bay must_do load 59.4% of capacity.
⇒ the full-service-visit doctrine is now enforceable **from the data**.

Also built: scenario **`busy_day`** — an idle depot made every prior test meaningless (a run once
enacted 3 assignments because 89 of 120 vehicles were deployed).

## ✅ PHASE 2 — atomic enactment + space-agnostic assignment
- **P0: enactment writes its own booking in the same transaction.** Calendar coverage **0% → 100%**.
- **P1: first bay enactments EVER** (new decision source label `needs_card`).
⇒ These two closed the "two uncoordinated selectors" flaw from [[project_booking_ledger_divergence]].

## ✅ PHASE 3 — CERTIFIED AT FULL LENGTH (151.0 sim-min = 108.6% of the 139 sim-min baseline)
| metric | baseline | certified | |
|---|---|---|---|
| throughput | 0.367/sim-min | **0.682** | **+85.8%** |
| charging | 0.367/sim-min | **0.649** | **+76.8%**, no regression |
| bays enacted | 0 of 51 / 0 of 209 | wash 4, **service 1 (FIRST EVER)** | |
| forward calendar coverage | 0% | **103/103 = 100%** | |
| real double-bookings | vacuous "0" | **0 of 577, guard armed** | |
| service-bay oversubscription | **2.3×** | **19.0%** | |
Bay utilisation: service 19.0% (2/2 touched), wash 17.1% (3/3), dcfc 35.3% (10/10), l2 39.7% (35/35).
`detail_bay 0` is **CORRECT** — the depot physically has **zero detail_bay stalls** (so earlier
"38 detail bookings" were never against detail bays).

**🚨 THE OVERLAP GUARD WAS DISABLED BY CONSTRUCTION.** `ottoq_stall_bookings_no_overlap` was scoped
`WHERE state IN ('held','active')` but **no booking was ever written in those states** (84 done /
78 released / 2 superseded) ⇒ every prior "zero double-bookings" pass was **vacuous**, and 22
vehicles had been booked into one service bay. Fixed with `..._no_overlap_v2` (EXCLUDE gist,
states held/active/done, NOT VALID first) **plus** making bookings **born `held`** and pass through
`active` — a calendar whose rows are born `done` cannot express "this space is claimed T1→T2".

## ⚠️ METHOD RULES LEARNED (reuse these)
1. **Measure occupancy as a UNION OF INTERVALS CLIPPED TO THE RUN WINDOW.** A naive sum of raw
   booking minutes reports service_bay **242%** and wash_bay 138% — an artifact of forward
   reservations extending past the run end.
2. **CAPTURE EVIDENCE ONLY AFTER THE RUN IS STOPPED.** A mid-run capture understated assignments
   6→10 and overstated cuOpt share 20%→33%.
3. **RUN ≥139 SIM-MIN.** Short runs (n=3, n=6) certify nothing; two phases were wasted on them.
4. Always state the DENOMINATOR — conditional vs end-to-end coverage differ hugely
   (98.8% conditional vs 68.1% end-to-end were both true of the same run).

## 🔴 OPEN — cuOpt share REGRESSED 37.3% → 12.6% (root cause DIAGNOSED)
New **per-invocation telemetry** resolved the long-standing ambiguity (`ottoq_cuopt_fire_log` was one
row per run, so "never invoked" and "invoked 60× and abstained 58" were indistinguishable):
**cuOpt is invoked and then ABSTAINS — it is not absent.** This is the OPPOSITE of the 2026-07-30
stale-on-arrival race in [[project_nvidia_layer_truth_2026_07_30]].
- SQL gate: 376 evals → 244 `sql_gate_no_candidates`, 49 `debounce`, **83 reached the edge fn**.
- Edge: 83 invocations, **76 abstained (91.6%)** — `no_free_stalls_demand_present` **47**,
  `solved_but_zero_proposals` 19 (31 candidates in → 0 out), `no_candidates_in_instance` 9.
- The 7 that produced output: 15 candidates → 15 proposals → **13 enacted = 86.7% conversion.**
⇒ **Conversion is healthy; SUPPLY is the bottleneck.**
- **SMOKING GUN:** `free_stalls_in = 0` on **48 of 83** edge calls (57.8%) while l2 utilisation was
  only 39.7% — **~27 chargers were physically free while cuOpt was told there were none.** The
  availability predicate counts `released`/`superseded` rows (**86% of the calendar**) which still
  carry full ~23-min windows. **Fix: availability must consider only `state IN ('held','active','done')`.**

## 🔴 ALSO OPEN
- **5 phantoms of 108 (4.6%)** — `otto_q_enacted` bookings with no decision behind them, written by a
  bay-exit reconciler outside the decision ledger ⇒ reverse coverage 95.4%, not 100%.
- **`leg_id` on only 4 of 108 enacted bookings (3.7%)** ⇒ for 96.3% the calendar still cannot answer
  **"why is this vehicle in this space."** P0's "one atomic operation" is half true: stall matches,
  reason does not travel.
- **`ottoq_bind_unbooked_bay_occupants` gives up `no_free_space` 34×/run** — one vehicle sat in a
  wash bay ~23 real-min unrecorded. **Reality must outrank a plan**: displace the stale claim, book
  the car that is physically there, emit an auditable conflict row.

---

# ✅✅ 2026-08-02 PHASE 6b — cuOpt FINALLY DRIVES THE DEPOT (2.8% → 42.4%)
Certified run: busy_day / seed 424242 / live 3x / **152.7 sim-min / 115 arrivals**. Preserved in
`public.phase6_cert_424242`; prior baseline in `public.phase5_cert_d78dd3b1`.

| metric | before | **after** |
|---|---|---|
| **cuOpt enacted share** | 3 of 106 = 2.8% | **50 of 118 = 42.4%** (15.2×) |
| candidate evaporation | 63% (46 of 73) | **3.7%** (2 of 54) |
| proposals killed by energy trim | **100%** (31 of 31) | **0** |
| phantoms | 14 of 120 = 11.7% | **0 of 136 = 0.0%** |
| arrivals in first 30 sim-min | 0 | **13** |
| arrival buckets covered | 8 of 10 | **10 of 11** |
| forward coverage · real double-bookings | 100% · 0 | **100% · 0** (guard `convalidated=true`) |
| assignments per ARRIVING VEHICLE | 0.981 | **1.026** |
Conversion 97/97 = **100%**. Gate→edge lag 4.11 s → **2.27 s**. Starvation: **none**.
**Seed determinism PROVEN**: `prime_fingerprint 680c6b5b568f1a9b8b2fbf4d68fdbb27` identical across 3
runs on seed 424242, different on seed 777777.

## 🔑 THE THREE ROOT CAUSES THAT HAD STARVED cuOpt (all measured, all fixed)
1. **The gated candidate set was DISCARDED AT THE WIRE.** `ottoq_cuopt_refresh` computed `v_cand`
   with a precise predicate, then posted **only** `{sim_run_id}`. The edge fn re-derived the cohort
   from live state ~4.1 s later — against a 4.27 s tick cadence, i.e. **one full tick late BY
   CONSTRUCTION** (pg_net only queues; it cannot transmit until the caller commits, and the commit
   happens *after* `ottoq_decide_tick` has already parked the cars).
   **FIX: send `candidate_ids` in the body; edge fn honours it.** Runtime proof = all 54 invocations
   report `cohort_mode="pinned"`. ⭐ **Prefer a RUNTIME marker like this over source inspection** —
   an earlier phase "fixed" the edge fn while it stayed byte-identical v22.
2. **The healthy beat almost never ran.** Metronome alternates DECIDE/FIRE. DECIDE-beat fires:
   63, 68% evaporation. FIRE-beat: only 10 fires, 30% evaporation — throttled by
   `cuopt_contention_min=2`, and two cars are rarely at the gate at once. **FIX: removed that veto and
   stood down the DECIDE-beat fire entirely** (it is structurally incapable of arriving in time).
   Now FIRE 54 fires / DECIDE 0.
3. **🚨 A DOCTRINE VIOLATION: `trimmed_by_cap` silently deleted 100% of optimal solutions.** 21
   invocations solved to `status:"Optimal"` and produced ZERO proposals. An energy cap was discarding
   the answer *after* the solve. That breaks [[project_vehicle_first_doctrine]] outright.
   **FIX: the cap is ADVISORY** — `would_trim_by_cap` (56) is still measured and logged, but no
   vehicle is denied a plug for energy. ⭐ **A real ceiling must SHAPE the solution as an LP
   constraint, never post-hoc delete it.**

## 🔴 NEXT LOSS — 37.3% BAY NO-SHOW (likely self-inflicted)
**19 of 51 bay bookings died `no_show_grace_elapsed`** across 18 vehicles. OTTO-Q reserves a bay and
nothing reliably moves the car into it. This is the honest cause of completed-work/arrival slipping to
0.539 (from 0.565) and of detail completions falling 5 → 1 — **not** teardown truncation (that story
was wrong). Bay DEMAND is real and much higher than before (51 bay bookings vs 15 completions).
**PRIME SUSPECT: the new departure-readiness gate pins vehicles in staging while their bay booking
keeps ticking — the gate manufacturing its own no-shows.**

---

# ✅ 2026-08-02/03 PHASES 7-9 — the depot now DOES the work, and the metric is honest
Certified run (phase 9): busy_day / seed 424242 / live 3x / **180.1 sim-min / 113 arrivals**.
Preserved in `public.phase9_cert_424242` (also `phase7_cert_424242`, `phase6_cert_424242`).

| metric | value |
|---|---|
| **honest work per ARRIVING VEHICLE** | **1.336** (151/113) — on a **stricter** definition than the inflated 0.800 |
| three-way (anti-inflation) | done **151** · interrupted **8** · legacy **170**; gap 19 = 8 interrupted **+ 11 staging/perimeter parking** |
| **inspect seam ALIVE** | 342 legs, **79 bookings 100% `source='inspect_seam'`, 78 done** — now the LARGEST finished-work category |
| bay no-show | 37.3% → **0.0%** (phase 7) → 7.3% (phase 9 regression) — grace NEVER widened, still 15 min |
| wash | 6 → 3 (regression) → **11** recovered |
| phantoms · forward coverage | **0 of 211** · **100%** |
| real double-bookings | **0** under the stricter `_v3` (held/active/done/**interrupted**) |
| assignments/arrival | **1.044 ex-inspect** (protected 1.026); churn **down 33.7%** |

## 🚨 THE ANTI-INFLATION LESSON — a metric can improve by LYING or by FORGETTING
1. **`done` did not mean finished.** 8 of 24 bay "dones" lasted <5 min, two **43 seconds**; one
   unfinished service appeared as **three** completed bookings. Mid-service ejections were being
   scored as completed work. **FIX: new terminal state `interrupted`**, and
   `ottoq_is_work_purpose()` now also excludes staging/perimeter parking. `workload_harness_metrics`
   emits **done / interrupted / legacy side by side so the correction cannot be hidden.**
2. **⭐ A metric that improves by FORGETTING outstanding work is WORSE than an inflated one.**
   `ottoq_reopen_visit_atoms` was **dead code** (`WHERE status IN ('open','in_progress')` then
   `IF v_visit IS NULL THEN RETURN 0`; interrupted vehicles have **0 open visits**). Interrupted work
   silently vanished. Now 5 of 8 re-plan (strict `atoms_reopened>0`: 2 of 8) — **still not finished**.
   ⚠️ `reopen_reason` lives in `meta->>'reopen_reason'` — there is **no such column**.

## 🚨 I GATED THE WRONG HANDLER (and the doctrine was never real)
Phase 8 gated `twin.ottoq_sim_advance_service_flow`, but the actual evictor is
**`twin.ottoq_sim_vehicle_exception_handler`** — 10 of 10 un-gated. Worse, the guard's
`deferred_awaiting_tech` branch was **unreachable dead code**, so
[[project_indepot_reassignment_gate]] had never actually held.
**NOW: it fires 54 times; 91 of 112 gate decisions (81.3%) protect live work; every eviction is
audit-stamped.** ⚠️ **Only HALF fixed — `severity='critical'` still auto-allows**: 21 of 58 evictions
still cut live work (19.8–113.4 sim-min destroyed).

## 📌 MEASUREMENT RULES THAT KEEP CATCHING US
- **State your OWN denominator.** A verifier could not reconstruct a prior phase's "147" — never
  silently reuse a number you cannot rebuild.
- **A new demand source changes the denominator.** assignments/arrival "rose" 1.026→1.743 purely
  because the inspect seam is new; ex-inspect it is 1.044. **Report both.**
- The harness classified **14** inspection stalls via `stall_kind` while the zone has **28**
  (`zone='arrival_inspection'`) — same numerator, double the reported utilisation.
- **Phase 8 shipped code but never finished its run** — every number read NOT MEASURED and nothing was
  preserved. **Finishing the run IS the deliverable.**

## 🔴 OPEN AT PHASE 10
cuOpt slid **42.4% → 36.4% → 18.0% raw / 28.8% like-for-like** (absolute 43→38). Supply is always the
bottleneck — diagnose from the per-invocation telemetry, and suspect a **newer path (inspect_seam /
gate_intake_staging) pre-empting cuOpt's candidates**, the same sequencing family as the original defect.
Also open: critical-severity evictions, re-plan 5→8 of 8, **parking squats 55.8% of the inspection
zone**, bay no-show 7.3%.

Links: [[project_indepot_reassignment_gate]], [[project_vehicle_first_doctrine]], [[project_cuopt_eligibility_doctrine]], [[project_booking_ledger_divergence]],
[[project_nvidia_layer_truth_2026_07_30]], [[project_full_service_visit_doctrine]],
[[project_forward_availability_doctrine]], [[project_legtype_abort_root_cause]].
