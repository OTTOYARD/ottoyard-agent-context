# 06 — Doctrine

**These are founder rulings, not preferences.** Each one was stated by Chase, usually while
correcting a design Claude had proposed. Violating one is a defect, not a style difference. Where a
doctrine and the current code disagree, **the code is wrong** — that gap is the work.

Each section names the memory file with the full record.

---

## D1 — The separation law
`memory/project_realdata_ingestion_seam.md`

> **OTTO-Q decides. OTTO-TWIN executes and owns world state. The renderer only draws.**

Any decision-layer code that mutates world state is a defect on sight. Any renderer code containing
world logic is a defect on sight.

This law was written after a measured failure: `ottoq_decide_tick` (brain) was writing
`vehicles.current_state` directly while the twin's rate-limited dispatcher actually performs
departures. The brain committed ~20 per tick, the twin could release 6, and the surplus fell into
`en_route_to_deployment` limbo — outside both the service flow and the dispatch flow. **43/33/34
vehicles lost per run.**

**Naming discipline:** anything `ottoq_sim_*` is twin-side and replaceable at production. Decision
functions (`decide_tick`, the shield, `book_appointment`, `evaluate_return_need`, `orchestrate`, the
planners) are brain-side and **must never read sim-only artifacts directly** — the brain reads their
*effects* via telemetry, wear, and needs.

**The seam is the data tables and the command bus, not the generators.** OTTO-Q reads
`ottoq_telemetry_packets`, `ottoq_vehicle_dispatches`, wear/needs state, and emits via
`ottoq_vehicle_commands` + `ottoq_comms_messages`. Real ingestion must be a **parallel writer into
the same tables and shapes** — never a new consumer path in the brain.

**The production cutover is a feed swap, not a rewrite.** `depots.feed_mode` (`'sim'` | `'external'`)
is built and certified: external mode switches off every twin fabricator (telemetry, charge
sessions, site energy, wear, heartbeats, prearrival, mock EMS, fault injectors) and keeps the entire
brain loop running. `ottoq-ingest` (edge function, v6) is the production door — five streams
(telemetry / arrival / ocpp / energy / incident), stamping `data_source='production'`.

---

## D2 — The depot is appointment-only
`memory/project_appointment_depot_doctrine.md`

**A vehicle never arrives with nothing to do.** If it is healthy and needs nothing, it stays
deployed, driving and earning — returning a healthy car is lost revenue. It should not even *render*
at the depot with nothing to do. **Arrival is the fulfilment of a decision already made, never the
trigger for one.**

**The handshake** — a request/response protocol, not OTTO-Q polling:
1. The **vehicle** sends a *"return to depot"* notification **with its telemetry**. This is the
   trigger. Reservations occur **only** at this moment — never earlier, never on a timer.
2. **OTTO-Q instantly analyses** the telemetry and builds that vehicle's **arrival workflow**
   (e.g. "charge + interior clean").
3. OTTO-Q **reserves the necessary stalls** and **communicates the selections back to the vehicle**.
4. The vehicle **knows where each stall is** and travels to each assignment itself, advancing once
   each stage is confirmed complete.
5. OTTO-Q may **reassign mid-flight** on malfunction, interruption, or delay, and communicates the
   change. The vehicle receives and responds.

**Consequence: a charging stall held empty for an inbound reserved vehicle is CORRECT AND
DESIRABLE, not waste.** Chase explicitly rejected "book only on arrival" (that *is* the broken
behaviour) and rejected borrowing-until-close. **The hold during transit is intended.**

**The defect this explained:** on one measured run, **99.0% of returns (390/394) fired on a fixed
timer** at average SoC 70.3, with **zero** pending service steps — contract-less gate-crashers
competing for stalls promised to real appointments. That was the source of the 82-car gate pileup.

**Charge policy:**
- Out driving → route back to charge **only once a LOW threshold is hit.** Topping up is never a
  return trigger.
- Top-up is allowed only under a certain SoC at certain times, depending on demand.
- **Charger class: DCFC first**, unless all occupied. L2 for longer/overnight needs.
- **Opportunistic top-off:** if a car is already at the depot for another reason and is at or below
  ~70% SoC, top it off before deploying.

**Deploy floor: 80%**, and it must be a real **per-operator SLA**. ⚠️ `ottoq_fleet_operator_slas` is
currently **empty**, so every "80" in the proposer, manifest generator, and SLA.001 is an
undefended hardcoded fallback.

**Contention rule:** if a need arises mid-shift and no slot is free for hours, the car **comes home
early with margin and waits in staging.** Safety first over revenue; never risk an unsafe SoC.

**The real overnight wave schedule:**
- **22:00–00:00** — vehicles begin returning for overnight hold, in **timed waves according to
  charge needs**.
