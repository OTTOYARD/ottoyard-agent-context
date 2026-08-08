---
name: project_forward_availability_doctrine
description: "⭐⭐CORE DOCTRINE (Chase, 2026-07-28) — orchestration = a FORWARD-TIME occupancy model of every depot space, not a refusal protocol; refusal is only the exception path"
metadata: 
  node_type: memory
  type: project
  originSessionId: 432c576c-ab48-4d5f-8c83-c6ce3cb594e6
  modified: 2026-07-30T00:26:58.411Z
---

Chase's correction, 2026-07-28, after I proposed building a twin "refusal vocabulary" as the centerpiece. **He rejected that framing.** In his words:

> It's not necessarily that spots get refused or the twin refuses anything, but rather OTTO-Q can see all stalls and all spaces on the depot ground, and knows which stall will be available at what time and for how long by which vehicle. It needs to know all vehicles at any given time, what their needs are and their vehicle state data, and what's available or not available at the depot based on timing and usage or utilization. All of this data should be understood and nothing should necessarily be refused because OTTO-Q wouldn't necessarily put a vehicle in an occupied stall. Now, vehicles can be rerouted based on edge cases, or scenarios where there is more congestion than others, or system malfunctions, or hardware malfunctions — in which case the intelligence layer should then reanalyze and scan the depot for availability based on its intelligence and utilization analysis and re-plan or temporarily hold or long-term hold.

> It is partially your responsibility to research all of this as CTO for a fleet orchestration intelligence layer. You should analyze this and make sure it is fully baked into our intelligence layer because that's the whole point of orchestration.

**What this means for design:**
- The core artifact is a **forward-time resource calendar** over every stall/bay/space: who occupies it, until when, who holds it next. Not point-in-time "is it free now."
- Demand side: every vehicle's needs + state, continuously, so plans are made against real requirements.
- **Refusal is not the mechanism — complete forward knowledge is.** A correct planner never sends a car to an occupied stall, so a "no" should be rare.
- Refusal/re-plan is the EXCEPTION path only: congestion above model, system malfunction, hardware malfunction. Response is re-analyze → re-scan availability → re-plan, temp-hold, or long-term-hold (see [[project_perimeter_hold_doctrine]]).
- This supersedes the "12-code closed refusal vocabulary" plan from the boundary audit; that vocabulary shrinks to a small exception set hanging off the calendar.

**Verified state as of 2026-07-28** (16-agent adversarial research; earlier looser claims corrected here):
- `ottoq_reoptimize_reservation_book` is **NOT a reservation book**. It is a greedy point-in-time grab: vehicles under 45% SoC take whatever DCFC is free *this instant*, `LIMIT 10`, wrapped in `EXCEPTION WHEN OTHERS` so it fails silently. No start time, no end time, no overlap test.
- A timeline DOES exist: `ottoq_itinerary_legs`, 47,502 rows, all with `planned_start_sim`/`planned_end_sim`; 11,810 carry `to_stall_id`. "Who holds stall S between T1 and T2" is answerable.
- But **95.3% of stall-bound charge legs (9,574 of 10,041) overlap another leg on the same stall.** Zero exclusion constraints existed in the schema. The calendar recorded intent without exclusivity.
- **Nothing reads it as intervals**: searching every `pg_proc` body for `tstzrange|tsrange|OVERLAPS|&&` returns only PostGIS geometry functions. Every decision path is present-instant.
- **Zero forward binding at plan time**: all 1,525 `planned`-status legs have no stall; `to_stall_id` is stamped at execution by `ottoq_itin_leg_open`. Forward holds DO exist separately via `stalls.reserved_by` written pre-arrival by `ottoq_sim_prearrival_contracts` (ETA-derived TTL, ~25 stalls held live) — but that shape is one mutable hold-from-now with NO start time, so it cannot express "free 09:00–11:00, taken 11:00–13:00".
- Coverage gap: stall binding exists only for charge/taxi/depart legs. ALL service legs are NULL — wash 2,812, detail 1,492, service 1,398, inspect 11,780, and **`stage` 5,281 legs averaging 5.9 hours (the overnight hold itself)**.
- ⚠️ CORRECTION to an earlier claim: reservation expiry **IS** honored, lazily at claim time, by 13+ functions incl. the live tick and the HW.004 safety rule. What is missing is garbage collection. The "62 of 62 expired" figure does NOT generalize — all 62 are debris on the *benchmark* depot 22222222; production depot 11111111 is 150/150 clean. Real defect: 3 functions read `reserved_by IS NULL` WITHOUT the expiry guard and so under-count free capacity — `ottoq_cuopt_refresh`, `ottoq_report_charger_fault` (starves fault recovery: 10 of 17 free DCFC visible), `ottoq_reoptimize_reservation_book`.
- **No re-plan exists.** `ottoq_plan_visit_itinerary` refuses to rebuild an itinerary with any unfinished leg, so after a disruption the car gets a new stall and keeps the stale plan. The 79,334 decisions logged `itinerary_amended` are a uniform time-slide with no resource check.
- ⚠️ **[[project_indepot_reassignment_gate]] is UNENFORCED**: exactly one function calls the guard, passing the argument that auto-approves, and never reads the result. The highest-volume in-depot move (staging→charging, **61,378 enacted**) bypasses it entirely.
- ⚠️ Safety bug: rule HW.003 sensor-liveness has no lower bound on age, so future-dated telemetry fails OPEN ("SOC fresh: -3,194,485 seconds old"). 116 vehicles currently carry future-dated SoC timestamps.

