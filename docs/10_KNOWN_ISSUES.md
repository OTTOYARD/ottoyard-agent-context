# 10 — Known issues and gap register

**Status as of 2026-08-08.** Every item here was measured, not suspected. Each carries what is
known, what is not, and where to look.

> **Each of these is point-in-time. Verify before you build on it.** Several entries in this project's
> history were "established facts" that turned out to be false and cost another session a wasted
> starting hypothesis.

---

> ## 🚨 CORRECTION NOTICE — 2026-08-08, second pass
>
> **The first draft of this register was written against stale local clones** (`otto-q-core` was
> **77 commits behind** `origin/main`). Re-verified against `origin/main` and the live database.
> **Migrations now run through 0022, not 0010**, and several items below are RESOLVED rather than
> open. Corrections are marked inline. **The lesson generalises: always `git fetch` and compare
> against `origin/main`.** See `docs/17_LOVABLE_AND_SYNC.md` §7.

## 🔴 P0 — blocks something important

### ✅ P0-1 · ~~Migration 0010 must not be applied as written~~ — **RESOLVED. It was fixed and applied.**
`memory/project_depot_layout_unification.md`

**APPLIED 2026-08-06, ledger version `20260806223619`, name `unify_depot_layout`.** The preflight halt
did its job: the file was rewritten to **re-home rather than retire**, exactly as required.

**The renderer's depot is now the database's depot.** Per depot the layout goes **150 → 160 stalls**:
staging 100 → 115, **l2 35 → 30**, dcfc 10, wash_bay 3, service_bay 2 — applied to **both** depots.
Every stall was re-plotted onto the real **452.13 × 313.98 ft parcel (3.26 acres)**, replacing the
360 × 220 ft / 1.82 acre frame the database had recorded. `ottoq_site_structures` and the depots'
`site_*` fields moved with it. **19 codes leave the layout per depot, of which 14 staging rows are
RE-HOMED via UPDATE**, not deleted. **Zero routines created, replaced, or dropped.**

⚠️ **Note the L2 count fell 35 → 30.** That is the correction of the over-programmed lot (the 5
appended `L2-STALL-21..25` rows that ran past the end of their own canopy). **Any capacity, charger-
ratio, or CapEx number computed before 2026-08-06 was computed against 35 L2 stalls that could not
physically exist.**

**Still true and still worth knowing:** `ottoq_plan_overnight_wave` counts `stall_type='l2'` with
**no status filter**, so "retire in place" via `status='closed'` would not work for L2. Evidence
preserved in `preimport_*_20260806`.

### P0-2 · The deploy transition does not pass through the safety shield
`memory/reference_ottoq_real_edge.md`

**No function that transitions a vehicle to `deployed` calls the rule engine.**
`ottoq_shield_and_log` defaults to `p_shadow = true` and has **zero deploy-path callers.** Safety
holds via an inline `WHERE current_soc >= 80` precondition plus an `ottoq_deploy_log` trigger.

The record is real (0 deploys below the stranding floor across 1,012+ dispatches) but the **naive
baseline shares the same inline filter**, so "0-unsafe vs baseline" proves shared code, not a shield
edge. **Routing the deploy transition through the shield is the single highest-value safety work
available.**

### ✅ P0-3 · ~~The AI assignment is discarded before it reaches the world~~ — **RESOLVED by migration 0019**
`memory/reference_ottoq_real_edge.md`

**FIXED 2026-08-09, verified by Hermes Agent.** Migration 0019 (`rider_flag_holds_the_vehicle`)
added `ottoq_brain_deploy_rank` as the **primary** ORDER BY in `ottoq_plan_dispatch_tick`:
```sql
ORDER BY ottoq_brain_deploy_rank(p_sim_run_id, v.id) ASC NULLS LAST,
         v.current_soc DESC, ottoq_sim_seeded_random(...)
```
`ottoq_brain_deploy_rank` reads `ottoq_decisions` where `action_context='redeployment'` and
`outcome_status='enacted'`, returning the MIN tick_seq — so the vehicle OTTO-Q decided to deploy
**first** is dispatched **first**. SOC and random seed are only tiebreakers. Confirmed in the
live database: 477 redeployment decisions ranked across 64 vehicles in the latest run.

