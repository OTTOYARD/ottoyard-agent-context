---
name: project_booking_ledger_divergence
description: "🚨🚨2026-08-01 FIXED: the forward availability calendar (ottoq_stall_bookings) never recorded the decision — a SEPARATE search picked its own stall. Agreement was 3/81 (3.7%), now 79/80 (98.8%). ⇒ ALL booking-derived capacity/occupancy/forward-utilisation numbers are VOID; −41% grid-peak is arithmetically clean but is a PRE-FIX number."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-01T19:42:19.159Z
---

# 🚨🚨 The calendar was making its own decisions

**[[project_forward_availability_doctrine]] says orchestration IS a forward-time occupancy calendar
of every space. That calendar was wrong about which stall a vehicle used, 78 times out of 81.**

## THE MECHANISM (not what I first thought)
❌ **NOT a transposition, and NOT a swapped-argument bug.** I suspected `ottoq.ottoq_book_stall`
(`(p_sim_run_id, p_stall_id, p_vehicle_id, …)` — both uuid, so a swap would be silently legal) was
being called with the pair reversed. **REFUTED:** both call sites
(`ottoq_find_and_book_stall:9-11`, `ottoq_react_to_refusals:31-33`) pass them correctly.

✅ **TWO INDEPENDENT STALL SELECTIONS for the same vehicle:**
1. **What the vehicle does:** `ottoq_decide_tick` picks a stall (cuOpt or greedy) → `ottoq_reserve_stall`
   → emits `begin_charge`. **185/185 decisions matched their issued command exactly — reality was
   always self-consistent.**
2. **What the calendar recorded:** `ottoq_book_workflow` → `ottoq_find_and_book_stall`, which runs
   its **own** `ottoq_stall_free_between` search and books whichever free stall it finds FIRST.
Two vehicles drawing from the same small pool in different orders each land in the other's stall —
which *looks* like a swap but is two uncoordinated searches. **Guaranteed wrong by construction, not
a race.**

⭐ **AND THE AVAILABILITY ORACLE WAS BLIND:** `ottoq_stall_free_between` excluded only
maintenance/closed stalls — **never occupied ones**. So a stall a car had just plugged into still
read as empty. The ledger was wrong in BOTH directions at once: physically-full stalls looked free,
and free stalls looked booked to someone else.

## ✅ FIX + PROOF (measured, same query both phases)
The decision is now what gets written: the enacted stall goes into the calendar directly.
| | before (run 6c6fb2bb) | after (run 68d60844) |
|---|---|---|
| same-tick decision vs calendar | **3 / 81 = 3.7%** | **79 / 80 = 98.8%** |
| calendar EVER holds the used stall | 12 / 185 = 6.5% | 81 / 116 = 69.8% |
| `reservation_honoured` path | 4.1% correct | 65/67 |
| double-booked stalls | 0 | **0** (fix created no collisions) |
Rollback: `ottoq_schema_snapshots` **snapshot 6166** (pre-fix, retain permanently).
Evidence preserved in `public.booking_ledger_proof_2026_08_01` (301 decisions) +
`public.booking_ledger_proof_bk_2026_08_01` (1,108 bookings) — named **outside** the `ottoq%` prefix
so the purge cannot eat them (same trick as `cuopt_enactment_proof_2026_08_01`).

## ⚠️⚠️ WHICH NUMBERS THIS KILLS — read before quoting anything
- **VOID:** every capacity / occupancy / forward-utilisation figure sourced from the booking table.
  Not "imprecise" — *unrelated* to what the depot did, and not correctable in a known direction
  because errors ran both ways.
- **ARITHMETICALLY CLEAN but PRE-FIX: the −41% grid-peak.** MEASURED: zero views read the booking
  table, and the energy tables are written by functions that never touch it. **BUT** the booking
  table IS read by **5 live decision functions** (arrival disposition, place_unplaced_vehicles,
  opportunistic charge, replan stranded undercharge, validate_assignment), so the broken calendar was
  *steering depot behaviour* while −41% was measured. ⇒ **Re-run and re-certify, do not retract.**
  Label it a pre-fix number until a clean multi-seed run reproduces it.
  (Consistent with [[reference_ottoq_real_edge]], which already flagged it for re-verification.)
- **NOT AFFECTED:** throughput, deploy counts, vehicle-return funnel — sourced from itinerary legs
  and commands, not bookings.

## ⏭️ STILL OPEN
1. **~1/3 of decisions still leave no calendar trace** — 31 of 41 on the unlabelled path. The fix
   only books when the vehicle has a planned *charging* leg; service-only visits and holds are still
   unrecorded. Under the doctrine that the calendar IS the product, that slice is unfinished.
2. `ottoq_stall_free_between` is **still structurally blind to occupancy** for those non-charge paths.
3. One unexplained mismatch of 80.
4. **Run 68d60844's throughput/energy/timing numbers are NOT trustworthy** — cron job 12 co-drove it
   and caused 3 lock timeouts. The ledger ratio survives (per-decision), the perf numbers do not.

## 📌 OPERATING LESSON — do NOT disable cron job 12
A sub-agent recommended disabling cron 12 as a "runaway scheduler." **I rejected that: job 12 IS the
demo metronome — the engine that makes START work.** Disabling it means pressing START does nothing.
It is harmless when idle (exits immediately when no run has `status='running'`). The real rule:
**never hand-`CALL ottoq_demo_metronome` while cron 12 is active** — let cron drive, or pause it only
for a verification window and restore it after. Another instance of
[[reference_db_surgery_lessons]]: verify sub-agent recommendations before acting on them.

Links: [[project_forward_availability_doctrine]], [[project_cuopt_eligibility_doctrine]],
[[project_nvidia_layer_truth_2026_07_30]], [[reference_ottoq_real_edge]], [[project_perimeter_hold_doctrine]].
