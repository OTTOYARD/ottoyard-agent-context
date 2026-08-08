---
name: project_needs_draw_already_exists
description: "🚨2026-08-07 MAJOR REFRAME: the per-run seed-deterministic needs draw Chase asked for ALREADY EXISTS and already hits his density band. Wash bays were empty because of a NIGHT GATE discarding 34 due washes, not thin demand. Seed 424242 is pinned so every run is identical. Do NOT build a new draw."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-08T03:21:06.742Z
---

# 🚨 DO NOT BUILD A NEEDS DRAW — IT EXISTS AND IT WORKS

## ✅ WHAT ALREADY EXISTS (I asserted it didn't; I was wrong)
`public.ottoq_run_boot_draw` → `public.ottoq_seed_vehicle_need_profiles` runs at **every run start** and
draws ~24 per-vehicle condition variables — **including `tire_tread_mm`, `brake_wear_pct`,
`sensor_health_pct`**, i.e. Chase's literal *"tire check, brake check, health check"*. Seed-deterministic
via `ottoq_sim_seeded_random(seed, 'vnp:x:'||id)`, set-based, order-independent, stamped
`drawn_for_run`/`drawn_seed`/`drawn_at_sim_clock`, PK'd per vehicle so each boot overwrites.
**Measured density vs Chase's 1-in-5-to-10 band:** pm due **25.9%**, calib **33.6%**, wash **29.3%**,
deep-clean **15.5%**, fault ≤sev3 **9.5%**, tire <4 mm **17.2%**, brake ≥65% **21.6%**. Already in band.
⇒ The mechanism is built. **It is simply not the thing that emits the atoms — the manifest is, and the
manifest reads a different table.**

## 🔴 WHY WASH BAYS SAT EMPTY — a GATE, not thin demand
`twin.ottoq_sim_generate_service_manifest`'s wash condition is
`(v_is_night AND wash_group = sim_day % 3) OR soil_index >= 0.75 OR cycles_since_wash >= 9`.
Measured on run `6f7518ef` (busy_day/424242, 98 visits, fleet 116):
- **34 of 116 vehicles were DUE a wash. 0 `exterior_wash` atoms were emitted.**
- Run spanned **09:08–19:18 America/Chicago; 0 of 98 arrivals fell in the night window** ⇒ gate closed
  for the entire run.
- Both escape hatches unreachable: max `soil_index` **0.4417** vs a 0.75 override; `cycles_since_wash`
  is drawn **2–5** vs a backstop of 9.
⇒ All 3 wash bays idle **by construction**. Opening the gate alone puts **~35 jobs into 3 bays**.
⚠️ Corroborated as persistent: 0009's header recorded *"exterior_wash → 0 atoms, ever."*

## ⛔ TWO PREMISES OF MINE THAT WERE FALSE
1. *"Thin demand is why the forward reservation found empty bays."* **FALSE for the service lane** — that
   same run carried **34 service-lane jobs against 2 service bays** (23 pm + 6 calib + 4 fault + 1 cosmetic).
   Demand was already heavy. If completions still found empty bays there, the cause is **timing or the
   assignment path**, not volume. Adding density would have buried that, not revealed it.
2. *"`interior_deep_clean` is barely drawn."* **FALSE** — 11 of 98 visits = **11.2%**, already ~1 in 9.

## 🐛 THREE DEFECTS FOUND
1. **SEED 424242 IS PINNED.** Every recent run in `ottoq_sim_runs` uses it ⇒ *"completely new
   probabilities each run"* is **false today regardless of the draw** — the world is identical every time.
   The pin was NOT located. **Find it before judging any draw work.**
2. **`service_cadence_policy.seed_phase_max` is decorative.** It holds 1.45/1.18/1.45/1.50 — the exact
   constants **hardcoded inside** `ottoq_seed_vehicle_need_profiles` — but the seeder **never reads the
   column**. Tuning it changes nothing while appearing to. Same shape as the vacuous guards.
3. **`prime_fingerprint` is NOT a determinism mechanism** (I said it was). No such function/column/table —
   it is a local variable emitted into an event payload. Determinism is enforced solely by
   `ottoq_sim_seeded_random` and `ottoq_crn_draw`, both IMMUTABLE over `hashtextextended`.

## ⚠️ DENSITY STACKING — the trap if anyone does add a draw
`busy_day` applies `pm_km_scale 0.010` and `calib_h_scale 0.02`, crushing `pm_interval_km` 8000→**80 km**
and `calib_interval_h` 250→**5 h**. **That** is what produces today's 23.5% mechanical_pm — not any
probability. A new draw on top would DOUBLE-COUNT and look like it over-drew. Scenario scales must return
to 1.0 in the same change, or one path be explicitly subordinated.

## ⇒ THE ACTUAL WORK (small, not a rebuild)
1. **Open/soften the wash night gate** so daytime demo runs can emit the washes vehicles are already due.
2. **Find and unpin seed 424242** so runs genuinely differ.
3. **Make `seed_phase_max` actually read**, or delete the column so it stops lying.
⚠️ Wash and detail share the SAME 3 wash bays — there are **ZERO `detail_bay` stalls**. Opening the wash
gate tightens that lane more than the parameter suggests.
⚠️ ALWAYS HOLD × oversubscription = a run that looks deadlocked but is only over-parameterised. Size the
service lane to ~75% of bay-minutes; add a boot-time assertion on projected vs available bay-minutes.

Links: [[project_needs_probability_draw_doctrine]], [[project_forward_bay_reservation_0011]],
[[project_forward_service_scheduling_doctrine]], [[project_full_service_visit_doctrine]],
[[feedback_rebuild_not_benchmark]].
