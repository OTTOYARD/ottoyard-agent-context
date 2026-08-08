# 11 — The backlog

**How to use this file.** Lane A is what Chase named as your starting focus. Everything after it is
the full opportunity surface, ranked by value and grouped so you can pick coherent work. **Once you
have read the package, you decide where to dive in** — Chase's words: *"Ultimately, once it reads,
it can decide exactly where it needs to dive into first or where it can be most optimally used."*

**Before starting anything here:** search for it first. This system is more built than it looks, and
"it already exists, half-wired" is the correct prior. Two of the biggest planned builds in this
project's history were cancelled on discovering the thing already existed.

**Each item is written as: the outcome, why it matters, where to start, and how you will know it
worked.**

---

# LANE A — the named starting focus

> *"A huge focus should be the twin simulator and visual layer and vehicle motion… vehicle cards and
> variable selection at the start of new runs. Also the complete ecosystem tie-in between the twin,
> Orchestra and Pulse so that they all depict the same information and OTTO-Q being the orchestrator
> and sorter of all of them."* — Chase, 2026-08-08

## A1 · The ecosystem tie-in: one run, one clock, three lenses
**Value: highest in this lane.** It is what turns three demos into one product.

**Outcome:** the twin cockpit, OTTO-PULSE, and OrchestrAV all show the same live run, the same sim
clock, the same tick, and the same per-vehicle facts — matching to the digit, because they read the
same RPC. When no run is live, all three say so identically and honestly.

**Start with:**
1. A shared run-context read. `ottoq_twin_run_context(run)` already exists — make all three consume
   it for run id, status, sim clock, tick, scenario, seed, speed.
2. Extend the `ottoq_depot_cards(depot_id, fleet_operator_id)` pattern. **One call feeds a whole
   cockpit; the operator filter is the only difference between Pulse and OrchestrAV.** Do not invent
   a parallel read surface.
3. Kill every mock fallback (P1-17). An honest "no live run" beats a plausible lie.
4. Remove or repoint OrchestrAV's dead `ycsis` realtime subscription (P2-8).

**Done when:** you can start a run in the twin cockpit, open all three surfaces, and every shared
number matches; and stopping the run makes all three say "no run" at the same moment.

## A2 · Surface the "why" in all three cockpits
**Outcome:** every surface can answer *why this vehicle is in this space, what it displaced, and
what happens next.* This is the actual differentiator and it is largely unsurfaced.

The data exists: `ottoq_decisions` (with `resolved_action_context`, source, rationale),
`ottoq_rule_evaluations`, `ottoq_itinerary_legs`, `ottoq_stall_bookings`.

**Blocked partly by P1-13** — `leg_id` is on only 3.7% of enacted bookings, so for 96.3% the calendar
cannot answer "why." **Fixing that is a prerequisite and is itself high value.**