- **~1% of the fleet stays out**, returning finally at **03:00**.
- Overnight vehicles run all normally required service queues OTTO-Q has acknowledged:
  charge → clean → maybe service → **park for final inspection**.
- **Redeploy in waves from ~04:30–05:00.**

---

## D3 — Forward availability, not refusal
`memory/project_forward_availability_doctrine.md`

Chase rejected a design centred on a twin "refusal vocabulary". In his words:

> *"It's not necessarily that spots get refused… but rather OTTO-Q can see all stalls and all spaces
> on the depot ground, and knows which stall will be available at what time and for how long by
> which vehicle. It needs to know all vehicles at any given time, what their needs are and their
> vehicle state data, and what's available or not available at the depot based on timing and usage
> or utilization. All of this data should be understood and nothing should necessarily be refused
> because OTTO-Q wouldn't necessarily put a vehicle in an occupied stall."*

⇒ **The core artifact is a forward-time occupancy calendar over every stall, bay, and space:** who
occupies it, until when, who holds it next. **Not** point-in-time "is it free now."

⇒ **Refusal is not the mechanism — complete forward knowledge is.** A correct planner never sends a
car to an occupied stall, so a "no" should be rare. Refusal and re-plan are the **exception path
only**: congestion above model, system malfunction, hardware malfunction. The response is
re-analyse → re-scan availability → re-plan, temp-hold, or long-term hold.

**Amendment (2026-07-29):** the **twin never checks-and-refuses.** Validation belongs to OTTO-Q,
pre-flight (`ottoq.ottoq_validate_assignment` called by `ottoq.ottoq_emit_vehicle_command`). A
conflicting instruction is recorded `refused / confirmed_by='otto_q_preflight'` at issuance and
never reaches the twin. `public.ottoq_sim_confirm_commands` is a **pure executor**.

**Measured state when this doctrine was given, for calibration:** 95.3% of stall-bound charge legs
overlapped another leg on the same stall; **zero** exclusion constraints existed; searching every
`pg_proc` body for `tstzrange|OVERLAPS|&&` returned only PostGIS — **every decision path was
present-instant.** Much of this has since been fixed (the EXCLUDE GIST guard, atomic booking). Much
has not.

---

## D4 — Forward service scheduling: "overdue" is a failure state
`memory/project_forward_service_scheduling_doctrine.md` — ⭐⭐⭐ **the highest-rated doctrine in the corpus**

> *"A vehicle should never leave the depot with a service that is required or needed, which would
> result in that service being marked as overdue. If the service is needed, it needs to be addressed
> immediately in depot upon the vehicle's next arrival. So it should be auto determined in OTTO-Q
> ahead of time for that next arrival. So essentially it is baked into that arrival."*

⇒ **If work is overdue, the system has ALREADY FAILED.** Overdue is a defect signal, not the normal
trigger.
⇒ **Forward scheduling, not reactive detection.** The workflow exists *before* the vehicle arrives.

**The eleven logic points (treat as spec):**
1. **Overdue = failure state**, never a trigger.
2. **Pre-determined and baked into the next arrival** — OTTO-Q decides ahead of time.
3. **Overflow path (exception only):** if a late/flagged service cannot be routed to a bay
   immediately, the vehicle holds in **temp staging** *or* in **long-term perimeter parking assigned
   to it**, until a bay frees and a **confirmed taxi** moves it. **Nothing is dropped or forgotten.**
4. **Most work is SCHEDULED, not flagged.** Flags are the edge case (user / manager / incident).
   Named scheduled work: every-3rd-night exterior wash · sensor calibration · battery health check ·
   tire inspection · brake inspection · misc light maintenance · software updates.
5. **Every service has a PLACE and a DURATION** — a preset bay/stall type and a timing requirement.
   (Founder estimates: sensor calibration ~20 min, brake inspection ~30–40 min. The *logic* is the
   point, not the numbers.)
6. **Triggered by WEAR or CALENDAR** — either way it is programmed ahead of time and scheduled for
   reservation as part of the normal arrival workflow.
7. ⭐ **Cadences span orders of magnitude.** A vehicle charges several times a day; a tire inspection
   is quarterly or semi-annual. ⇒ **MOST ARRIVALS ARE CHARGE-ONLY.** Service is rare per-arrival yet
   certain over a longer horizon. **Any design that evaluates every service on every arrival is
   wrong.**
8. *"Every scenario needs to be explicitly written out and built into OTTO-Q. This is the crux of
   our intelligent software."*
9. **NVIDIA sits ON TOP**, sorting and managing this data in real time for optimal orchestration.
10. **The loop:** twin GENERATES vehicle variables (randomly) → OTTO-Q SORTS/OPTIMISES → OTTO-Q
    sends orchestration directions BACK → twin PLAYS THEM OUT.
11. *"You may need to spend more time building out the variable needs of the vehicles themselves in
    the twin world model."*

