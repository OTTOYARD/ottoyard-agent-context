# 02 — OTTO-Q: the decision brain

## 1. The architecture in one named pattern

OTTO-Q implements **Simplex / Runtime Assurance**, a published safety architecture from avionics.
This is deliberate: it is citable, an OEM safety engineer recognises it, and it gives a principled
answer to "how do you let AI drive a depot without letting AI make an unsafe call."

```
                    ┌──────────────── frozen decision frame ────────────────┐
                    │  capture the world once, hash it, decide against it   │
                    └───────────────────────────────────────────────────────┘
                                            │
  L2  PERFORMANT LAYER          ────────────┴────────────
  (may be an optimizer, an        propose an action
   LLM, a heuristic — anything)   cuOpt · Nemotron · greedy · needs-card
                                            │
  L1  DETERMINISTIC SHIELD      ────────────┴────────────
  (52 rules, non-raising,         allow → enact
   fail-closed)                   deny  → substitute the L1 safe default
                                            │
  L3  AUDIT / GRADE             ────────────┴────────────
                                  record decision, rule evals, counterfactual
```

**The doctrine, stated as the intelligence service's own README states it:**
*model proposes → optimizer disposes → shield guarantees → loop learns.*

L2 can be as clever as you like — that is where cuOpt, Nemotron, and any learned model belong. L1
is dumb, deterministic, and has the final word. Nothing reaches the world without passing L1.

---

## 2. The runtime loop

### 2.1 Two clocks that are NOT the same thing

This is the single most misunderstood part of the system, and Chase corrected Claude on it directly.

- **The twin's world step** advances physics — vehicles move, batteries charge, weather changes.
  Measured warm: **~914 ms median**.
- **OTTO-Q's decision pass** thinks. Measured warm: **~181 ms median**.

⇒ **OTTO-Q already answers in ~0.18 s.** The twin's physics costs ~5× the brain. Smooth 1:1 motion
is therefore a **twin-side engineering problem, not an OTTO-Q speed problem.**

⇒ **A "tick" is not OTTO-Q's decision rate.** Do not conflate them. They are already separate
functions (`twin.ottoq_sim_advance_tick_world` vs `ottoq_sim_decide_and_dispatch`).

### 2.2 What wakes OTTO-Q (founder doctrine, binding)

Both of:
- **A slow constant background scan** — *"especially analysing energy/grid/battery triage."*
- **Push/spontaneous activation** on a need — *"a random flag for immediate vehicle return for
  cleaning or sensor, etc."*

### 2.3 The run lifecycle is a hard contract

- **NOTHING RUNS UNTIL START.** Neither OTTO-Q nor the twin. On START the twin *"comes alive and
  begins selecting variables for OTTO-Q to begin sorting."* Depot ticks only on an explicit start.
- **PAUSE freezes everything in time** within the twin.
- **STOP completely resets and wipes the twin clean** for a new run — **while saving that run's data
  log.** Destructive to the world, preserving to the record. Both halves are required.

Functions: `ottoq_start_demo_run(scenario, speed, days, seed)` · `ottoq_pause_run(run, reason)` ·
`ottoq_resume_run(run)` · `ottoq_sim_stop_and_reset(run, reason)` → which calls `ottoq_archive_run`
**before** the wipe, with no exception handler, so a failed archive fails the stop loudly.

⚠️ **RESUME must re-anchor `last_tick_at`.** `advance_tick_world` derives elapsed time from
`clock_timestamp() - last_tick_at`; a paused run keeps a stale anchor, so the first tick after
resume would jump the sim clock by the entire paused interval. This is fixed — do not regress it.

### 2.4 The clock

**`live` playback mode delivers exact 1 real second = 1 sim second** (measured boundary-aligned:
ratio 1.0000 over one tick, 1.00017 over two). A viewing multiplier (2×, 3×) scales the twin, and
**OTTO-Q keeps real-time latency regardless** — no brain-clock scaling, no sim/real latency
conversion. Chase: *"OTTO-Q will make decisions that won't play out for several minutes at a time."*

⚠️ **The right instrument for measuring the clock is `sim_clock_current` vs `last_tick_at`**, both
written by the same statement, so they are phase-aligned by construction. Comparing a
step-advancing clock against a continuous `now()` produces up to a full tick of phase error at each
end — that is how a bogus 1.246 ratio was once reported.

**The remaining gap is cadence, not ratio.** Ticks fire ~8–31 real seconds apart while tick compute
is ~1.15 s. Smoothness is bounded by the pg_cron metronome's firing cadence.

### 2.5 The twin never freezes for a decision