## A3 · The forward calendar as a real timeline
**Outcome:** `ottoq_stall_bookings` rendered as a Gantt-style forward timeline in Pulse ("what is
claimed, when, by whom, and why") and as "your car's plan" in OrchestrAV.

**Why it matters:** the forward calendar **is** the orchestration (D3). Right now it is invisible,
which means the single most defensible idea in the product is not shown to anyone.

## A4 · Vehicle cards — finish and merge
**Outcome:** per-vehicle OTTO-Q work-order cards in both cockpits.
Card at rest = owner badge (Pulse only) + current step + progress bar + next step. Click to expand =
the full sequence, done ✓ / current ▶ / upcoming, with times. **Streamlined, not bulky.**

Backend RPCs are **live and verified**. Branches exist and may be unmerged:
`ottoyard-field-ops`: `claude/ottoq-vehicle-cards` (base `claude/prime-ops-panel`) ·
`ottoyard-OTTO-Q`: `claude/ottoq-sequence-cards`.

**Check whether they merged before rebuilding anything.**

Preserve the honesty rules: prune `skipped` legs, scope to the current visit, **omit `deviation_s`**
(corrupt baseline), atoms key is `'svc'`.

## A5 · Start-of-run variable selection
**Outcome:** the operator console offers a scenario preset **and** an advanced panel exposing the
real variability knobs — fleet size, arrival rate, needs density, charger fault rate, staffing,
weather severity, grid stress — writes them into the run's variability profile, and the boot
manifest (`ottoq_twin_boot_manifest`) reports exactly what was drawn.

**Two blockers to resolve first:**
1. **P0-4 — seed 424242 is pinned.** Until that is found, runs are identical no matter what the UI
   offers.
2. **The deck override JSON does not cover the `_rates` knobs** (arrival/dispatch, ETA delay, DR
   likelihood, charger fault, incident, staffing). Work out how a scenario seeds `_rates` before
   promising a control that sets them.

**Done when:** two runs with different chosen variables produce demonstrably different worlds, and
each is exactly reproducible from its seed.

## A6 · Motion realism — the specific open items
1. **Fix the 3D car scale** (P1-6). `BoxGeometry(2.2, 0.85, 4.9)` is metres in unit-space — a toy
   car. Uniform group scale ≈ 2.0 is the cheap fix. **Highest visual-quality-per-line-of-code in the
   repo.**
2. **Resolve the east-avenue overlap** (P1-5). Layout decision, not a motion fix. Mirror the west
   avenue's 4.1u clearance.
3. **Add stall-vs-lane clearance to the geometry guard** (P1-7 hole 4). The guard would have blessed
   the east-avenue conflict forever.
4. **Reconcile the three car lengths** (2D 10.2u, 3D 4.9m, physics `CAR_LENGTH=9`). ⚠️ Needs a
   certified motion pass, not a drive-by edit.
5. **Drive the residual overlap down.** The fixture measures 221 body-overlap samples and 130
   wedged-car samples — improved from 543/297, **but not zero. Do not claim zero.**

⚠️ **Do not re-propose:** routing cars onto the parking access aisle before departing (measured
305 → 728 overlaps) · correcting the parked heading (305 → 378) · enabling `separationSteer`
(radius 5.5 > 4.8u lane separation — it would *manufacture* the head-on swerve).
⚠️ **Do not re-litigate the lane network.** It is deliberately structure-aware and the flow is
founder-locked.

## A7 · Tick cadence for smooth motion
**Outcome:** motion that looks continuous rather than stepped.

**The measurement that matters:** ticks fire **8–31 real seconds apart** while tick compute is
**~1.15 s**. **Smoothness is bounded by the pg_cron metronome's firing cadence, not by engine
speed.** The 1:1 clock ratio is already exact and verified — this is a cadence problem, not a ratio
problem. Do not re-derive that.

Options worth exploring: a finer metronome, client-side interpolation over a longer horizon, or
having the renderer extrapolate between server ticks using the itinerary's planned timings.

## A8 · Push OTTO-Q assignments into the photoreal twin
**Outcome:** the RTX view shows vehicles actually moving on OTTO-Q's decisions, not a static depot.

The WebRTC data channel is already active. This is the "spectacle plus substance" payoff of the
whole Tier-B investment.

---

# LANE B — realism and the world model

## B1 · Cross-variable correlations ⭐
**The single biggest realism gap.** Only **one** fitted correlation exists across seven ingested
corpora. Independent random knobs are not realistic stress. **Reality co-moves:** a heat wave means
AC load up *and* charge rate down *and* grid price up *and* arrival pattern shifted.

Correlated variability is what turns the twin from a random-number generator into a **statistical
clone** — and it is what makes any OTTO-Q-beats-baseline claim credible.

**Start:** `ottoq_calibration_datasets` / `_distributions` / `_profiles`, and
`twin_ingest/ingest_calibration.py` in the workspace repo. Also worth deepening the corpora
themselves (full ACN 130k, full-year NOAA via CDO, fuller NYC TLC and CA DMV).

## B2 · Open the wash night gate (P1-1)
34 of 116 vehicles due a wash, **0 atoms emitted**, all three wash bays idle by construction.
⚠️ Opening it puts ~35 jobs into 3 bays that are shared with detail, and there are **zero
`detail_bay` stalls.** Size the lane to ~75% of bay-minutes and add a boot-time assertion.

## B3 · Find and unpin seed 424242 (P0-4)
Prerequisite for A5 and for judging any variability work.

## B4 · Make the wear model stateful (P1-14)
Tire tread, brake wear, SoH, sensor health, software version and odometer are drawn once and frozen.
⚠️ **Confirm scope with Chase first** — it borders on the explicitly-non-required wear model.
The likely correct shape: completion writes back (which D4 requires anyway) and elapsed-time /
mileage advances the counters, without building a physics wear simulation.

## B5 · Make `seed_phase_max` real or delete it (P1-16)
A column that appears to tune the draw and does nothing.

## B6 · Build the missing scenario decks
`traffic_surge` (high dispatch + ETA delay) · `mass_incident_response` (incident spike) ·
`overnight_wave_prestage` (day/night ops).
⚠️ **Open question first:** how does a scenario seed the `_rates` knobs? They are not in the deck
override JSON.

## B7 · The city-map arrival visualisation
Founder direction, deliberately **deferred behind motion and realism**. Pick a real metro (Nashville);
at run spin-up, place deployed vehicles at randomised positions around the city as moving dots on
roads. Each vehicle's arrival ETA = realistic travel time from its position. Traffic and weather
delays flex the ETA dynamically (the weather cascade already exists — precipitation → ETA ×2.4), so
OTTO-Q must react agilely. Show vehicles driving in from the road **into** the depot.

⚠️ **All vehicle randomness lives in the twin variability ecosystem, never in the renderer.**
⚠️ **Mapbox is banned** — a leaked token cost $1–2k. **MapLibre with free tiles only.**

This also creates the real ETA that P0-5 needs. **The two are the same problem seen from two ends.**

---

# LANE C — the forward-scheduling core (the crux of the product)

## C1 · Give the needs card an arrival side ⭐
`next_arrival_at`, `minutes_to_arrival`, `arrival_confidence`. Join `ottoq_approach_band`.
**Today no function in any schema joins those two objects.** This unlocks everything in Lane D.

## C2 · Stop the ETA overwriting the dispatch plan
Write the refreshed ETA to its own column so plan-vs-actual stops being **circular by construction**
and a training signal exists at all.

## C3 · Give `ottoq_book_appointment` a real ETA
It is a hardcoded 30-minute constant. **True forward visibility is 30 flat minutes**, and only after
the vehicle has already decided to come home. D4 requires a horizon long enough to pre-book a
40-minute brake inspection.

## C4 · Teach `ottoq_book_appointment` to reserve bays
Its stall search only queries `('dcfc','l2')` then `'staging'` — the strings `wash_bay` and
`service_bay` **do not appear in it.** Measured: **0 of 110 needs had a bay reserved pre-arrival**
against 43 bay-requiring atoms.

⚠️ **Migration 0011 already did a version of this** (`ottoq_reserve_inbound_bays`,
`ottoq_book_workflow_legs`, `ottoq_svc_to_stall_type`; 7/7 booked pre-arrival, median lead 30.8
sim-min). Branch `fwd-bay-reservation-0011`, applied and pushed, **not merged.** Check its state
before rebuilding.
⚠️ **Anchor on the LIVE ETA, not `planned_return_at`** — median divergence −114.9 min.
⚠️ **`service_cadence_policy.lane` is the live requirement column**, not `service_definitions`
(which shares 1 of 15 atom names and requires a `detail_bay` that does not exist).

## C5 · Make completion write back
If finishing a service does not advance last-done, **every vehicle drifts permanently overdue and
cadence is cosmetic.** D4 depends on this absolutely.

## C6 · Fix the staging sort (P1-3)
Purpose, not overflow. `ORDER BY (staging_role='temp') DESC` in four call sites.

## C7 · Bind the overnight hold to a specific perimeter space
The overnight-hold leg carries **no stall binding**, and nothing gives a vehicle *its* space. Chase's
hard constraint is **one dedicated spot per vehicle** for safety-recall scenarios. Today that is
aspirational in the data.

## C8 · Fix bay no-show (P1-12)
Prime suspect: the departure-readiness gate pins vehicles in staging while their bay booking keeps
ticking — **the gate manufacturing its own no-shows.**
Also: `ottoq_bind_unbooked_bay_occupants` gives up `no_free_space` 34×/run. **Reality must outrank a
plan** — displace the stale claim, book the car physically present, emit an auditable conflict row.

## C9 · Carry `leg_id` onto every booking (P1-13)
3.7% today. The stall matches; **the reason does not travel.** Prerequisite for A2.

## C10 · Garbage-collect reservations, and fix the three blind readers (P1-9)
Three functions read `reserved_by IS NULL` without the expiry guard and under-count free capacity —
including `ottoq_report_charger_fault`, which sees only 10 of 17 free DCFC and therefore **starves
fault recovery.** The executor's vacate also fails to clear `reserved_by`, leaking capacity that
reads as phantom scarcity.

---

# LANE D — the intelligence layer

## D1 · Fix the cuOpt supply starvation (P1-4) ⭐
`free_stalls_in = 0` on 48 of 83 edge calls while ~27 chargers were physically free.
**Availability must consider only `state IN ('held','active','done')`.** One predicate.

## D2 · Stop discarding the AI's choice (P0-3) ⭐⭐
`ottoq_sim_auto_dispatch_tick` re-picks by `soc DESC, seeded_random`. **Until this is fixed, no AI
contribution is measurable at all.** Everything else in this lane is unfalsifiable without it.

## D3 · Re-point cuOpt at the regime where it can actually win
Per-tick stall assignment is a polynomial LAP — greedy is provably near-optimal and cuOpt can only
tie. **cuOpt's real home is the compressed overnight wave under charger scarcity**: ~100 drained
vehicles time-sharing scarce DCFC/L2/bays/staff under a cumulative energy cap with hard
morning-deploy due dates.

The deterministic EDF baseline is already shipped (`ottoq_plan_overnight_wave`, verified: 94 planned
/ 0 stranded / 68 L2 + 26 DCFC). **That is the honest floor cuOpt must beat.**
⚠️ **Benchmark OR-Tools CP-SAT against cuOpt.** If CP-SAT wins on the bench, ship CP-SAT and keep
cuOpt for the sub-structure — **unless NVIDIA-in-the-loop is a positioning requirement. That is an
open founder question.**

## D4 · Fold cuOpt INTO the one assignment step
Founder mandate: *"cuOpt should be built INTO OTTO-Q… all feature in one."* One assignment step that
uses cuOpt as its engine with greedy as its **fallback, not its rival.** Removes the split-tick's
cadence cost and the proposal-vs-greedy race.

## D5 · Implement the Zone A/B/C eligibility rule (D10 doctrine)
The approach band exists but has two defects: **Zone C over-fires catastrophically** — 539
`reopened:hardware_malfunction` vehicle-ticks vs 10 `reopened:flag` (normal charger occupancy read
as a malfunction), which inverts the doctrine; and the ETA is **quantized to the 30-minute tick
grid**, so a 10-minute freeze boundary is crossed only 3 times in 1,850 observations.
**The fix for the second is the clock (live mode), not the band.**

## D6 · Close the Nemotron training-data loop
The approval co-pilot writes its recommendation but **does not store the human's accept/override
verdict** — no training signal exists. Build the
`(context → recommendation → verdict → outcome-N-ticks-later)` graded tuple log: few-shot memory
now, SFT set later. Needs a few thousand **real** (not seeded-dice) tuples plus an offline
replay-vs-historical-verdicts harness before any model touches a live run.

## D7 · Retire the continuous dial-railing conductor
Replace with **coarse once-per-regime discrete posture selection**
(`daytime_churn | pre_wave_prestage | overnight_wave | morning_commit` → a vetted preset). The
current conductor pins rails while narrating the opposite — a diligence liability.

## D8 · Wire the co-pilot into the UI
Trigger it automatically on approval creation, and surface the recommendation in the Pulse and
OrchestrAV approvals queues. **Standing rule: the co-pilot advises; the guard and the human
enforce.**

## D9 · Make the exception path intelligent
Today's exception path is **pure seeded-random dice.** Nemotron triage over real long-tail
exceptions is a genuine opportunity and exactly what Tier 3 is for.

## D10 · Then, and only then: learn the forecast
Predict arrival time and true dwell/service duration from history; feed cuOpt and Nemotron the
reconciled picture. **Blocked on C1–C5.** You cannot train on a constant.

---

# LANE E — safety, audit, and the OEM bar

## E1 · Route the deploy transition through the shield (P0-2) ⭐
The single highest-value safety work available. Turns an honest-but-modest claim into a real one.

## E2 · Enforce rule 1 with triggers, not an inversion (P1-10) ⭐
`ottoq_events` already blocks UPDATE/DELETE with a BEFORE trigger regardless of grants. **The same
pattern on `vehicles.current_stall_id` and `stalls.current_vehicle_id` makes rule 1 real with no
schema DDL and none of the inversion's eight blockers.**
**Do not attempt the naive inversion.** Do the safe prep list in
`memory/project_ottoq_twin_boundary.md` first.

## E3 · Wire every reassignment path to the guard (P1-2)
Including the 61,378-enacted staging→charging path. And decide what `severity='critical'` should do
— it currently auto-allows, and **21 of 58 evictions still cut live work.**

## E4 · Fix the backwards uniqueness index
`idx_stalls_one_vehicle_per_stall` is `UNIQUE ON stalls(current_vehicle_id)` — it enforces **one
stall per vehicle**, the opposite of its name, and **structurally cannot** detect two vehicles
claiming one stall.

## E5 · Populate `ottoq_fleet_operator_slas` (P1-8)
Make the 80% deploy floor a real per-operator SLA instead of an undefended hardcoded fallback in
three places.

## E6 · Complete the geometry guard (P1-7)
Fix the fence check that silently skips depot 2; add DB-side aisle checking; add stall-vs-lane
clearance.

## E7 · Real tenancy and RLS
The security deploy gate. **Founder decision: deferred until post-funding.** Do not spend autonomy
budget here unless asked — but never weaken what exists, and always flag anything new that will need
hardening.

## E8 · Prove the audit trail end to end
`ottoq_generate_audit_bundle`, `ottoq_sign_bundle`, `ottoq_verify_audit_bundle`,
`ottoq_verify_event_signature`, `ottoq_replay_window`, `ottoq_causation_chain` all exist. **The #1
durable moat is "a signed, replayable audit trail."** Demonstrating it working end to end — pick a
decision, replay it, verify the signature, show the causation chain — is a high-value artefact for
diligence and probably a day's work.

---

# LANE F — energy

## F1 · Re-run the energy-$ certification
Every prior energy number was scored against a baseline with **phantom charger capacity** and a
**battery charging into its own peak**. Both are fixed. **The numbers are stale and must be
re-earned.**

## F2 · Day/night-aware energy caps
`energy_orchestrate` caps daytime at `service_max × 0.5`, which may be too aggressive against
daytime fast turnaround. Loose daytime, tight overnight.

## F3 · Fold the energy term into the wave scheduler's objective
Do not run two energy controllers. Coordinate via the shared cap with the MPC.

## F4 · Move the bridge token and AWS URL to Supabase secrets (P2-6)

## F5 · Honest MPC benchmarking
The 38.3% shave is against the **naive** heuristic. A same-day refutation showed **a reserve-aware
reactive controller matches the LP to the penny.** Establish the well-tuned reactive baseline and
report against **that**.

---

# LANE G — the CapEx / charger-ratio proof

## G1 · The compressed overnight wave experiment ⭐
**Chase's thesis:** the flagship has 45 charge stalls for 116 vehicles (2.6:1). Cutting to
**30 (10 DCFC + 20 L2)** = 3.9:1 saves 15 L2 units of CapEx. *"When we're trying to optimize for as
few chargers to as many vehicles as possible, that's when orchestration comes into play."*

**The first probe gave a refining result, not a confirmation:** cutting 45→30 **narrowed** OTTO-Q's
edge (1.33× → 1.08×) because the cert scenario spreads returns over 6–10 h — 30 chargers are not the
*time*-binding constraint, so all policies converge. **`throughput_per_hr` is the wrong metric and
the spread shift is the wrong scenario.**

**The right experiment:** 100–120 AVs returning 22:00–03:00 in a tight window, double-constrained by
few chargers **and** little time. The differentiators are:
(a) % fleet fully charged and serviced by ~05:00 · (b) 0 stranded cars · (c) 0 under-charged AM
deploys · (d) energy peak with BESS.

**Then sweep 45 → 30 → 24 → 20 chargers and report the count at which each policy FIRST strands a
vehicle.** The gap is the CapEx saved — an audit-proof *feasibility* measure, not a snapshot metric.

**A related certified result worth knowing:** under the 100%-charge doctrine, **arrival timing beats
hardware.** One hour of proactive early recall converts "impossible even at 30 chargers" into
"zero-missed at 22." Mechanism: a full charge from ~28% takes ~22 min on DCFC but **~5 h on L2** — an
early-recalled vehicle can use a cheap slow charger all night; a late one *requires* a fast charger.
**Wave shaping converts DCFC demand into L2 demand → a smaller, cheaper plant.**

---

# LANE H — hygiene and infrastructure

| # | Item | Notes |
|---|---|---|
| H1 | **Inventory and clean ~300 scratch tables** | 461 tables in `public`. Some are the only surviving evidence for a published cert — **inventory before deleting, get sign-off.** |
| H2 | **Queued replacement for `trg_ottoq_auto_incident_report`** | Currently disabled; it did a synchronous scan of a huge table inside `decide_tick`. |
| H3 | **Fix the purge's permanent orphaning** | It can only reach children of runs that still exist. 252 + 79 rows stranded forever. |
| H4 | **Retire or repoint `ottoq-progress`** | Orphaned on dead `vehicle_schedules`/`schedule_tasks`. |
| H5 | **Wire or retire `ottoq_recommendations`** | 83,812 rows, **zero** ever executed. |
| H6 | **Elastic IP for the Isaac box** | Removes the `localStorage` IP-override dance. |
| H7 | **Merge or close the stale branches** | Several carry real work: `fwd-bay-reservation-0011`, `fwd2-inspection-condition-resets`, `claude/ottoq-vehicle-cards`, `claude/ottoq-sequence-cards`, `claude/prime-ops-panel`, `claude/blackbox-panel`, `unify-depot-layout`, `layout-unify-importer`. **Inventory these first — several backlog items may already be done on a branch.** |
| H8 | **Re-verify `OTTOQ-TWIN-BOUNDARY.md` is on `main`** | It was one branch deletion from gone. |
| H9 | **Full UI audit against OTTOYARD branding** | Standing founder want. Tokens are in `04_ORCHESTRA_AND_PULSE.md` §5. Both apps still use generic shadcn themes. **"No lazy UI."** |
| H10 | **A real MQTT broker** | The comms layer is OEM-spec and proven round-trip but emulated in-database. |
| H11 | **Repo naming** | `ottoyard-OTTO-Q` is OrchestrAV, not the brain. Every reviewer trips on this. A rename is disruptive but the confusion is real and recurring. **Chase's call.** |

---

# The three-question filter before you start anything

1. **Does it already exist?** Search the RPC list, the migrations, and the branches. The prior is
   "yes, half-wired."
2. **Would it survive an OEM engineer reading it adversarially?** If it only works because this is a
   simulation, it breaks the swap test and it is worth nothing.
3. **Can I prove it worked?** If you cannot name the query, the row count, or the screenshot that
   will demonstrate it, you do not yet have a deliverable — you have an intention.
