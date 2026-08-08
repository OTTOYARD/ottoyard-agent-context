---
name: reference_depot_plan_scale
description: Depot plan-space scale (1 unit = 0.4785 m) and the lane/car geometry budget that every renderer must tie to — the numbers that caused the head-on artefact.
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

**THE YARDSTICK: 1 plan unit = 0.4785 m (1.57 ft).** The 2D plan (`src/lib/sitePlan.ts`, viewBox `0 0 300 220`) is drawn to REAL dimensions, and the 3D frame is **1:1 with plan units** — `coordUtils.toWorld` is `x3d = x2d - 150; z3d = 110 - y2d` with NO scale factor. Triple-confirmed by geometry nobody tuned for it: PARK_RUNS `dy: 5.7` = 8.95 ft stall width, StagingZone stripes at ±2.9 = 9.11 ft, stall depth 10.5u = 16.5 ft. A 9.1 × 16.5 ft bay is textbook.

**THE LANE BUDGET.** `LaneGraph.rightOffset = 2.4`, and `addRoad()` puts BOTH directions on ONE centreline each offset right — so two opposing drive-lines are **4.8 units (2.30 m) apart**. `lanePaint.LANE_PAINT_WIDTH = 4.8` matches by construction and the SVG stroke is centred on the drive-line, so the two painted ribbons tile edge-to-edge with zero gap/overlap. **The paint has always been correct.**

**THE BUG THIS EXPLAINS (fixed 2026-07-18):** the 2D car body in `VehicleDot.tsx` was `5.0 × 8.0` units — WIDER than the 4.8-unit lane separation. Two cars passing in opposite directions overlapped by 0.20u of body (0.60u including the 0.4 stroke). **Clearance was NEGATIVE** — they tracked their lanes perfectly and still collided. That is exactly Chase's "sometimes it looks like they're heading right for each other." Now **4.2 × 10.2 units (2.01 m × 4.88 m)**, stroke 0.25 → 0.6u (0.29 m) of daylight.

**STILL INCONSISTENT — four different cars exist (deferred, needs a motion re-cert):**
- 2D body: 4.2 × 10.2 u (fixed, correct)
- 3D `Vehicle3D` body: `BoxGeometry(2.2, 0.85, 4.9)` — authored in METRES and dropped into unit-space, so it renders at ~48%: a **toy car**. Should be ≈ 4.2 × 10.2 u (uniform group scale ≈ 2.0 is the cheap fix).
- physics `traffic.ts CAR_LENGTH = 9` (4.31 m) — a third length used by IDM car-following.
Changing `rightOffset` or `CAR_LENGTH` moves routed motion, so those need a certified pass, not a drive-by edit.

**DO NOT re-enable `separationSteer`** (`traffic.ts:74`) as-is: it is currently dead code, and its `radius = 5.5` is LARGER than the 4.8u opposing-lane separation, so every oncoming car would trigger up to 0.3 rad of steer — it would MANUFACTURE the head-on swerve. Lateral position is otherwise unclampable-by-construction: the pose IS the arc position on the offset polyline (`RailFlow.pointAt`), so steering cannot push a taxiing car off its lane.

**KNOWN PRE-EXISTING:** `LaneGraph.route` offsets the car's own start point along with the polyline, so a mid-drive re-route snaps the car **2.4 units sideways** — half the lane separation, i.e. visually into the oncoming lane. Proven byte-identical with and without pause (2.3994944851328492). Not yet fixed.

Links: [[project_depot_lanes_rails]], [[project_depot_motion_timing_realism]].