*"Twin should be in an absolute state of motion with vehicles, chargers, energy etc always firing
and rendering."* If OTTO-Q cannot place a vehicle in time, the answer is **not** to block and **not**
to fail — it is **temporary staging**, the designed pressure-relief valve, then re-plan.

---

## 3. The five decision types

`ottoq_decide_tick(p_sim_run_id)` is the orchestrator. For each decision it: builds a context →
asks L2 for a proposal → runs it through the L1 shield → enacts or substitutes the safe default →
records to `ottoq_decisions`.

| # | Decision | L2 proposer | L1 safe default |
|---|---|---|---|
| 1 | **stall_assignment** — which space does this vehicle go to | `ottoq_l2_propose_stall_assignment` | `ottoq_l1_safe_default_stall` |
| 2 | **redeployment** — which vehicle goes back out | `ottoq_l2_propose_deploy` | `ottoq_l1_safe_default_deploy` |
| 3 | **charge_disposition** — charge now, hold, or stop | `ottoq_l2_propose_charge_disposition` | `ottoq_l1_safe_default_charge_disposition` |
| 4 | **service_sequencing** — order of the scarce service bay | `ottoq_l2_propose_service` + `ottoq_service_priority_propose` | `ottoq_l1_safe_default_service` |
| 5 | **energy / BESS** — battery setpoint and charge cap | `ottoq_l2_propose_bess`, `ottoq_energy_orchestrate` | `ottoq_l1_safe_default_bess` |

**The external-proposal seam** is how AI gets in: `ottoq_l2_external_proposal(run, action_context,
entity_type, entity_id)` reads `ottoq_external_proposals` and **prefers `source='cuopt'`** within a
freshness window. Loops 1, 2 and 5 all consume it via `COALESCE(external_proposal, heuristic)` —
so the heuristic is always the safe fallback and the shield is untouched either way.

Proposals are submitted by `ottoq_submit_external_proposal`, which **always writes status
`pending`** — it is structurally incapable of writing a decision or marking anything enacted. That
is the provenance guarantee: an external model cannot fake an enactment.

---

## 4. The L1 safety shield

**52 rules in `ottoq_rules`, 29 currently active.**

| Category | Rules | Active | Enforcement |
|---|---|---|---|
| `energy_safety` (EN.001–005) | 10 | 5 | block |
| `sla_contract` (SLA.001–007) | 12 | 6 | block / warn / log_only |
| `state_machine` (SM.001–005) | 7 | 4 | block |
| `time_window` (TW.001–005) | 7 | 6 | warn / log_only |
| `hardware_safety` (HW.001–006) | 6 | 3 | block |
| `concurrency` | 4 | 2 | block |
| `audit_integrity` | 2 | 1 | block |
| `role_authorization` | 2 | 1 | block |
| `sensor_liveness` | 2 | 1 | block |

Each rule has an evaluator function `ottoq_eval_<code>` (e.g.
`ottoq_eval_sla_001_min_soc_at_deployment`, `ottoq_eval_hw_001_connector_compatibility`).

**Entry points:**
- `ottoq_evaluate_rules_for_action(action_context, entity_type, entity_id, context, ...)` — the
  primitive. Runs every rule that applies to that action.
- `ottoq_shield_probe(...)` — **non-raising** wrapper. This is what makes runtime assurance
  possible: you can ask "would this be allowed?" without an exception.
- `ottoq_shield_and_log(run, tick, depot, actions, shadow)` — batch evaluation with logging.
- `ottoq_l1_override_authorized(...)` — override authority gate. Some rules (EN.005 grid-event
  hardstop) are **non-overridable**.

**Design properties that matter:**
- **Fail-closed** on evaluator error at block tier. An evaluator that crashes denies, it does not
  pass.
- **Non-raising core** — the shield returns a verdict, it does not throw into the caller.
- Every evaluation is written to `ottoq_rule_evaluations` (~87k rows currently).

### ⚠️ The honest framing of the safety record

Read this before you ever describe the shield to anyone.

An adversarial re-audit (2026-07-23) found that **no function that transitions a vehicle to
`deployed` calls the rule-engine shield.** `ottoq_shield_and_log` defaults to `p_shadow = true` and
has zero deploy-path callers. Safety on the deploy path actually holds via:
1. an **inline deterministic precondition** — `WHERE current_soc >= 80` in the dispatch caller
   SELECTs, and
2. an independent `ottoq_deploy_log` trigger recording SoC-vs-floor at the transition.

The shield **does** enforce (override-to-default) for stall_assignment, charge, and energy
contexts. Just not the → `deployed` transition.

**Therefore:** the absolute safety record is real (zero deploys below the stranding floor across
1,012+ dispatches), but call it an **inline deterministic precondition**, not a "rule-engine
shield," and never present "0-unsafe vs baseline" as a *shield differentiator* — the naive baseline
shares the same inline filter, so that A/B proves shared code, not an edge.