### P0-4 · Seed 424242 is pinned — every run is identical
`memory/project_needs_draw_already_exists.md`

Every recent run in `ottoq_sim_runs` uses seed `424242`, so *"completely new probabilities each
run"* is **false today, regardless of the draw.** **The pin has not been located.** Find it before
judging any variability, needs-density, or forecast work.

### P0-5 · The arrival forecast has nothing to learn from
`memory/project_arrival_forecast_learning_doctrine.md` — full detail in `07_NVIDIA_AI_LAYER.md` §6

- No arrival side on the needs card: 74 columns, **not one** contains "arriv", "eta", or "return".
- `ottoq_return_eta_minutes` is a **hardcoded 30-minute constant** — no distance, no route, no
  traffic.
- The refreshed ETA **destructively overwrites the dispatch plan** (109 of 112 rows =
  `returning_started_at + exactly 30 min`), making plan-vs-actual **circular by construction**.
- The dispatch-time plan is useless anyway: 0 of 112 arrivals within 5 min; median absolute error
  **68.4 min**, p90 215.5, mean bias **+92.4 min late**.
- **No function in any schema joins `ottoq_vehicle_needs_card` to `ottoq_approach_band`.**
- ✅ ~~`ottoq_book_appointment` cannot reserve a bay~~ — **RESOLVED by migration 0011** (applied
  2026-08-06, `20260806231121`). It now calls `ottoq.ottoq_reserve_inbound_bays` **before** its
  charge-stall search, so a busy tick with no free charger no longer skips the bay hold.
  Reservations land in `ottoq_stall_bookings` with `source='return_signal_prearrival'`.
  🔴 **But those holds still do not BIND — see P1-18. That is now the live problem.**

**You cannot train a forecast on a constant.**

### P0-6 · The wash rotation was never seeded either
`otto-q-core` MIGRATION_LOG, migration 0018 (applied 2026-08-08)

A second, independent instance of P0-4, found while building the rider-flag recall. The founder's
rule is that cleaning variability is *"selected and determined at the start of any given simulation
run. Randomized each time."*

The every-third-night wash gate reads `config.wash_group`, and **all 216 active autonomous vehicles
already carry one**; the fallback is `abs(hashtextextended(vehicle_id::text, 77)) % 3`.
**Neither branch involves `random_seed`** ⇒ **the wash rotation was byte-identical in every run ever
executed.** Addressed in 0018 — **verify the current state rather than assuming either way.**

---

## 🟠 P1 — real defects with measured impact

### P1-1 · The wash night gate discards due washes
`memory/project_needs_draw_already_exists.md`

`twin.ottoq_sim_generate_service_manifest`'s wash condition is
`(v_is_night AND wash_group = sim_day % 3) OR soil_index >= 0.75 OR cycles_since_wash >= 9`.

Measured on run `6f7518ef` (busy_day / 424242, 98 visits, fleet 116):
**34 of 116 vehicles were DUE a wash. 0 `exterior_wash` atoms were emitted.** The run spanned
09:08–19:18 local; **0 of 98 arrivals fell in the night window.** Both escape hatches were
unreachable: max `soil_index` **0.4417** against a 0.75 override; `cycles_since_wash` is drawn 2–5
against a backstop of 9. **All three wash bays idle by construction.**

⚠️ Opening the gate puts **~35 jobs into 3 bays**, and wash and detail share those same 3 bays —
there are **ZERO `detail_bay` stalls.** Size the lane to ~75% of bay-minutes and add a boot-time
assertion on projected vs available bay-minutes.

### P1-2 · The in-depot reassignment gate has never actually held
`memory/project_indepot_reassignment_gate.md`

Exactly one function called `ottoq_indepot_reassignment_guard`, passed the argument that makes it
auto-approve, and **never read the result.** The highest-volume in-depot move (staging → charging in
`ottoq_decide_tick`, **61,378 enacted**) bypassed it entirely. Only one approval row exists in the
system's entire history.

