---
name: project_forward_bay_reservation_0011
description: "⭐2026-08-06 migration 0011: the return signal now reserves BAYS ~30 sim-min before arrival (7/7 booked pre-arrival). Contains 3 FALSIFIED premises worth more than the build — planned_return_at is NOT the arrival estimate, service_definitions does not speak the twin's vocabulary, and the outbound-itinerary blocker does not exist."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-06T23:50:41.503Z
---

# ⭐ 0011 — THE DEPOT LEARNED TO HOLD A BAY BEFORE THE CAR ARRIVES

**Version 20260806231121**, branch `fwd-bay-reservation-0011`, applied + pushed, not merged.
Adds `ottoq_reserve_inbound_bays`, `ottoq_book_workflow_legs`, `ottoq_svc_to_stall_type`.

**What it does:** at the RETURN SIGNAL, for each bay-requiring atom on the needs card, it reserves the
bay on the shared `ottoq_stall_bookings` calendar for [predicted arrival + queue, + the service's preset
duration]. **Measured: 15 pre-arrival holds written; 7 of 7 whose vehicle genuinely arrived were booked
BEFORE arrival, median lead 30.8 sim-min.** 6 holds queued on ONE bay in 6 non-overlapping windows.
When a 7th would not fit the 240-min horizon it recorded `unbookable` + `hold_required` rather than
overbooking. **ALWAYS HOLD: 0 defects** (0 of 82 active dispatches left with pre-existing open work).
Before this, the return-signal path searched only `('dcfc','l2')` then `'staging'` — the strings
`service`/`wash`/`detail` did not appear in it at all.

## 🚨 THREE PREMISES I ASSERTED THAT WERE FALSE — do not re-assert them
1. **`planned_return_at` is NOT the arrival estimate.** It is `GENERATED ALWAYS AS dispatched_at +
   planned_duration_min` — the **dispatch-time plan**. The twin's actual arrival test is
   `sim_clock >= returning_started_at + return_eta_minutes`, i.e. the **live ETA**, which
   `ottoq_book_appointment` already computes as `v_eta_at`. Measured divergence: **median
   `plan_minus_live` = −114.9 min**. Anchoring holds on `planned_return_at` would have reserved bays
   ~2 hours before the vehicle could physically occupy them. ⇒ **Anchor on the LIVE ETA;
   `planned_return_at` is the FALLBACK.** Both anchors + their divergence are written into the booking.
2. **`service_definitions` does not speak the twin's vocabulary.** Its 9 active codes share exactly
   **1 of 15** atom names with the needs card (`exterior_wash`). Worse, `full_detail`/`interior_detail`
   require `detail_bay` and **the depot has ZERO detail_bay stalls** — unbookable by construction.
   ⇒ **`service_cadence_policy.lane` is the live requirement column** (covers all 15 atoms; already
   what `ottoq_decide_tick` 4b uses).
3. **The "outbound itinerary blocks inbound planning" blocker DOES NOT EXIST.**
   `twin.ottoq_sim_dispatch_vehicle` calls `ottoq_release_visit_artifacts` at dispatch, which skips
   planned legs and completes the itinerary. Measured: **63 of 63 `depart` legs are `skipped` on
   `completed` itineraries; ZERO are `planned`.** The guard's real case is a leftover planned `inspect`
   leg when `twin.ottoq_sim_generate_service_manifest` supersedes a visit need without closing its
   itinerary. ⚠️ Also: the old "median lead +0.05 min" figure is partly an ARTEFACT —
   `ottoq_plan_visit_itinerary` stamps `sim_created_at = p_clock` (the PREDICTED arrival), so lead
   measured that way is ~0 by construction. The evidence that stands is the **count** of pre-arrival
   bay rows, which was zero.

## 🕐 A REAL CLOCK BUG, PROVEN (8th+ of its class here)
**`ottoq_sim_stop_and_reset` writes the REAL WALL CLOCK into the SIM-domain column `actual_return_at`**
— 76 rows in one run. Read naively it inflated 0011's headline from **30.8 → 545.5 min**. The agent
caught it and reported the honest number.
⇒ **Every historical post-stop arrival metric in this codebase is suspect until checked.** Exclude
teardown-stamped rows with a sim-domain sanity bound and say how you did it.

## 📐 MEASUREMENT RULE CORRECTED
**STOP archives the run and empties the depot.** So: occupancy, coverage, placement and per-vehicle
state must be captured **BEFORE** the stop; ledger/calendar/booking metrics **after**. An earlier run
reported coverage all-zeros with `{"offline": 116}` — that is **NOT ESTABLISHED**, not 100%.

## 🔴 STILL OPEN
1. **No hold has ever been observed BINDING.** Every hold was still `state='held'` at the stop and was
   released by teardown. The depot writes the promise; nobody has watched it keep one. **This is the
   whole thesis and it is unproven.**
2. **Gate-discovered needs get no forward hold** — 1 of 18 outstanding bay atoms. 0011 hooks only the
   return signal.
3. **`ottoq_svc_to_stall_type` is DEAD CODE** — introduced as the TOTAL resolver, described that way in
   MIGRATION_LOG, but nothing calls it; the booking path still uses an inline partial CASE.
4. **The tick strains**: metronome averaged 38 s, peaked 69.5 s against a 60 s schedule, froze 2.3
   real-min at tick 149, cron 10 failed twice. 0011's forward-walk (≤24 stall searches per contended
   leg, hot path) is SUSPECTED but unproven. ⚠️ A previous slow tick was **event-write amplification**,
   not the shield, and pg_stat_statements `track=top` misattributed it — use session-scoped
   `track_functions`.
5. Wash/detail pre-arrival holds barely exercised — the seed drew almost no wash atoms.

Links: [[project_forward_service_scheduling_doctrine]], [[project_arrival_forecast_learning_doctrine]],
[[project_forward_availability_doctrine]], [[project_depot_layout_unification]],
[[project_tick_cost_root_cause]], [[project_legtype_abort_root_cause]], [[feedback_rebuild_not_benchmark]].