**The load-bearing prerequisite:** the **arrival forecast**. If OTTO-Q only sees a vehicle minutes
out, a 40-minute brake inspection cannot be pre-booked and the doctrine cannot hold.

⚠️ **Completion must write back.** If finishing a service does not advance last-done, every vehicle
drifts permanently overdue and cadence becomes cosmetic.

---

## D5 — Needs are a probability draw, not accumulated wear
`memory/project_needs_probability_draw_doctrine.md` · `memory/project_needs_draw_already_exists.md`

Needs are a **seed-deterministic draw at run start**, roughly 1 in 5 to 1 in 10 per need type.
**Wear modelling is an explicit non-requirement.**

> 🚨 **The draw ALREADY EXISTS and is already in band.** `ottoq_run_boot_draw` →
> `ottoq_seed_vehicle_need_profiles` draws ~24 per-vehicle variables including tire tread, brake
> wear, and sensor health. **DO NOT BUILD A NEW ONE.** Claude asserted it didn't exist and was
> wrong.

---

## D6 — The full-service visit is atomic
`memory/project_full_service_visit_doctrine.md`

**Every visit is atomic: ~100% charge plus ALL services before redeploy.** A vehicle does not leave
half-serviced to come back later.

This has a measurable cost — it keeps more cars mid-service at any instant, which is exactly why
final-frame instantaneous metrics like `vehicles_turned_around` structurally penalise OTTO-Q and
**must not be quoted.**

---

## D7 — Vehicle-first: never hold a vehicle back for energy
`memory/project_vehicle_first_doctrine.md`

**Vehicles and chargers are never held back.** Energy shaving happens **only** via forecast + BESS +
scheduling — never by denying a vehicle a plug.

**A doctrine violation caught in the act:** `trimmed_by_cap` was silently deleting **100% of optimal
cuOpt solutions** — 21 invocations solved to `status:"Optimal"` and produced zero proposals because
an energy cap discarded the answer *after* the solve. Fixed by making the cap **advisory**:
`would_trim_by_cap` is still measured and logged, but no vehicle is denied a plug for energy.

> ⭐ **The general rule this teaches: a real ceiling must SHAPE the solution as an LP constraint,
> never post-hoc delete it.**

---

## D8 — Temp staging and perimeter staging are different PURPOSES, not overflow tiers
`memory/project_perimeter_hold_doctrine.md`

| Class | `stalls.staging_role` | Count | Purpose |
|---|---|---|---|
| **Temp spots** | `'temp'` | 24, interior rows | Short, transient, mid-workflow: quick inspection **after** all services are rendered; quick reassignment on disruption inside the original workflow (e.g. charger faults mid-charge). |
| **Perimeter spots** | `'long'` | 176, lot-edge ring | Longer-term: longer-term inspection; **overnight hold** — after the normal overnight return and all services, the vehicle is staged in **its specific perimeter space** for a few hours until morning dispatch. |

**The defect (inverse of doctrine):** every staging pick sorts
`ORDER BY (s.staging_role = 'temp') DESC` — temp is **always** preferred, spilling to perimeter.
Appears in `ottoq_decide_tick` (×2), `ottoq_book_appointment`, `ottoq_sim_prearrival_contracts`. So
an overnight hold takes an interior temp spot whenever one is free, consuming the 24-spot
quick-turnaround buffer. **The sort encodes capacity overflow; the doctrine wants purpose.**

`stalls.zone` is **not** the lever — it is read by exactly one function
(`ottoq_twin_depot_layout`, a renderer passthrough) and nothing branches on it.

---

## D9 — No re-route inside the depot walls without approval
`memory/project_indepot_reassignment_gate.md`

> *"A vehicle can't switch workflow queuing once it is inside the depot walls without
> technician/supervisor approval."*

Physical safety and predictability: a vehicle already moving or parked inside a constrained yard
must never receive a surprise re-route from an optimizer. Humans own in-yard exceptions.

**The gate:** `ottoq_indepot_reassignment_guard(vehicle, run, reason, payload)`
- Outside walls (`deployed`/`en_route_to_depot`/`en_route_to_deployment`/`offline`) → **allowed**.
- Inside + `reason='resource_fault'` → **allowed** (broken charger or bay is a necessity — the
  exception triad).
- Inside + discretionary → **DENIED**, queues an `ottoq_ops_approvals` row of type
  `indepot_reassign` (30-min expiry, no duplicate pendings).

**Standing rule:** any new code path that changes an in-depot vehicle's stall, queue, or workflow
**MUST call the guard first.** Never bypass.

**Amendment (2026-07-28) — auto-advance while there are no technicians:** stages auto-advance once
their designated timing completes, but are **marked complete** exactly as a technician would on
OTTO-PULSE. **This is not a bypass, it is an auto-approver** — the gate is still called and an
approval row is still written; only `decided_by` differs. `ottoq_stage_advance_approval(...)` reads
knob `tech_approvals_required` (default 0 = auto). Swapping in real humans later is a config change,
not a code change.