**BUILT 2026-07-28:** `public.ottoq_stall_bookings` — the forward calendar. One row per (stall, vehicle, purpose, `during tstzrange`), with `EXCLUDE USING gist (sim_run_id WITH =, stall_id WITH =, during WITH &&) WHERE state IN ('held','active')`. Double-booking is now physically impossible within a run; parallel runs are isolated. Proven live: adjacent bookings accepted, overlapping rejected with `23P01`, cross-run accepted. Also built `public.ottoq_fn_definition_backups` as the restore point for function surgery.

## ⭐⭐ CORRECTION 2 (Chase, 2026-07-29) — THE TWIN NEVER CHECKS AND REFUSES

After step 2 shipped precondition-checking inside the twin's confirm loop, Chase rejected that placement outright:

> The twin should never check the world and refuse. Twin is essentially a cluster of world variables that OTTO-Q is responsible for orchestrating. It is literally OTTO-Q's sole purpose to optimize and orchestrate and never suggest a reservation or assignment or workflow that will conflict with another vehicle, or a stall that is malfunctioning or out of service or blocked for any reason. The twin basically just sends a set of facts from a vehicle perspective — and if the simulation has already spun up, there will be several vehicles already assigned to certain stalls, but OTTO-Q will recognize this and distribute vehicles correctly and efficiently, never conflicting with any other assignments or reservations. Make sure that is rock solid and coded into OTTO-Q and not within the twin's reasoning. The twin never makes decisions like that. The twin only sends variable signals and scenario constraints to OTTO-Q when a new run is started.

> When the run starts, signals go to OTTO-Q like "67 vehicles out dispatched and ride hailing" with randomly generated needs, SoC, telemetry (ETA, traffic, weather). Loaded in the TWIN, sent to OTTO-Q; OTTO-Q assigns stalls and service workflows based on vehicle needs and depot capacity. Grid/battery discharge factored at a macro level. Without OTTO-Q, vehicles arrive and must be parked first, teleoperators talk to technicians, etc. — the intelligence layer solves that downtime and disorganization.

**Architectural consequence:** conflict validation runs at ISSUANCE, inside OTTO-Q — the emit choke point pre-flights every stall-carrying instruction against live occupancy, live reservations, stall status, charger state, and the forward calendar. A conflicting instruction is refused BY OTTO-Q AT EMIT (`confirmed_by='otto_q_preflight'`), never recorded as the twin's judgment; the reaction loop then re-plans it. The twin's confirm loop is a pure executor: delay, apply, report — zero world-evaluation, zero refusal logic. Refusal STATUS survives on the ledger, but it always means "OTTO-Q caught its own conflict pre-flight," not "the twin said no."

**Design frame (prior art):** interval scheduling on parallel dedicated machines with machine calendars — same shape as airport gate assignment, hospital OR scheduling, container-terminal berths. Enforce hard exclusivity in Postgres; do the packing in a solver OUTSIDE the database (OR-Tools CP-SAT: interval vars, NoOverlap, cumulative). **cuOpt is routing/LP only — no interval or no-overlap primitives, integer solver in beta and cannot prove optimality; do not use it for calendar work** (cf. [[reference_nvidia_ai_integration]]). Re-plan on a rolling horizon with a frozen near-term window.

Related: [[project_appointment_depot_doctrine]] (depot is appointment-only; vehicle signals return → OTTO-Q replies with workflow + reserved stalls), [[project_indepot_reassignment_gate]] (no in-depot re-route without tech approval), [[project_full_service_visit_doctrine]].