**Partially improved:** the guard now fires 54 times and 91 of 112 gate decisions (81.3%) protect
live work. **But `severity='critical'` still auto-allows: 21 of 58 evictions still cut live work
(19.8–113.4 sim-minutes of work destroyed).**

### P1-3 · Staging selection is backwards to doctrine
`memory/project_perimeter_hold_doctrine.md`

Every staging pick sorts `ORDER BY (s.staging_role = 'temp') DESC` — temp is **always** preferred.
Appears in `ottoq_decide_tick` (×2, ~lines 165/217), `ottoq_book_appointment:112`,
`ottoq_sim_prearrival_contracts:65`. An overnight hold therefore takes an interior temp spot,
consuming the 24-spot quick-turnaround buffer before spilling to the 176 perimeter spots where it
belongs. **The sort encodes capacity overflow; the doctrine wants purpose.**

### ✅ P1-4 · ~~cuOpt supply starvation~~ — **RESOLVED (2026-08-03 fix)**
`memory/project_orchestration_build_2026_08_01.md`

**FIXED 2026-08-03.** `ottoq_stall_free_between` now filters
`b.state IN ('held','active','done','interrupted')` — `released` and `superseded` are excluded
from availability. Verified 2026-08-09: 0 interrupted bookings in certified runs, so the
`interrupted` inclusion has zero practical impact. 

cuOpt share has since oscillated 42.4% → 36.4% → 18.0% raw / 28.8% like-for-like. **Conversion is
healthy (86.7–100%); supply is the bottleneck.** Suspect a newer path (`inspect_seam` /
`gate_intake_staging`) pre-empting cuOpt's candidates — the same sequencing family as the original
defect.

### ✅ P1-5 · ~~The east-avenue lane/stall overlap~~ — **RESOLVED (2026-08-09)**
`memory/project_twin_render_offmap_bug.md`

**FIXED 2026-08-09 by Hermes Agent.** E-column (`staging_east`, 25 stalls) shifted 2 plan units
east: JS `x0` 284.5→286.5, `carport.x` 278→280; DB `relative_x` 437.21→440.35 (both depots).
Migration 0027 applied. Verified with live simulation: 25/25 stalls occupied normally, 0 interrupted
bookings. Clearance went from −0.9u to +3.85u. Mirrors the west avenue pattern (4.1u clearance,
no hotspot).

### ✅ P1-6 · ~~Four different cars exist~~ — **RESOLVED (PR #60, #61)**
`memory/reference_depot_plan_scale.md`

**FIXED 2026-08-09 by Hermes Agent.** PR #60 scaled 3D car geometry 2.08× to match 2D (10.2 plan
units). PR #61 aligned CAR_LENGTH from 9→10.2. All three car lengths now agree: 2D=10.2u, 3D=10.2u,
physics=10.2u. Fixture test confirms stuck samples dropped 130→73 (44%).

### P1-7 · The geometry guard has four holes
`memory/project_depot_layout_unification.md`

1. **NULL handling** (fixed) — Postgres `LEAST`/`GREATEST` skip NULLs, so 5 dimensionless bays
   inherited their comparand's edges and produced **725 phantom overlaps** (779 instead of 54). The
   JS twin had the mirror bug (`null/2 = 0` ⇒ zero-area point) and printed **"PASS: zero overlapping
   pairs"** *plus* advice to delete all five founder exemptions as stale.
2. **The fence check silently skips depot 2** — it INNER JOINs one `FENCE-PERIMETER` row that only
   exists for depot 1, so depot 2 contributes nothing and **passes unearned.**
3. **No aisle check on the DB side at all** — aisle width is measured only source-side in the JS
   guard.
4. **No stall-vs-LANE clearance check** — which is exactly why P1-5 exists. **The guard would have
   blessed it forever.**

### P1-8 · The `ottoq_fleet_operator_slas` table is empty
`memory/project_appointment_depot_doctrine.md`

Chase's ruling: the 80% deploy floor must be a real **per-operator SLA**. The table is empty, so
every "80" in the proposer, the manifest generator, and shield rule SLA.001 is an **undefended
hardcoded fallback.**

### P1-9 · Reservation TTL is decorative
`memory/project_forward_availability_doctrine.md`

