---
name: project_arrival_and_tick_doctrine
description: "⭐DOCTRINE (Chase, 2026-07-28) — depot ticks ONLY on an explicit start (random time-of-day), and arriving vehicles are never auto-parked; staging is for quarantine, a flagged inspection, or a real perimeter hold"
metadata: 
  node_type: memory
  type: project
  originSessionId: 432c576c-ab48-4d5f-8c83-c6ce3cb594e6
  modified: 2026-07-29T15:57:59.264Z
---

Two doctrine points from Chase, 2026-07-28, both built and verified.

## 1. The depot ticks only when started

> "The depot should only tick when I (or a user, or MCP) press the main start button. No sense in having it run non-stop in the background. When it starts, random time during the day should spin up."

**As-found:** `ottoq_cron_tick` (cron job 10, `*/2 * * * *`) ran unconditionally with **zero running runs** — advancing the world, POSTing to `ottoq-orchestrate-tick` (cuOpt), waking `ottoq-orchestrator-agent` every ~10 min, and committing `arrived_at_gate → staged_awaiting_service` via `ottoq-wave-admit`. ~720 solver + 144 agent calls/day against hosted APIs for an idle depot. Real money (cf. [[reference_mapbox_token_incident]]).

**Built:** a gate at the top of `ottoq_cron_tick` — `IF NOT EXISTS (SELECT 1 FROM ottoq_sim_runs WHERE status='running') THEN RETURN; END IF;`. Verified: 0 HTTP queued, returns in 9.3 ms while idle.

**Do NOT disable cron job 12 (`ottoq_demo_metronome`)** — it is the start button's engine and *already* self-gates (`EXIT WHEN NOT v_any` on runs where `run_by NOT IN ('production_live','cert_harness')`). Job 2 (weekly ingest) and job 11 (nightly retention purge) are housekeeping; leave them. Job 13 (`ottoq_cert_battery_step`, every minute) was NOT changed — still runs unattended, worth revisiting.

**Random start time:** `ottoq_start_demo_run` now shifts `sim_clock_start/current/end` by a random minute-of-day. Seed-derived (`abs(hashtext(seed::text)) % 1440`) when a seed is passed, so seeded/cert runs stay reproducible; `random()` only when unseeded. Verified: seed 910100 → 19:50 twice, seed 903900 → 10:21, 186 distinct times in 200 unseeded draws. ⚠️ `ottoq_start_demo_run` calls `ottoq_purge_prior_runs` — **calling it destroys prior run data**; do not invoke casually to test.

## 2. ⚠️ SUPERSEDED — see section 3. Vehicles are never auto-parked on arrival

> "Make sure vehicles aren't automatically staged to park when they arrive. They should mainly only return to depots to charge, clean and maybe inspect or calibrate quickly before dispatch or long term (overnight or otherwise) holding on perimeter. There obviously are exceptions to the parking, like if there is a major issue flagged that required immediate quarantine or offline status. Otherwise it can temporary stage for an inspection if flagged before technician approval and then charge/clean workflow or just straight re-deployment."

**As-found:** `ottoq_decide_tick` lines 160-170 — when `ottoq_honour_reservation_proposal` ABSTAINS (no charger free), the vehicle is parked in a staging stall as `staged_awaiting_service`. Staging was a waiting room.

**Built:** `ottoq_arrival_disposition(run, depot, vehicle, clock, long_hold_threshold_min=90)` → jsonb, advisory only. Priority order:
1. `quarantine` — atom svc in (quarantine, immobilize, tow, safety_hold)
2. `perimeter_hold` — `urgency='overnight_hold'` with a future `dispatch_due_at`
3. `temp_inspect` — **`requires_tech_greenlight` ONLY**
4. else consult the calendar for the next charger release: wait ≥ threshold → `perimeter_hold`; otherwise `wait_at_gate` consuming **no stall**