**Closing this gap — routing the deploy transition through the shield — is real, high-value work.**

---

## 5. Determinism and reproducibility

- **CRN (Common Random Numbers):** `ottoq_crn_draw(run_seed, stream, slot_key, sim_day, tick)` and
  `ottoq_crn_normal(...)`, both IMMUTABLE over `hashtextextended`. Plus
  `ottoq_sim_seeded_random(seed, salt)`. **These two are the only determinism mechanisms.**
  (`prime_fingerprint` is *not* one — it is a local variable emitted into an event payload. Claude
  once claimed otherwise and was wrong.)
- CRN pairing is what makes A/B arms comparable: the same seed produces the same world for every
  policy, so differences are attributable to the policy.
- A run replays exactly from its seed. That is why `ottoq_run_archives` stores the **reproducibility
  key** (scenario + seed + policy + depot) plus outcome metrics rather than 24 MB of raw bytes.

**The frozen frame:** `ottoq_capture_decision_snapshot(run, tick, depot, clock)` writes to
`ottoq_decision_snapshots` with a sha256 integrity assert (`ottoq_assert_snapshot_integrity`).
Decisions are made against a captured frame so the world cannot shift underneath a decision
mid-evaluation, and so a decision can be replayed and graded later.

---

## 6. The policy arms (A/B)

`ottoq_sim_decide_and_dispatch` routes by the run's policy:

| Policy | Function | Role |
|---|---|---|
| `otto_q` | `ottoq_decide_tick` | **The treatment arm.** The product. |
| `greedy` | `ottoq_greedy_tick` | Naive baseline — lowest-SoC-first, seeded-random stall pick, one attempt per tick. |
| `fifo` | `ottoq_fifo_tick` | First-in-first-out baseline. |
| `manual` | `ottoq_manual_tick` | Human-ops baseline. Real, 56 benchmark rows. |

⚠️ **`decide_tick` is the treatment arm only, not a shared base.** Any latency or constraint you
add to `decide_tick` lands on OTTO-Q alone and biases the A/B *against* the product.

The baselines were **each found to be dishonestly favourable at least once**. Read
`memory/reference_ottoq_real_edge.md` in full before running or quoting any A/B.

---

## 7. The forward availability calendar

**This is the conceptual heart of OTTO-Q, and it is what makes it an orchestrator rather than a
queue.**

> Orchestration is **a forward-time occupancy calendar of every space**, not a refusal protocol.

`ottoq_stall_bookings` is that calendar. A booking is born `held`, passes through `active`, and ends
`done`, `released`, `superseded`, or `interrupted`. Overlap is prevented by an EXCLUDE GIST
constraint (`ottoq_stall_bookings_no_overlap_v3`, covering held/active/done/interrupted).

⚠️ **A calendar whose rows are born `done` cannot express "this space is claimed T1→T2".** The
original overlap guard was scoped `WHERE state IN ('held','active')` while **no booking was ever
written in those states** — so every prior "zero double-bookings" result was **vacuous**. That is
fixed. Do not regress it.

⚠️ **Availability must count only `state IN ('held','active','done')`.** Counting `released` and
`superseded` rows (86% of the calendar, still carrying full ~23-minute windows) is what told cuOpt
there were zero free stalls while ~27 chargers were physically free.

---

## 8. Forward service scheduling (the core doctrine, expanded in `06_DOCTRINE.md`)

> **"Overdue" is a failure state, not a trigger.**

A vehicle must never leave the depot needing a service. Work is **predicted and reserved before the
next arrival** — the workflow exists *before* the vehicle gets there. If work is overdue, the system
has already failed.

Supporting machinery:
- `public.ottoq_vehicle_needs_card` — a **66-column view, one row per vehicle**: `must_do_now[]`,
  `deferrable_now[]`, `must_do_legs`, `overall_urgency`, `next_deploy_at`, `minutes_to_deploy`,
  `est_charge_min`, `open_must_do_min`, **`fits_window`** (does the required work fit before
  redeploy?), `need_statement jsonb`. Ten per-dimension urgency ladders graded
  `ok | due_soon | due | overdue | critical`.
  ⚠️ **Its run column is `run_id`, NOT `sim_run_id`, deliberately.** `ottoq_purge_prior_runs` scans
  `information_schema.columns` for `sim_run_id` on `ottoq%` names — including views — and would
  issue a failing DELETE against this view every run start. **Do not rename it.**
- `ottoq_run_boot_draw(run)` → `ottoq_seed_vehicle_need_profiles(run)` — draws ~24 per-vehicle
  condition variables at every run start (tire tread, brake wear, sensor health, soil index, …),
  seed-deterministic, PK'd per vehicle so each boot overwrites.