`stalls.reserved_by` / `reservation_expires_at` are honoured **lazily at claim time** by 13+
functions — but nothing garbage-collects them.
⚠️ **Three functions read `reserved_by IS NULL` WITHOUT the expiry guard** and therefore **under-count
free capacity**: `ottoq_cuopt_refresh`, `ottoq_report_charger_fault` (this starves fault recovery —
only 10 of 17 free DCFC visible), and `ottoq_reoptimize_reservation_book`.
⚠️ The executor's vacate does **not** clear `reserved_by` / `reservation_expires_at` — a capacity
leak that reads as phantom scarcity.

### P1-10 · Rule 1 (OTTO-Q never writes world state) is not enforced
`memory/project_ottoq_twin_boundary.md`

`ottoq_decide_tick` runs 8 direct `UPDATE vehicles` / `UPDATE stalls`. A designed inversion was
**adversarially reviewed and rejected — 3 of 4 reviewers said REJECT.**

**Do not attempt the naive "delete the UPDATEs, set apply_required" approach.** The blockers:
- The actuation window is **30 sim-minutes**, not the 1–3 minutes every risk rating assumed.
- Reservations die mid-flight (600 s / 900 s TTLs against an 1800 s window; `ottoq_reserve_stall`
  never renews).
- There is **no apply-time re-validation** — the executor's occupy is unconditional;
  `ottoq_validate_assignment` runs pre-flight only.
- ⚠️ **The safety net is backwards.** `idx_stalls_one_vehicle_per_stall` is
  `UNIQUE ON stalls(current_vehicle_id) WHERE NOT NULL` — it enforces **one STALL per VEHICLE**, the
  opposite of its name. It **structurally cannot** detect two vehicles claiming one stall.
- A same-tick reconciler race: `ottoq.ottoq_place_unplaced_vehicles` runs ~2 statements after
  `decide_tick` in the **same call**, and its cursor is exactly the in-flight population.