⚠️ **Calibration lesson:** my first version also triggered rule 3 on `readiness_check`/`inspection`/`calibration`, which recreated auto-parking under a new name — `readiness_check` is on **193 of 193** open visits, `must_do`, and needs no approval (it is the routine 3-min PRE-DISPATCH check `ottoq_plan_visit_itinerary:125` already schedules AFTER services — that is Chase's temp-spot use, not an arrival event). Only 6 atoms in the whole system carry `requires_tech_greenlight` (5 fault_repair, 1 cosmetic_repair). **Flag = requires_tech_greenlight.**

**Verified across the live fleet** (116 autonomous, Nashville): 103 → `wait_at_gate` (no stall), 12 → `perimeter_hold`, 1 → `temp_inspect`. Previously all 116 would have parked.

**OUTSTANDING:** `ottoq_arrival_disposition` is not yet wired into `ottoq_decide_tick` (the 28KB abstain block at lines 160-170 still parks). Also un-wired: the `ottoq-wave-admit` edge function still commits `arrived_at_gate → staged_awaiting_service`.

---

## 3. ⭐⭐ THE CORRECTION (Chase, 2026-07-28) — NEVER wait at a gate; temp staging is a TOOL

I misread section 2 and built a `wait_at_gate` outcome. **Chase rejected it outright.** His four points, verbatim intent:

**(1) No gate queues, ever.** *"Waiting at gate is inherently congestive and is exactly what we are solving to avoid. Vehicles should ALWAYS have somewhere safe to park and stage."* When he said "park" in section 2 he meant **long/perimeter park** — NOT "pull into a charge or service bay for the OTTO-Q assignment." Temporary staging is legitimate and useful.

**(2) OTTO-Q's sole purpose is PRE-ARRIVAL workflow assignment.** *"A vehicle sends note of low battery or flag, OTTO-Q reads signal and rapidly analyzes all aspects and variables surrounding the depot, and sends back a custom OTTO-Q workflow for that vehicle arrival. Never waiting at gate, always arriving with the flow accepted (for instance — charge then external wash, then temporary stage/park, then redeployment/dispatch). That is the assignment the vehicle should receive AHEAD of arrival."* The **whole sequenced workflow with reserved resources** is issued before the vehicle gets there.

**(3) Temporary staging triggers** (all sequenced "behind the scenes and instantaneously"):
   - (a) charger dies/malfunctions halfway through a charge;
   - (b) temporary rush on chargers, **or** a DCFC frees in ~10 min and that beats burning an overnight-utilized L2 → temp-stage, then re-route to the DCFC when it opens;
   - (c) flagged major issue/smell, **or** a left item flagged missing → temp staging reservation with **plenty of reserved time for tech confirmation**. Tech clears → green-light redeploy, or push a quick top-off charge/clean, which OTTO-Q pushes to the vehicle immediately for taxiing.

**(4) TWIN-side arrival-need distribution** (simulation realism, NOT an OTTO-Q decision):
   - **DAY: ~90-95%** need a charging stall **+ SIMULTANEOUS interior inspection at that same charging stall**, then a quick green-flag for redeployment.
   - **NIGHT:** "5-point inspection" — charging + interior, then **wash every 3rd night (full fleet cycles through wash every 3 days)**, then perimeter/long-term parking where a **technician does a full walk-around**, then it sits a few hours before early-morning redeployment.
   - Random flags / sensors / recalibrations / services / anomalies = **long tail, much smaller percentages.**

**BUILT:** `ottoq_arrival_disposition` rewritten — `wait_at_gate` removed entirely; every branch yields a destination. Actions: `quarantine` · `temp_stage_tech_hold` (generous window, default 120 min) · `perimeter_hold` · `proceed_to_charge` (carries `concurrent_interior: true`) · `temp_stage_await_resource` (carries `await_stall_type` + `await_until` for the re-route) · `redeploy`. Knobs: `p_long_hold_threshold_min` 90, `p_dcfc_wait_tolerance_min` 15, `p_tech_hold_min` 120.

**Flag calibration (measured twice, wrong twice before landing):** a tech hold means `requires_tech_greenlight` **or** `item_retrieval` ONLY. `readiness_check` (193/193 visits) and `triage_check` (47, all must_do, none needing approval) are ROUTINE — including either recreates auto-parking. Live result across 116 Nashville AVs: 53.4% proceed_to_charge · 19.8% redeploy · 10.3% temp_stage (service-only) · 8.6% perimeter_hold · 7.8% tech hold. Zero gate queues.

⚠️ **Postgres gotcha hit here:** `CREATE OR REPLACE FUNCTION` **cannot change a parameter list** — adding `p_zones` produced an *overload*, and the 7-arg call then failed with `42725 function is not unique`. Always `DROP FUNCTION` the old signature when changing params. (Cf. [[reference_db_surgery_lessons]].)

**OUTSTANDING BUILD** (research in flight): pre-arrival multi-resource booking of the whole workflow; temp-stage→re-route mechanics for the three trigger cases; concurrent charge+interior modelling (the planner currently serializes every atom, inflating every DAY visit); twin-side day/night need generator matching the 90-95% / 5-point / wash-every-3-days distribution; and wiring all of it into `ottoq_decide_tick` + the `ottoq-wave-admit` edge function.

---

## 4. ⭐ ROOT CAUSE OF THE GATE QUEUE (found in a live run, 2026-07-29)

First live run of the new stack (run `54312698`, started 19:06) showed the gate queue growing **monotonically 0 → 13 → 21 → 27 → 38 → 62 of 120 vehicles** while **128 stalls sat empty** — a dispatch failure, not capacity. Vehicles oscillated gate → `staged_awaiting_service` (no stall) → gate, 3+ round trips each.

**`arrived_at_gate` was being used as a WORK-QUEUE MARKER meaning "needs reassignment", not as a physical place.** Only three functions actually WRITE it (most of the 26 that mention it merely read):
- `ottoq_sim_advance_deployed_telemetry:242` — legitimate: the vehicle physically arrives.
- `ottoq_sim_advance_service_flow:37` — "STEP 0 deadlock breaker": any staged vehicle below the SoC floor with unfinished charging was pushed back to the gate with NO stall. **This was the oscillation engine.**
- `ottoq_opportunistic_scan:76` — after an approved opportunistic charge, bounces the vehicle from staged states back to the gate. **STILL UNFIXED.**

The bounce bought nothing: `ottoq_decide_tick`'s charge loop **already accepts `staged_awaiting_service`** as an eligible state (its line 123). So a vehicle can wait in a real temp stall and still be picked up for a charger.

**FIXED** `ottoq_sim_advance_service_flow` to temp-stage (booked stall, `staged_awaiting_service`) instead of bouncing to the gate. **Live result: at_gate 62 → 0, and deployed jumped 1 → 21** — the oscillation had been starving the deploy path entirely.

⚠️ Note: my earlier fix to `ottoq_sim_stop_charge_session` was correct but was NOT the cause; the fault-path vehicles were reaching the gate through the deadlock breaker. Verifying one function is not the same as sweeping all writers — always enumerate every writer of a state before declaring a state-machine bug fixed.

**STILL OPEN after the fix:** 75 vehicles in `staged_for_departure` hold no stall (should be on the perimeter overnight); `in_wash_bay` vehicles still carry `current_stall_id = NULL` (`ottoq_book_workflow` is built and proven but NOT yet called from the tick); `ottoq_opportunistic_scan:76` still bounces to the gate; expired `held` bookings are never released.

---

## 5. TWIN DAY/NIGHT ARRIVAL DISTRIBUTION — BUILT 2026-07-29

`ottoq_sim_generate_service_manifest` is the ONE generator deciding what a returning vehicle needs (271 lines). As-found it had **no day/night branch at all** — `v_hour` only set urgency, never the work. Measured 92.7% charge but only 29.7% charge+interior; washes fired in daylight (44.8% of DAY visits) more than at night (23.0%), backwards from doctrine.

**Five changes, all applied in one asserted transaction:**
1. **`v_is_night`** from a plan-tunable window (`night_start_hour` 20, `night_end_hour` 6, local America/Chicago).
2. **New atom `interior_inspection`** — `concurrency='cabin'`, `at_charge_stall=true`, est 3-5 min, fired at `day_interior_inspection_p` 0.93 / `night_interior_inspection_p` 0.95. This is the atom that pairs with charge to make the 90-95%. It is an INSPECTION; `interior_tidy` remains the soil-triggered escalation. (No inspection atom existed anywhere before — the only interior work was cleaning.)
3. **Wash night-gated + dispatch-counter gate REMOVED.** The old `v_cycles >= 1` precondition counted DISPATCHES not days, making the interval trip-dependent (2.07 days, not 3.0) and — because `ottoq_benchmark_reset` strips the counter from vehicle config — permanently shut on the benchmark depot (0 washes in 688 visits, invalidating every cert that involved bay/labour contention). Rotation is now `v_is_night AND wash_group = sim_day % 3`, i.e. a calendar third of the fleet per night, each vehicle every 3rd night. `soil_index` override kept (a filthy car washes off-rotation).
4. **New atom `perimeter_walkaround`** — night only, `night_walkaround_p` 0.90, `concurrency='hold'`, `at_perimeter=true`, est 10-15 min. Deliberately NOT `requires_tech_greenlight`: that flag routes a vehicle into a tech HOLD on arrival, and this is routine overnight work.
5. **Long tail scaled** at source via `long_tail_scale` (default 0.5) applied to sensor_clean / interior_deep_clean / sensor_calibration / mechanical_pm / cosmetic_repair, so every downstream draw inherits it.

**VERIFIED live** (run `1bbd419b`, visits generated after the change, sim at 11:01 local = DAY): 100% charge, **90.9% interior_inspection, 90.9% charge+inspection** (target 90-95% ✅), **0% wash in daylight** (was 44.8% ✅), 0% walkaround (night-only ✅).

All knobs read from the `service_manifest` feed plan, so they are tunable without code: `night_start_hour`, `night_end_hour`, `day_interior_inspection_p`, `night_interior_inspection_p`, `night_walkaround_p`, `long_tail_scale`, `wash_soil_override`.

⚠️ STILL OPEN: `ottoq_evaluate_return_need`'s wash-cadence rung still uses the dispatch-count test (`cycles_since_wash`) in two places, and `trg_dispatch_bump_wash_cycle` still increments it — so the RETURN trigger is still trip-based even though the MANIFEST is now day-based. Also the planner still lays bay jobs strictly after charging, inflating any visit with a bay job.

See [[project_perimeter_hold_doctrine]], [[project_forward_availability_doctrine]], [[project_indepot_reassignment_gate]], [[project_appointment_depot_doctrine]], [[reference_ottoq_real_edge]].
