---
name: project_needs_probability_draw_doctrine
description: "⭐⭐CORE (Chase 2026-08-05): brake/tire/battery needs are a PROBABILITY DRAW at run start (~1 in 5-10 vehicles), NOT accumulated wear. Demos are minutes and sims cover ~24h, so long-horizon wear modelling is a non-requirement. Mileage/calendar triggers get built as DORMANT config for real fleets later."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-05T23:07:05.152Z
---

# ⭐⭐ NEEDS ARE A DRAW, NOT AN ACCUMULATION (Chase, verbatim intent)

> *"Since the twin simulator will spin up new variables each time, the condition of these three
> variables on any given vehicle will probably just need to be **randomly selected**. Since I won't be
> running the simulation for days or weeks at a time, it doesn't really matter... You can just
> randomly fill in the **probability** of a brake check or a health check or tire check into each
> variable spin up when a simulation is started. So there might just be **a 1 in every 5 to 10 vehicle
> need** for a specific service that will display upon that vehicle arriving for that simulation run.
> When a new simulation run has started they will be **completely new probabilities** selected for that
> variable set. **Just like vehicle state of charge, arrival times, other needs, available stalls,
> energy/grid consumption**... Point being, it will just be **another variable selected**."*

## ⭐ THIS CORRECTS A GAP I RAISED THAT ISN'T ONE
I flagged as a defect that *"tire tread, brake wear, battery SoH, sensor health, software version and
odometer are drawn once at run boot and frozen — a tire never wears."* **For this product, that is the
CORRECT design, not a bug.** Demos run for minutes; sims cover ~24 sim-hours. Nobody will ever observe
a tire wearing. Accumulating wear over sim-time models something unobservable at real cost.
⇒ **DELETE "make the twin's wear model stateful" from the build order.** It was solving a
non-problem. The draw produces identical observable behaviour for a fraction of the work.

## THE RULES
1. **Per-run probabilistic draw.** At run start each vehicle gets its service needs drawn. Target
   density ~**1 in 5-10 vehicles** carrying a specific bay service for that run.
2. **New run ⇒ completely new draw.** Same class of variable as SoC, arrival times, stall availability
   and energy/grid — i.e. it belongs in the existing variability machinery, not a new subsystem.
3. **Seed-deterministic.** Same seed ⇒ same draw, or A/B testing dies. (Non-negotiable; the existing
   `prime_fingerprint` discipline applies.)
4. **Horizon is ~24 sim-hours, often much less.** *"I won't need the ability to show what will happen
   six months from now."* Demos are several minutes, optionally sped up.

## 🔮 THE FUTURE PATH — BUILD THE SHAPE, LEAVE IT DORMANT
> *"In the future, we can begin to predict that variability just on timing. So when a vehicle hits
> 5,000 miles, it gets a battery check, or when the vehicle hits 10,000 miles it gets a tire check. Or
> even more simply just every quarter or month or semi-annual checkup... but that's all in the future
> when there's actual vehicles that will be running long periods of time."*
> *"For now just probability for simulation purposes, and in the future or on the back end have a place
> where we can **select triggers or thresholds for services to be automatically queued via OTTO-Q,
> probably in the vehicle preferences or settings**."*

⇒ Build a **threshold/trigger config surface** (mileage AND calendar) in vehicle preferences/settings
that OTTO-Q *would* read to auto-queue services — wired but **not driving the sim**. The sim keeps
using the draw. When real vehicles run for months, flip the source from draw → thresholds with no
re-architecture. ⚠️ Note `service_cadence_policy` (15 rows) already carries `interval_h`/`interval_km`
— that IS the threshold surface; it needs a per-vehicle override/preferences layer, not a rebuild.

## ⇒ WHAT THIS MEANS FOR THE FORECAST LAYER
The forecast is **NOT** "predict wear months out." It is:
**given this run's drawn needs plus live conditions, decide WHICH ARRIVAL each service happens at,
and reserve the stall + dwell time for it** — inside a ~24-hour window.
That is far more tractable and matches [[project_arrival_forecast_learning_doctrine]]'s three-step
shape (all variables → this run's variables → deduce → reserve).

Chase on the bar: *"This layer will truly need to be pretty frontier. So make sure you're going to
depth with this stuff."*

Links: [[project_arrival_forecast_learning_doctrine]], [[project_forward_service_scheduling_doctrine]],
[[project_twin_variable_backend]], [[project_cuopt_eligibility_doctrine]], [[project_demo_loop_ops]].