- **52% is un-invertible today** — `stage`(175) + `enter_wash`(17) + `enter_service`(10) = **202 of
  391 commands carry no `stall_id`**, and the executor's apply is gated `AND v_stall_id IS NOT
  NULL`. Flagging them `apply_required` marks them `executed` **while the world never moves** — and
  `stage` is the redeploy gate, so **the depot silently stops sending cars out.**
- **A/B invalidation:** `decide_tick` is the treatment arm only. The baselines write instantly, so
  the latency tax lands on OTTO-Q alone.
- **Two latent silent kill-switches:** `confirm_commands` sits inside `IF v_feed_sim THEN`, so the
  first depot flipped to a live feed **freezes entirely**, surfaced only as a `RAISE WARNING`; and
  `ottoq_api_otto_q_decide` never calls `confirm_commands`, so decide-alone becomes a no-op depot.

**The safe prep is fully specified in that memory file. Do that first.**

**The cheapest real enforcement available today:** `ottoq_events` already blocks UPDATE/DELETE with
a BEFORE trigger regardless of grants. **The same trigger pattern on `vehicles.current_stall_id` and
`stalls.current_vehicle_id` makes rule 1 real with no schema DDL.**

### P1-11 · The refusal path has never fired
`memory/project_ottoq_twin_boundary.md`

`ottoq_vehicle_commands.status` allows `'refused'` and `ottoq_ack_vehicle_command` accepts a
refusal — but **31,157 executed, 1,566 issued, ZERO refused / confirmed / expired, ever.**
`ottoq_sim_confirm_commands` stamped `executed` unconditionally.

Partially addressed by the pre-flight validation work (refusals now occur at issuance with
`confirmed_by='otto_q_preflight'`). **Verify the current state before assuming either way.**

### P1-12 · Bay no-show
`memory/project_orchestration_build_2026_08_01.md`

37.3% of bay bookings once died `no_show_grace_elapsed` — OTTO-Q reserves a bay and nothing
reliably moves the car into it. Driven to **0.0%** in phase 7, then **regressed to 7.3%** in phase 9
with the grace window never widened.
**Prime suspect: the departure-readiness gate pins vehicles in staging while their bay booking keeps
ticking — the gate manufacturing its own no-shows.**

`ottoq_bind_unbooked_bay_occupants` gives up `no_free_space` **34× per run**; one vehicle sat in a
wash bay ~23 real-minutes unrecorded. **Reality must outrank a plan:** displace the stale claim,
book the car that is physically there, emit an auditable conflict row.

### P1-13 · `leg_id` on only 3.7% of enacted bookings
`memory/project_orchestration_build_2026_08_01.md`

4 of 108 enacted bookings carry a `leg_id` ⇒ for **96.3% the calendar cannot answer "why is this
vehicle in this space."** The "one atomic operation" is half true: the stall matches, **the reason
does not travel.**

### P1-14 · The twin's wear model is frozen
`memory/project_arrival_forecast_learning_doctrine.md`

Only `drive_km_total`, `drive_hours_total`, `soil_index`, and `open_dtc_count` accumulate.
**Tire tread, brake wear, battery SoH, sensor health, software version and odometer are drawn once
at boot and frozen — a tire never wears.** Those are exactly the dimensions the founder's named
services (tire inspection, brake inspection, battery health check, software update) depend on.

⚠️ **This is not a contradiction of D5** (needs are a draw, not accumulated wear). The draw sets the
*starting condition* correctly; the issue is that **completion and elapsed time never move it**, so
a vehicle's condition is static within and across a run. Confirm with Chase what "stateful" should
mean here before building — it borders on the explicitly-non-required wear model.

### P1-15 · Migration 0005 needs a narrow revert
`memory/project_db_outage_2026_08_05.md` · branch `fwd2-inspection-condition-resets` (pushed, **not merged**)

Keep §3 (inspection wiring — proven correct). **Remove §2's unconditional
`exterior_soil_level := round(soil_index, 3)`**: it copies a *sensor*-soil counter (structurally
capped ~0.30) into a *body*-soil field banded at 0.45/0.65/0.85, so it can **only ratchet a vehicle
toward `ok`.** It had already laundered **8 of 17** moved vehicles out of the wash bands with no
wash bay involved. Also gate the `open_fault_codes := '{}'` / `worst_fault_severity := 99` writes
riding the same array.

### P1-16 · `service_cadence_policy.seed_phase_max` is decorative
`memory/project_needs_draw_already_exists.md`

It holds 1.45 / 1.18 / 1.45 / 1.50 — the exact constants **hardcoded inside**
`ottoq_seed_vehicle_need_profiles` — but the seeder **never reads the column.** Tuning it changes
nothing while appearing to. **Either make it read, or delete the column so it stops lying.**

### P1-17 · Mock fallbacks in the cockpits
`memory/reference_live_data_model.md`

Several OTTO-PULSE hooks (vehicles / stalls / incidents) still fall back to mock arrays on API
error. **An honest empty state beats a plausible lie.** This directly undermines the ecosystem
tie-in mandate.

---

### P1-18 · Forward bay reservations are written but have NEVER bound ⭐
`otto-q-core` MIGRATION_LOG rows 0011, 0014, 0015, 0016 · evidence tables `ottoq_bay_binding_witness`,
`proof0012_binding_blocker`, `p0014_prestop_*`

**This is the sharpest open problem in the forward-scheduling core, and it is a good one.**

Migration 0011 made the return signal reserve service and wash bays ~30 sim-minutes before arrival —
**15 pre-arrival holds written, 7 of 7 whose vehicle genuinely arrived booked BEFORE arrival, median
lead 30.8 sim-min.** The mechanism works.

**But no hold has ever reached `active` and `done`.** Final census on the certification run:
**18 held, 1 released, 0 active, 0 done.**

**The sharpest single piece of evidence** (`proof0012_binding_blocker`, 1 row): booking `49d1c436` is
the only `return_signal_prearrival` hold whose window opened while its vehicle was actually inside
the depot — it reserved `NASH-SVC-01` for 14:48–15:25, sim clock 14:52, **inside the window. It did
not bind.** It was released at 14:52:23 with `release_reason = 'replanned_no_window'` while the
vehicle sat at `NASH-L2-STALL-29` in state `charging_l2` — i.e. **legitimately upstream in its charge
leg, exactly where the full-service-visit doctrine says it should be.**

⇒ **Binding is not blocked by run length or tick cost. The replanner gives the bay up instead of
pushing the hold later.** That is a design question, not a bug hunt.

**Three migrations have already attacked it, and each surfaced a real defect:**
- **0014** — the proof was unprovable post-hoc, because `vehicles.current_stall_id` is a point-in-time
  scalar with no history **and the success path renames the evidence.** Added
  `ottoq_bay_binding_witness` + an AFTER UPDATE trigger stamping old/new state, old/new source, the
  vehicle's *actual* stall, the window, and **both clock domains** at the instant of transition.
- **0015** — a ready car now takes a free bay instead of waiting on a forecast. New
  `ottoq.ottoq_vehicle_bay_ready(run, vehicle, clock)` (TOTAL — any error returns FALSE, and FALSE
  only ever means "fall back to today's behaviour"). The window gate was **narrowed, not deleted**:
  `lower(during) <= clock` became `(lower(during) <= clock OR bay_ready(...))`, while
  `upper(during) > clock` is **untouched**, so a lapsed hold still never seats.
- **0016** — a car seated by OTTO-Q now carries the twin's service timer (`svc_step`,
  `service_ends_at` pinned to the booking's own window end), so **its work actually takes as long as
  it was planned to take.** 0015 made binding real and thereby exposed that latent partial seam.

**What is left:** get one hold to `state='active'` with `vehicles.current_stall_id = booking.stall_id`,
and then to `done`. Until that happens, the forward calendar reserves bays that nothing sits in.

### P1-19 · The metronome ceiling guard shipped dead, and the pattern will recur
`otto-q-core` MIGRATION_LOG rows 0012, 0013

0012 added a guard so the metronome stops before `statement_timeout` guillotines it mid-tick. It read
the deadline with `current_setting('statement_timeout', true)::numeric`.

**`current_setting` returns the GUC *display* form.** On this instance that is `'2min'`, while
`pg_settings.setting` is `'120000'`. `'2min'::numeric` raises 22P02, the function's own EXCEPTION
handler set the ceiling to 0, and the guard's `IF` was false on every call. **It was dead code from
the moment it shipped.** 0013 fixed it by reading `pg_settings.setting`.

> ⭐ **The generalisable rule: `current_setting()` gives you the display string; `pg_settings.setting`
> gives you the raw value in the GUC's base unit. Never cast the former to a number.**

This is the same family as every other vacuous guard in this codebase. **Prove a guard has fired.**

## 🟡 P2 — real, lower urgency

| # | Issue | Detail |
|---|---|---|
| P2-1 | **~300 scratch tables in `public`** | 461 tables total. `phase9_*`, `p0019_*`, `fwd2_*`, `build3_*`, `preimport_*`, `p12_contaminated_run904_*`… Deliberately named outside the `ottoq` prefix so the purge would not sweep them. Some are the **only** surviving evidence for a published certification — inventory before deleting. |
| P2-2 | **Only one fitted cross-variable correlation** | The biggest realism gap in the twin. Reality co-moves; the twin's knobs are independent. |
| P2-3 | **Tick cadence is irregular** | 8–31 real seconds between ticks while compute is ~1.15 s. Motion smoothness is bounded by the pg_cron metronome, not the engine. |
| P2-4 | **Photoreal tier renders but does not move** | The RTX view shows the depot; OTTO-Q assignments are not pushed over the WebRTC data channel. |
| P2-5 | **EC2 public IP changes on stop/start** | Cockpit override via `localStorage('ottoq_omniverse_ip')`. **An Elastic IP would fix it permanently.** |
| P2-6 | **Bridge token + AWS URL hardcoded** | In edge fn `ottoq-energy-mpc` and the replan defaults. Move to Supabase secrets. |
| P2-7 | **`ottoq-progress` edge fn is orphaned** | Points at dead `vehicle_schedules` / `schedule_tasks` (stale since 2026-06-19). Retire or repoint. |
| P2-8 | **OrchestrAV realtime is dead code** | `useOTTOQRealtime` subscribes to `ycsis` tables, which are dead for `gxdrc` views. Remove or repoint; poll instead. |
| P2-9 | **The user → operator binding is spoofable** | Client-side only, anon key, no JWT. Fine for demo; real tenancy is the security gate. |
| P2-10 | **Purge orphans rows permanently** | It deletes children `WHERE sim_run_id IN (SELECT … FROM ottoq_sim_runs …)`, so it can only reach children of runs that **still exist.** 252 `ottoq_vehicle_commands` and 79 `ottoq_events` rows are stranded forever. |
| P2-11 | **`trg_ottoq_auto_incident_report` is DISABLED** | It ran a synchronous scan of an 11 GB table inside `decide_tick` on **every** `severity='safety_critical'` insert (63 in 15 minutes). **Leave disabled**; a queued replacement is future work. |
| P2-12 | **`ottoq_recommendations`: 83,812 rows, 0 ever executed** | Nobody has ever executed one. Either wire it or retire it. |
| P2-13 | **Parking squats 55.8% of the inspection zone** | Reported at phase 10. |
| P2-14 | **The harness mis-counts inspection stalls** | It classified **14** via `stall_kind` while the zone has **28** (`zone='arrival_inspection'`) — same numerator, double the reported utilisation. |
| P2-15 | **Black-box frontend panel** | Backend built and verified (`ottoq-run-blackbox?run=<uuid>`, 21.8 MB in 12 s). The frontend panel was pushed and awaits merge — **check whether it landed.** |
| P2-16 | **Real MQTT broker** | The comms layer is built to OEM spec (MQTT topics, SAE J2735 BSM, ISO 20078 envelope, J3016/PAS 1886 teleop) and proven round-trip, but emulated in-database. A real broker is the remaining piece. |
| P2-17 | **`ottoq_sim_auto_dispatch_tick`'s vehicle-pick loop is not depot-scoped** | Only bites under concurrent multi-depot runs, which do not happen today. |
| P2-18 | **Phantom bookings** | 5 of 108 `otto_q_enacted` bookings had no decision behind them, written by a bay-exit reconciler outside the decision ledger. Driven to 0 in a later phase — **re-verify.** |

---

---

## 🆕 Subsystems that exist but this package originally documented nothing about

Found on the second pass, because the local clones were 77 commits behind. **Read the
`otto-q-core/MIGRATION_LOG.md` rows for these before touching anything nearby.**

### Rider-flagged cleaning recall — migrations 0018, 0019, 0020 (all applied 2026-08-08)

**This is the "push wake" from the runtime cadence doctrine (D12), now built.** A rider flags a
vehicle as needing cleaning; OTTO-Q recalls it mid-deployment rather than waiting for its next
scheduled return.

- **0018** — new table `public.ottoq_rider_cleaning_flags`, 3 new columns on `vehicle_need_profile`,
  6 policy rows, 3 functions. Also fixed the unseeded wash rotation (see P0-6).
- **0019** — *"0018 made the recall reach a wash bay. It did not take the car off the road, which is
  the claim the demo makes."* Measured on 0018's own runs: **9 of 10 flag-linked dispatches were sent
  out with the flag ALREADY due** (flag age at dispatch 43 → 1,349 min) and recalled one tick later —
  only **1** was a genuine mid-deployment recall. **2 of 18 flags never fired at all**, because they
  matured while the vehicle was parked. New `public.ottoq_rider_flag_due` and
  `ottoq.ottoq_rider_flag_indepot_sweep`; 4 existing routines replaced.
- **0020** — makes flag consumption and placement atomic. 3 new trigger functions
  (`ottoq_rider_flag_placement_guard`, `ottoq_rider_flag_mark_served`,
  `ottoq_reanchor_rider_flags_on_clock_rebase`). **Dropped one constraint** —
  `ottoq_visit_needs_vehicle_id_visit_key_key`, a **run-blind uniqueness** that was itself the defect
  — with its exact definition captured verbatim to `public.mig0020_prestate` first. Both replaced
  function bodies were **md5-guarded against their pre-images**.

### One-vehicle-one-stall — migration 0021 (applied 2026-08-08)

**Found while certifying 0020, not reported by anyone.** Run `0a3c6910` froze at tick 5 and never
advanced. The metronome kept firing every minute and kept **rolling the entire tick back** on
`duplicate key value violates unique constraint "idx_stalls_one_vehicle_per_stall"` — **20
consecutive times.** `tick_count` sat at 5 while `next_tick_due_at` kept moving, **so from the
outside the run looked alive.**

> *"That is the same failure family as every other defect on this project: motion reported but not
> earned."*

Located exactly via `PG_EXCEPTION_CONTEXT`: `twin.ottoq_sim_advance_service_flow` line 351, whose
`WHERE` clause **guards the wrong side of the index.** Fixed by adding
`public.ottoq_stall_seat_is_exclusive` + a `BEFORE INSERT OR UPDATE OF current_vehicle_id` trigger on
`stalls`. **This partially addresses E4** — but the backwards index itself still exists.

### Migration 0017 — written, deliberately NOT applied

`0017_stop_is_two_phase_so_a_run_can_always_be_stopped.sql` **replaces the START engine**, and the
instance was still freezing episodically when it was written, so it could not be tested safely.
**Leave it alone unless you can certify it.**

### Migration 0022 — applied, but on an unmerged branch

`a_run_owns_its_rows`, version `20260808182226`, on branch `p0022-run-scope-integrity`
(+3 commits ahead of `origin/main`). *"45 FKs where zero ever exist."*
**This is the only unmerged branch in any OTTOYARD repo.**

---

## ✅ Recently resolved — do not re-investigate

| Issue | Resolution |
|---|---|
| **The 14 GB database outage (2026-08-05)** | Resolved. **14 GB → 652 MB**; ticks 48–65 s → ~1 s. Root cause was event-write amplification, not bloat: ~1.7 KB of live JSON per event across 4M events. |
| **"The off-map render bug"** | **Never reproduced.** 0 sightings across a captured 116-vehicle run. It was recorded as an established fact and handed to a fresh session as a premise, costing a wasted hypothesis. |
| **"cuOpt is async"** | **Falsified.** The hosted LP is synchronous — 12/12 HTTP 200 with inline solutions, 547–721 ms at production scale. The real cause was **ordering** (pg_net dispatches only after COMMIT). |
| **The booking-ledger divergence** | Fixed. Two uncoordinated stall selections; agreement went 3/81 (3.7%) → 79/80 (98.8%). ⚠️ **All booking-derived capacity numbers from before 2026-08-01 are VOID.** |
| **The `leg_type` abort** | Fixed. One unmapped twin service code (`interior_inspection`) hit a CHECK constraint and **aborted the entire `decide_tick` transaction every tick** — zero stall assignments — while cron reported `succeeded`. `perimeter_walkaround` was a second, still-unexploded instance. Fixed via `ottoq_svc_to_leg_type(text)`. |
| **The RLS public-policy hole** | Fixed. 31 tables were world-writable behind policies *named* "Service role full access" that were actually `TO PUBLIC`. |
| **The phantom-charger baseline** | Fixed and structurally prevented by a partial unique index. **54,274 overlapping session pairs, max 100 vehicles on one charger.** |
| **The `prime_fingerprint` claim** | **False.** It is not a determinism mechanism — no such function, column, or table. It is a local variable emitted into an event payload. Determinism comes solely from `ottoq_sim_seeded_random` and `ottoq_crn_draw`. |
| **"planned_return_at is the arrival estimate"** | **False.** It is `GENERATED ALWAYS AS dispatched_at + planned_duration_min` — the dispatch plan. Median divergence from the live ETA: **−114.9 min.** Anchor on the **live ETA**. |
| **"service_definitions speaks the twin's vocabulary"** | **False.** Its 9 active codes share **1 of 15** atom names with the needs card. `service_cadence_policy.lane` is the live requirement column. |
| **"The outbound itinerary blocks inbound planning"** | **False.** `twin.ottoq_sim_dispatch_vehicle` calls `ottoq_release_visit_artifacts` at dispatch. Measured: 63 of 63 `depart` legs are `skipped` on `completed` itineraries; **zero** are `planned`. |
| **"The metronome is 35.7% of DB time"** | **Retracted** — a measurement artifact. `pg_stat_statements.track` is `top`, which rolls nested work into the caller. |
