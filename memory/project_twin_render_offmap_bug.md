---
name: project_twin_render_offmap_bug
description: "✅RESOLVED 2026-08-06: the off-map render bug does NOT reproduce on main (0 sightings, 116 vehicles). The REAL motion bug was LaneGraph.route() offsetting its own endpoints, teleporting cars 3.2u into a neighbouring stall at a 5.7u pitch — causes BOTH pile-up and permanent wedging. Measure body overlap, never centre distance."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-06T04:15:35.512Z
---

# ✅ THE OFF-MAP BUG IS RESOLVED — AND THE REAL BUG WAS SOMETHING ELSE

## ⚠️ CORRECTION TO THIS FILE'S PREVIOUS CONTENTS
This note previously asserted "DEFINITIVE: the motion complaints are a RENDERER bug (parks off-map
deployed cars)." **That is no longer true and was handed to a fresh session as an established fact,
costing it a wasted starting hypothesis.** Verified 2026-08-06 against a captured real run
(`busy_day`, **116 vehicles, 31 snapshots, 465 geometry samples**): **0 off-map sightings.** Deployed
cars are correctly dropped from the scene; an off-site car is only ever drawn while actively driving
out the gate. Fixed by motion work already merged to `main`.
⇒ **Lesson: re-verify a stored "established fact" before passing it to another session as a premise.**

## 🎯 THE REAL BUG — `LaneGraph.route()` displaced its own endpoints
`route()` applied the drive-on-the-right lane offset to the **whole polyline including both
endpoints**. But `from` is where the car physically IS and `to` is the exact point it must reach —
only the interior road vertices are centrelines. Measured drift: **3.20 units on both endpoints**,
against a **5.7-unit stall pitch**.
**One bug, both symptoms:**
1. A car starting a route **teleported 3.2u sideways into its neighbour's stall** ⇒ "piling into
   each other."
2. From that displaced start its own path ran *inside* the parked neighbour, so the car-following
   model read a **negative gap and pinned speed to 0**. Every wedged car sat at `s=0` holding no lock,
   and the 45-s watchdog rebuilt the identical route from the identical spot ⇒ "never drains."
Fix: restore the two endpoints after the shift; interior lane offsets unchanged.
Separately: the gate queue **spawned cars inside each other** (8u spacing for a 10.2u car). Fixed with
pitch 11 + a queue bound (4/poll, 5th deferred not dropped).

**Measured on the fixture:** body-overlap samples **543 → 221**; wedged-car samples **297 → 130**;
distinct overlapping pairs 110 → 90; off-map 0 → 0. 237/237 tests pass. **Not zero — do not claim it.**

## ⭐ METHOD RULES EARNED HERE
- **Measure ORIENTED BODY OVERLAP (the 4.2 x 10.2 box actually drawn), never centre distance.**
  Perimeter stalls are pitched 5.7u apart, so a distance test flags every pair of parked neighbours —
  the first pass reported **31 phantom collisions** that way.
- **Sample DURING motion, not after the scene settles.**
- **Replay a captured fixture, not a live run.** These bugs are intermittent; a frozen recording makes
  a fix falsifiable. Fixture: `src/engine/__fixtures__/twinRun.busyday.json` + `replay.ts`.

## 🚫 MEASURED WORSE — DO NOT RE-PROPOSE
- **Routing cars onto the parking access aisle before departing** (mirroring the bay-exit manoeuvre):
  305 → **728** overlap samples. It funnels cars onto the aisle *centreline* into oncoming traffic —
  and the rail stepper deliberately ignores oncoming cars as leaders, so they drive through each other.
- **Correcting the parked heading** so temp-block cars nose away from their aisle: 305 → **378**.
- Still standing: **do NOT enable `separationSteer`** (see [[reference_depot_plan_scale]]). The real
  causes were dull. No A*, no Dubins paths were needed.

## 🔴 OPEN — needs a layout decision, not a motion fix
**East-avenue right-of-way conflict.** `EAST_AISLE_X`=275 with lane offset 3.2 ⇒ northbound lane body
spans x 276.1–280.3, while an E-column car parked nosing east spans 279.4–289.6 ⇒ **0.9u of overlap on
every northbound pass.** Now the dominant residual hotspot. Options: move the carport (layout) or drop
the lane offset to ≤2.3 (it was deliberately raised for passing clearance and the lane paint tracks it).
West avenue has 4.1u clearance and shows no hotspot.
⚠️ **This exposes a HOLE IN THE GEOMETRY GUARD** built for [[project_depot_layout_unification]]: it
checks stall-vs-stall and stall-vs-structure but **NOT stall-vs-lane**. Add lane clearance to the guard.

Links: [[project_depot_lanes_rails]], [[reference_depot_plan_scale]],
[[project_depot_motion_timing_realism]], [[project_parallel_session_tree_collision]].
