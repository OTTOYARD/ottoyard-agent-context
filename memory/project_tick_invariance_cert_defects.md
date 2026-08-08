---
name: project-tick-invariance-cert-defects
description: Tick-invariance cert seed 910100 is NOT a valid invariance result; completed_dispatches is teardown-inflated; visit_key lacks run scoping (real orchestration harm); arrival-event metric counts forecaster calls
metadata: 
  node_type: memory
  type: project
  originSessionId: 66e0041b-8cbc-409b-9660-c049711d61ca
  modified: 2026-07-26T05:59:27.522Z
---

2026-07-26 investigation of the seed-910100 tick-invariance cert (arms 30/10/5-min,
240 sim-min, 116 vehicles, run_by='cert_harness'). Measured live, adversarially verified.

**The "116 dispatches -> 14-21 visits" funnel gap was an ARTIFACT.** `status='completed'`
does not mean arrival. 109/111/111 of the ~116 were closed by the teardown sweep in
`ottoq_sim_stop_and_reset`. The reliable tell is NOT `return_trigger='run_stopped'` (that
catches only 95/98/100 — the sweep COALESCEs, so cars that merely *decided* to return keep
a real trigger). The tell is the CLOCK plus four arrival-only columns: teardown rows carry
`actual_return_at` = the arm's wall-clock `finished_at` with `actual_duration_min`,
`arrival_jitter_min`, `soc_at_return_pct`, `miles_driven` all NULL. Real arrivals = 7/5/8,
independently confirmed by 7/5/8 rows in `ottoq_oem_webhook_log`.
Real funnel: 116 deployed -> 21/18/19 entered 'returning' -> 21/18/19 booked (100%) ->
7/5/8 arrived -> 0/0/3 redeployed. Only ~16-18% of the fleet returns in 4 sim-hours.

**Never use `completed_dispatches` as a funnel denominator.** Use returns-initiated.
Migration `20260726043932` re-ran `ottoq_tick_invariance_metrics` at 04:39:32 — AFTER all
arms ended — overwriting the harness's deliberate pre-teardown snapshot. That backfill is
why 116 is stored, and it inflates `visits_per_dispatch`.

**REAL BUG (live harm, not just measurement): `ottoq_visit_needs.visit_key` is not run-scoped.**
`visit_key = vehicle_id || ':' || to_char(vehicles.last_state_change,'YYYYMMDDHH24MISS')`
and the constraint is `UNIQUE(vehicle_id, visit_key)` — no `sim_run_id`. The generator's
`ON CONFLICT (vehicle_id, visit_key) DO UPDATE` also never rewrites `sim_run_id`. So a later
run's booking is silently redirected into an earlier run's row. Verified: 7 vehicles
returning at the terminal tick (`...:20260902230000`, all `low_soc_reserve`, 39-44% SoC) had
their bookings absorbed by the 30-min arm, AND wiped that arm's `meta.workflow_plan` +
`meta.booked_stall_id` on exactly those 7 of 21 rows. Downstream harm: after a collision the
run-scoped manifest read inside `ottoq_book_appointment` returns nothing -> `has_charge=false`
-> no charger reserved -> low-battery vehicle parked in a staging stall on a 0-minute plan.

**Arrival-event metric measures the forecaster, not arrivals.** `event_type ILIKE '%arriv%'`
= 33/77/136 is mostly `ottoq.arrival_forecast`, emitted a hard ~2.000 per TICK by the two
engine call sites (`ottoq_energy_orchestrate`, `ottoq_opportunistic_scan`) — plus 2/8/15
out-of-band emissions from `ottoq_agent_board` polled externally over PostgREST, which makes
the metric depend on wall-clock duration and whether a cockpit was open. `ottoq_predict_arrivals`
is mis-declared `STABLE` while `PERFORM`ing `ottoq_record_event` (a write) — real defect, mark
VOLATILE, though `proparallel='u'` masked it here. `twin.vehicle_arrived` is NOT an arrival
event — `ottoq_sim_dispatch_vehicle` emits it at DEPLOYMENT start (`mode:deployment_start`,
"reusing arrival type for dispatch logging"). The true arrival event is `twin.oem_webhook_emitted`,
which fired 7/5/8 = 100% of real arrivals.

**The cert is confounded and should not be quoted as a tick-invariance result.**
`ottoq_sim_decide_and_dispatch` calls the Nemotron orchestrator each tick and it WRITES
run-scoped policy params (`updated_by='ottoq_prime'`); `ottoq_policy_get` resolves run scope
before depot, so they bind. Run-scoped params by arm: 30-min **0**, 10-min **2**, 5-min **5**.
Timing measured precisely: 3 writes landed in-run (10-min arm 1.6-1.8s before end; 5-min arm
`forecast_horizon_min=45` 34s before end), and **4 landed AFTER the 5-min arm ended** — so
they never influenced that arm but do pollute its stored scope for any replay. Either way a
nondeterministic LLM writer is active inside a determinism cert, and intervention count
scales with tick count (the independent variable).

Also: seed **903900 is contaminated** — its arms carry `run_by='tick_invariance'`, which is
NOT in the demo metronome's exclusion list (`'production_live','cert_harness'`), and arm
36ccdbcc got extra ticks (12 vs 11, 360 vs 300 declared sim-min). Do not quote seed 903900.

Cross-run mutation of `ottoq_visit_needs` is broad: `ottoq_sim_generate_service_manifest`'s
supersede and carryover UPDATEs are both unfiltered by `sim_run_id`, and at least four other
run-agnostic writers exist (`ottoq_release_visit_artifacts`, `ottoq_sweep_orphaned_visit_artifacts`,
`ottoq_benchmark_reset`, `ottoq_sim_run_scenario`). Fleet-wide, 7,442 of 7,720 superseded rows
(96.4%) are only explicable cross-run.

Diagnostic pack: `~/Desktop/OTTO-Q V1/OTTOQ_FUNNEL_GAP_DIAGNOSTIC_2026-07-25.sql`.
Related: [[project_appointment_depot_doctrine]], [[project_full_service_visit_doctrine]],
[[reference_db_surgery_lessons]], [[project_demo_loop_ops]].
