---
name: project_forward_service_scheduling_doctrine
description: "⭐⭐⭐CORE DOCTRINE (Chase 2026-08-04): 'overdue' is a FAILURE STATE, not a trigger. A vehicle must never leave the depot needing a service. Work is PREDICTED and RESERVED before the next arrival — forward scheduling, not reactive detection. Corrects my backwards overdue→must_do proposal."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-04T22:29:59.995Z
---

# ⭐⭐⭐ THE CORRECTION: I HAD THE ARROW BACKWARDS

I proposed **"overdue services escalate to `must_do`"**. Chase corrected it, and the correction
reframes the architecture rather than a threshold:

> *"A vehicle should never leave the depot with a service that is required or needed, **which would
> result in that service being marked as overdue**. If the service is needed, it needs to be addressed
> immediately in depot upon the vehicle's **next arrival**. So it should be **auto determined in
> OTTO-Q ahead of time** for that next arrival. So essentially it is **baked into that arrival**."*

⇒ **If work is overdue, the system has ALREADY FAILED.** Overdue is a defect signal, not the normal
trigger. This is why the `must_do` gate strangled bay recovery (14 candidates → 1, a 93% loss): I was
trying to make *overdue* work important, when overdue work should not be routine at all.
⇒ **FORWARD SCHEDULING, not reactive detection.** The workflow exists BEFORE the vehicle arrives.
This is [[project_appointment_depot_doctrine]] taken to its real conclusion.

## THE ELEVEN LOGIC POINTS (verbatim intent — treat as spec)
1. **Overdue = failure state**, never a trigger.
2. **Pre-determined and baked into the next arrival** — OTTO-Q decides ahead of time.
3. **Overflow path (exception only):** if a late/flagged service can't be routed to a bay immediately,
   the vehicle holds in **temp staging** OR in **long-term perimeter parking ASSIGNED TO IT**, until a
   bay frees and a **confirmed taxi** moves it. ⚠️ Reinforces [[project_perimeter_hold_doctrine]] —
   these are different PURPOSES, not overflow tiers. **Nothing is dropped or forgotten.**
4. **Most work is SCHEDULED, not flagged.** Flags are the edge case (user/manager/incident). Named:
   every-3rd-night exterior wash (already built) · sensor calibration · battery health check · tire
   inspection · brake inspection · misc light maintenance · software updates.
5. **Every service has a PLACE and a DURATION** — a preset bay/stall type and a timing requirement.
   His estimates: sensor calibration ~20 min, brake inspection ~30-40 min (explicitly estimates; the
   LOGIC is the point).
6. **Triggered by WEAR **or** CALENDAR** — either way it is *"programmed ahead of time and scheduled
   for reservation"* as part of the normal arrival workflow.
7. **⭐ CADENCES SPAN ORDERS OF MAGNITUDE.** A vehicle charges *several times a day*; a tire inspection
   is *quarterly or semi-annual*. Daily / weekly / quarterly. ⇒ **MOST ARRIVALS ARE CHARGE-ONLY.**
   Service is rare per-arrival yet certain over a longer horizon. Any design that evaluates every
   service on every arrival is wrong.
8. *"Every scenario needs to be explicitly written out and built into OTTO-Q. **This is the crux of
   our intelligent software.**"*
9. **NVIDIA sits ON TOP**, sorting/managing this data in real time for optimal orchestration.
10. **THE LOOP: twin GENERATES vehicle variables (randomly) → OTTO-Q SORTS/OPTIMISES → OTTO-Q sends
    orchestration directions BACK → twin PLAYS THEM OUT.**
11. *"You may need to spend more time building out the variable needs of the vehicles themselves in
    the twin world model."* ⇒ the twin's per-vehicle model likely needs deepening.

## ⭐ THE IMPLIED ARCHITECTURE (my read — confirm against the audit)
1. **Service catalog**: per service — cadence (calendar AND wear/mileage), duration, required space
   type, concurrency class.
2. **Per-vehicle service ledger**: last-done + next-due PER SERVICE TYPE. ⚠️ **Completion must write
   back** — if finishing a service doesn't advance last-done, every vehicle drifts permanently overdue
   and cadence is cosmetic.
3. **Arrival forecast**: when will this vehicle next return? ⚠️ **This is the load-bearing
   prerequisite.** If OTTO-Q only sees a vehicle minutes out, a 40-min brake inspection cannot be
   pre-booked and the doctrine needs a longer horizon.
4. **Pre-arrival planning**: compute what is due *by that arrival*, build the itinerary.
5. **Reservation**: book the bays on the forward calendar **before** the vehicle arrives.
6. **Overflow**: temp staging or assigned perimeter hold + a watcher that confirms the taxi when a bay
   frees. ⚠️ Evidence the handoff is weak today: 19 of 51 bay bookings once died
   `no_show_grace_elapsed`; `ottoq_bind_unbooked_bay_occupants` gave up `no_free_space` 34×/run.
7. **Twin plays it out** and feeds observed wear back, closing the loop.

## 🔴 KNOWN CONTRADICTION TO RESOLVE
Every bay service except `fault_repair` is `must_do=false` (exterior_wash 14,907/14,907 deferrable).
Under this doctrine that is not a "make them mandatory" fix — it is that **scheduled work should never
reach the point of being optional-vs-mandatory, because it was booked in advance.**

## ⚠️ MEASURED STATE WHEN THIS DOCTRINE WAS GIVEN
Cut-short **charging** recovery **58.3%** (7 of 12) — works. Cut-short **bay** work **0%** — the new
recovery path executed **zero times** in 10,738 decisions; `resume_candidates_seen = 0`.
Two gates caused it: the `must_do` filter (14→1) and a `staged_awaiting_service` state requirement the
one survivor never reached (it sat on a charger).

Links: [[project_appointment_depot_doctrine]], [[project_full_service_visit_doctrine]],
[[project_perimeter_hold_doctrine]], [[project_forward_availability_doctrine]],
[[project_orchestration_build_2026_08_01]], [[project_realdata_ingestion_seam]],
[[reference_ottoq_service_ops_research]].