- `ottoq_plan_visit_itinerary`, `ottoq_build_workflow_plan` — turn needs into a timed itinerary.
- `ottoq_vehicle_itineraries` / `ottoq_itinerary_legs` — the timed legs. **Twin-owned.**
- `ottoq_evaluate_return_need`, `ottoq_ingest_vehicle_signal`, `ottoq_decide_return_on_signal` — the
  return-signal seam.

---

## 9. The command bus (how a decision becomes motion)

```
OTTO-Q                                       TWIN
  │                                            │
  ├─ ottoq.ottoq_validate_assignment  (PRE-FLIGHT, pure read)
  │     conflict? → record refused/'otto_q_preflight', never reaches the twin
  │                                            │
  ├─ ottoq.ottoq_emit_vehicle_command ─────────▶ ottoq_vehicle_commands (issued)
  │     RETURNS uuid (was void — that was a blocker)
  │                                            │
  │                          public.ottoq_sim_confirm_commands  ← PURE EXECUTOR
  │                            CRN delay, apply placement, mark executed
  │                                            │
  └─ ottoq.ottoq_react_to_refusals ◀───────────┘ (consumes refusals exactly once,
        books an alternative, emits a new command with reroute_after)
```

**Doctrine correction that produced this shape:** the twin **never checks-and-refuses.** Validation
belongs to OTTO-Q, pre-flight. A refusal on the ledger now always means *"OTTO-Q caught its own
conflict before issuing."* The twin is a pure executor.

⚠️ **The actuation window.** `confirm_commands` runs at the **top** of the world pass and
`decide_tick` runs **last**. So a command emitted at tick N is first examined at tick N+1. At
`tick_interval_seconds=30 × time_scale=60` that is **30 sim-minutes**. Every risk assessment written
against "1–3 minutes" is unscored. The safe prep is to add a **second** `confirm_commands` call
*after* `decide_and_dispatch` in the same transaction, which collapses the window to 1–3 minutes.

---

## 10. Rule 1 and the schema separation

The long-term architecture is OTTO-Q and OTTO-TWIN as separate services with separate databases.
Progress so far, inside one database:

- Schemas exist and are populated: **`twin` = 71 functions, `ottoq` = 55, `public` = the rest**
  (`public` also contains ~744 PostGIS functions, so its raw count of 1088 is misleading).
- `anon` has **USAGE on neither `ottoq` nor `twin`** — the decision layer and the twin engine are
  structurally unreachable from client keys and invisible to PostgREST.
- All 396 routines were pinned to `search_path = twin, ottoq, public, extensions` so future moves
  need zero function edits.
- 8 ambiguous functions were split into observation half (twin) and decision half (ottoq).

**Rule 1 — "OTTO-Q never writes world state" — is NOT yet enforced.** `ottoq_decide_tick` still runs
direct `UPDATE vehicles` / `UPDATE stalls`. A designed inversion was **adversarially reviewed and
rejected** (3 of 4 reviewers said REJECT) — the goal is right, the sequencing was wrong.
**Do not attempt the naive "delete the UPDATEs, set apply_required" approach.** The full list of
prerequisites is in `memory/project_ottoq_twin_boundary.md` and is summarised in
`docs/10_KNOWN_ISSUES.md`.

The cheapest real enforcement available today: `ottoq_events` already blocks UPDATE/DELETE with a
BEFORE trigger regardless of grants. **The same trigger pattern on `vehicles.current_stall_id` and
`stalls.current_vehicle_id` would make rule 1 real with no schema DDL at all.**

---

## 11. Vehicle states

`vehicles.current_state` has exactly **17 labels**. Observed in a recent snapshot:
`staged_for_departure`, `en_route_to_depot`, `charge_complete_holding`, `deployed`, `charging_dcfc`,
`charging_l2`, `in_service_bay`, `in_detail_bay`, `staged_awaiting_service`, `emergency_staged`,
plus `arrived_at_gate`, `en_route_to_deployment`, `in_wash_bay`, and others.

⚠️ **There is no "approaching" state, and there deliberately never will be.** A car 40 minutes out
and one 90 seconds out are indistinguishable in `en_route_to_depot`. The approach band is a
**derived, read-only view** (`ottoq_approach_zone(vehicle, run)` + view `ottoq_approach_band`)
computed from the existing ETA clock. No new state, no new writers, nothing can strand in it.

⚠️ **Every seam that maps a vocabulary must be a TOTAL function.** One unmapped word
(`leg_type`) once aborted every decision in the depot while cron reported success. If you write a
`CASE` over a vocabulary, give it an `ELSE` that is safe and loud.
