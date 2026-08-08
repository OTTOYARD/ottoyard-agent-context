---
name: project_arrival_forecast_learning_doctrine
description: "⭐⭐CORE DIRECTION (Chase 2026-08-05): the arrival forecast is where ML/variable learning belongs — reconcile ALL variables against the actual scenario/sim variables, deduce WHEN each service should happen, then ACTUALLY RESERVE it (stall assignment + dwell time). Lean the NVIDIA/AI layer in here."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-05T22:38:35.338Z
---

# ⭐⭐ THE FORECASTING / LEARNING DIRECTION (Chase, verbatim intent)

> *"With the arrival forecast, this will be a place where training and almost machine or variable
> learning will be extremely important. It will need to **first look at all variables, and then actual
> scenario or simulation variables, and deduce between the two** when activities or services should
> take place, and then **the actionable part of reserving that within the depot from a stall
> assignment and stall dwell-time perspective**. This could also be a great time to leverage our AI
> tools through NVIDIA or anything else to train agents... You just have to make sure you're covering
> all the bases and leveraging all tools at our disposal to make a succinct orchestration and
> intelligence tool that will ultimately be OTTO-Q."*

## THE THREE-STEP SHAPE HE DESCRIBED
1. **ALL VARIABLES** — the vehicle's full standing profile: cadences, wear, mileage, SoH, soil,
   software version, fault history, deployment commitments. The *general* picture.
2. **ACTUAL SCENARIO / SIM VARIABLES** — what is true in THIS run right now: current SoC, live
   ETA, real depot contention, staff/lane capacity, live faults, weather.
3. **DEDUCE BETWEEN THE TWO** — reconcile the standing profile against live conditions to decide
   **WHEN** each service should happen (which arrival, at what time).
4. **THEN ACT** — the part he stressed as *"the actionable part"*: **reserve it in the depot**, as a
   concrete **stall assignment** with a **dwell time**. A forecast that does not end in a reservation
   is not orchestration.

⇒ This is the learned layer sitting on top of [[project_forward_service_scheduling_doctrine]]:
that doctrine says work must be predicted and reserved BEFORE arrival; this says the *prediction*
is a learning problem, and the *reservation* is the deliverable.

## ⭐ WHY THIS IS THE RIGHT PLACE FOR THE AI LAYER (my read)
Everything else in OTTO-Q is deterministic and should stay that way — the L1 shield's 52 safety rules,
the atomic-visit guarantee, the in-depot reassignment gate. **Forecasting is the one part that is
genuinely uncertain**: when will this car come back, how long will the work really take, will the bay
be free. That is exactly where a learned model earns its place and where a wrong answer is safe,
because the deterministic layer still vets the result.
Consistent with the standing architecture: **the optimizer/model proposes, the deterministic shield
allows or denies, then dispatch.** And with [[project_cuopt_eligibility_doctrine]]: cuOpt should be
built INTO OTTO-Q as the assignment engine, greedy as fallback.

## 🔴 WHAT MUST EXIST FIRST — measured 2026-08-04, all still true
The forecast has nothing to learn from yet:
- **No arrival side at all.** `ottoq_vehicle_needs_card` has 74 columns and **not one** contains
  "arriv", "eta" or "return". It has `next_deploy_at` (departure) and no `next_arrival_at`.
- **The ETA is a hardcoded constant.** `ottoq_return_eta_minutes` is one line —
  `GREATEST(1, COALESCE(ottoq_policy_get(run,'return_eta_minutes',30), 30))`. No distance, no route,
  no traffic. **True forward visibility is a flat 30 minutes**, and only after the vehicle has already
  decided to come home.
- **It destructively overwrites the dispatch plan** — 109 of 112 rows had
  `scheduled_return_at = returning_started_at + exactly 30 min`, so any plan-vs-actual accuracy check
  is circular by construction.
- **The dispatch-time plan is useless as a prediction anyway**: 0 of 112 arrivals landed within 5 min
  of it; median absolute error **68.4 min**, p90 215.5 min, mean bias +92.4 min late.
- **`ottoq_book_appointment` cannot reserve a bay.** Its stall search only queries
  `('dcfc','l2')` then `'staging'` — the strings "wash_bay" and "service_bay" do not appear in it.
  **0 of 110 needs had a bay reserved pre-arrival** against 43 bay-requiring atoms.
- **The outbound trip blocks the inbound plan.** `ottoq_plan_visit_itinerary` early-returns if an
  active itinerary with any `planned` leg exists; a deployed vehicle still owns its outbound itinerary
  with an un-executed `depart` leg, so the en-route planning call returns 0 **every time**.
  Measured: 0 of 100 itineraries built >1 min before arrival; median lead **+0.05 min** — at the curb.
- **The twin's wear model is half-built**: only `drive_km_total`, `drive_hours_total`, `soil_index`,
  `open_dtc_count` accumulate. **Tire tread, brake wear, battery SoH, sensor health, software version
  and odometer are drawn once at boot and frozen — a tire never wears.** Those are exactly the
  dimensions the founder's named services (tire/brake inspection, battery health check, software
  update) depend on.

## ⭐ MY RECOMMENDED BUILD ORDER (before any model is trained)
1. **Give the card an arrival side** — `next_arrival_at`, `minutes_to_arrival`, `arrival_confidence`.
   Join `ottoq_approach_band`; today **no function in any schema joins those two objects.**
2. **Stop the ETA overwriting the plan** — write the refreshed ETA to its own column so plan-vs-actual
   stops being circular and a training signal exists at all.
3. **Make the twin's wear model complete and stateful** so there is something real to learn from.
4. **Teach `ottoq_book_appointment` to reserve bays**, and unblock the pre-arrival planning path.
5. **THEN** learn: predict arrival time and true dwell/service duration from history, and feed
   cuOpt/Nemotron the reconciled picture. ⚠️ Nemotron REVIEWS, never decides.
⚠️ **You cannot train a forecast on a constant.** Steps 1-4 create the labels; step 5 is meaningless
without them.

Links: [[project_forward_service_scheduling_doctrine]], [[project_cuopt_eligibility_doctrine]],
[[project_appointment_depot_doctrine]], [[project_nvidia_layer_truth_2026_07_30]],
[[project_frontier_intelligence_architecture]], [[project_twin_variable_backend]].