> ⚠️ **Audit finding: this doctrine was never actually enforced.** Exactly one function called the
> guard, passed the argument that makes it auto-approve, and **never read the result.** The
> highest-volume in-depot move (staging → charging in `ottoq_decide_tick`, **61,378 enacted**)
> bypassed it entirely. Wiring every reassignment path to the guard is **outstanding work.**

---

## D10 — cuOpt's three eligibility zones
`memory/project_cuopt_eligibility_doctrine.md`

| Zone | Vehicle state | cuOpt authority |
|---|---|---|
| **A — OPEN** | en route / returning, **has not yet approached** | **Full re-optimization.** Reservations here are provisional. This is where the optimizer earns its keep. |
| **B — FROZEN** | **approached OR entered** the depot | **No override.** The itinerary IS the vehicle's committed arrival workflow. Leave it alone. |
| **C — RE-OPENED** | frozen, then hit by **malfunction / congestion / flag** | cuOpt re-engages for that vehicle. The **exception** path, not a routine tier. |

⚠️ **The boundary is "APPROACHED", not "inside the gate."** Claude proposed freezing at the depot
walls; Chase tightened it to **approach**. Do not silently substitute `arrived_at_gate`.

Zone C is the only path that may touch an in-depot vehicle, so it **must route through D9's gate**.
Zone A needs no approval because nothing is committed yet.

**Architecture mandate:**
> *"cuOpt should be built INTO OTTO-Q, so OTTO-Q working literally is all feature in one. NVIDIA and
> our own proprietary intelligence."*

Target shape: **one** assignment step inside OTTO-Q that uses cuOpt as its engine, with the local
greedy as its **fallback, not its rival.** Two competing paths racing each other is exactly why
cuOpt never won.

---

## D11 — Energy: OTTO-Q signals, it never actuates
`memory/project_energy_orchestration_seam.md`

OTTO-Q **reads demand cycles** (real-time as load builds, and historical patterns), analyses, and
**sends control signals to an external energy system** — e.g. Schneider EcoStruxure: *"pull this
much now," "switch off," "discharge the battery," "shift to off-peak."*

**OTTO-Q does NOT move energy itself.** It is the brain that commands the energy hardware.

The twin holds the energy physics *and* runs a middleware "energy system" (an EcoStruxure stand-in)
that receives OTTO-Q's commands and executes them on simulated hardware. Eventually that middleware
is replaced by real EMS hardware. **Same pattern as twin-holds-telemetry / OTTO-Q-decides.**

Seam: `ottoq_energy_commands` (the signal log) · `ottoq_sim_energy_controller` (the mock EMS) ·
`ottoq_active_charge_cap_kw` · `ottoq_energy_orchestrate` (the brain).

---

## D12 — Runtime cadence
`memory/project_runtime_cadence_doctrine.md` — covered in detail in `02_OTTOQ_ARCHITECTURE.md` §2

- The **tick is not OTTO-Q's decision rate.**
- **Nothing runs until START.** PAUSE freezes. **STOP wipes the twin but saves the log.**
- OTTO-Q keeps **real-time latency** while the twin runs at 2–3×.
- **The twin never freezes for a decision.** Temporary staging is the designed pressure-relief valve.

---

## D13 — Positioning: sell the ecosystem, never one number
`memory/project_ecosystem_positioning.md` · `memory/reference_ottoq_real_edge.md`

The single *provable* edge under narrow adversarial audit is the **safety guarantee**. That is the
credibility floor. But **do not sell on one layer** — sell the **integrated ecosystem**, of which
the safety shield is the trust anchor. Both are true simultaneously.

Any single facet, isolated and adversarially audited, can look modest. Narrow optimizer
sub-problems tie good simple logic. The value compounds across the combined system.

---

## D14 — Rebuild, not benchmark
`memory/feedback_rebuild_not_benchmark.md`

During a rebuild the bar is **"it works and moves correctly"**, NOT "it beats a baseline."
Numbers are evidence, never a score. If a correctness fix makes throughput fall because the old
number was computed on a lot that cannot physically exist, that is a **realism improvement**, and
saying so is the honest report.

---

## D15 — Demo loop ops
`memory/project_demo_loop_ops.md`

Bounded demo runs of roughly **one sim-day**. **Never 30-day marathons.** Chase grabs snapshots,
uses 2×/3× to capture hours in a fraction of the time, and jumps ahead or resets to have OTTO-Q
reassess against the twin's new variable spin-up.

⚠️ Certification runs are **not free** — one session added ~3 GB to `ottoq_events` by repeatedly
running full-length certifications into a container that was already nearly full. **Budget runs
against remaining headroom.**
