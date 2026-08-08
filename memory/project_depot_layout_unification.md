---
name: project_depot_layout_unification
description: "🚨2026-08-06: the depot existed TWICE and the copies disagreed. Nobody had ever drawn the database's — it has 54 overlapping stall pairs, 13 stalls inside a building, and 20 of 45 chargers with no drivable aisle. Founder ruling: the renderer's 3.3-acre layout is real; the DB is the source and gets corrected. Migration 0010 authored, NOT applied."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-06T12:44:00.396Z
---

# 🚨 THE DEPOT EXISTED TWICE

## THE FINDING
Two layouts, never connected: `public.stalls` (a table) and the renderer's `sitePlan.ts` (a hand-
authored code file). **A grep of the renderer for the DB's coordinate columns returns ZERO hits** — it
never reads them. Chase has only ever seen the renderer's.

**Nobody had ever drawn the database's depot.** Drawn for the first time, it fails on seven counts:
- **54 overlapping stall pairs, 76 of 150 stalls** — identically in BOTH depots
- **13 staging stalls inside the wash building**; 4 inside the BESS compound; 2 service bays inside the office
- **20 of 45 charging stalls have no aisle a vehicle can turn into** (12 ft clear; a 90° module needs 64)
- 2 of 4 solar canopies shelter no chargers; 2 stalls share one identical point; no perimeter road
- the 5 service/wash bays have **NULL** width and depth; `absolute_point` NULL on all 300 rows

**Root cause: the site is over-programmed.** The DB fence is 1.82 acres holding a program needing ~3.3.
Stalls were placed by formula into a lot with no room, so they landed inside buildings. The tell: the
5 stalls that make the DB say 35 L2 instead of 30 (`L2-STALL-21..25`) run past the end of their own
canopy and cause 10 of the 54 overlaps. Someone needed the count to be 35 and appended rows.
The renderer by contrast: **0 overlaps, 24.8 ft main aisle**, and a deliberate "corner overlap fix" in
its git history — someone was watching.

## ⚖️ FOUNDER RULINGS (binding)
1. **The 3.3-acre rendered lot (452.13 × 313.98 ft) is the real parcel.** Import the renderer wholesale.
2. **The 2 service bays inside the office are INTENTIONAL** (attached service garage) — whitelist, don't fix.
3. **Aisle standard is two-tier**: ≥24 ft two-way, ≥20 ft one-way. The single 20.5 ft one-way perimeter
   staging lane is ACCEPTED (clears the 20 ft fire-apparatus minimum). Do not raise the bar.

**ARCHITECTURE: the DB stays the SOURCE (everything that decides reads it); the renderer supplies the
CONTENT. Then renderer AND the USD generator both read FROM the DB (`ottoq_twin_depot_layout`).**
Cheap because geometry is barely read — only ~4-8 call sites, and **nothing computes a footprint,
collision, swept path or aisle width.**

## 🛑 STATUS: 0010 AUTHORED, NOT APPLIED — and it must not be applied as written
Branch `unify-depot-layout` (otto-q-core) + `layout-unify-importer` (ottoyarddepot-sim).
**The apply HALTED at preflight, correctly.** My brief said "5 stalls removed, zero references." The
file actually retires **19 per depot**, and **13 of those carry real booking history**.
`ottoq_stall_bookings.stall_id` is **ON DELETE CASCADE** ⇒ applying it would silently vaporise ledger
rows with no error. Evidence preserved in `preimport_*_20260806` (123 bookings, 300 stalls, 12 structures).
**FIX REQUIRED: re-home, never retire.** A stall may only be DELETED if it has zero references across
all 17 FK columns; otherwise UPDATE it in place. Staging is fungible — the design already said
"re-homing beats retiring," then the generator retired 12 anyway.
⚠️ "Retire in place" (status='closed') does NOT work for L2: `ottoq_plan_overnight_wave` counts
`stall_type='l2'` with **no status filter**, so a closed row keeps broadcasting capacity.

## 🕳️ FOUR HOLES IN THE GEOMETRY GUARD (the guard is the point — it must not bless bad layouts)
1. **NULL handling, ×2 (FIXED).** Postgres LEAST/GREATEST skip NULLs ⇒ the 5 dimensionless bays
   inherited their comparand's edges and produced **725 phantom overlaps** (779 instead of 54). The JS
   twin had the mirror bug (`null/2 = 0` ⇒ zero-area point) and printed **"PASS: zero overlapping
   pairs"** *plus* advice to delete all five founder exemptions as stale. Now reports **NOT ESTABLISHED**.
2. **The fence check silently skips depot 2** — it INNER JOINs one `FENCE-PERIMETER` row that only
   exists for depot 1, so depot 2 contributes nothing and the check PASSES unearned.
3. **No aisle check on the DB side at all** — aisle is measured only source-side in the JS guard.
4. **No stall-vs-LANE clearance check** — found by the motion session: `EAST_AISLE_X`=275 with a 3.2
   lane offset ⇒ **0.9u overlap on every northbound pass**. West avenue has 4.1u and no hotspot ⇒ the
   principled fix is to mirror the west. The guard would have blessed this forever.

## CONSEQUENCES TO EXPECT (direction only — we are rebuilding, not benchmarking)
Mean pairwise stall distance **144.64 → ~204 ft**: travel legs get longer, dwell rises, throughput
falls. That is a realism improvement — the old distances were computed on a lot that cannot exist.
Also fixes a latent renderer bug: staging codes were six series all restarting at 001
(`W001`/`E001`/`N001`…), and the renderer maps by trailing digits ⇒ six real stalls collapsed onto one
rendered spot and STAGE-20..115 were never addressed. A single `NASH-STG-001..115` sequence fixes it.

Links: [[project_twin_render_offmap_bug]], [[reference_depot_plan_scale]], [[project_depot_lanes_rails]],
[[project_booking_ledger_divergence]], [[project_legtype_abort_root_cause]], [[reference_visual_layer_plan]],
[[feedback_rebuild_not_benchmark]].
