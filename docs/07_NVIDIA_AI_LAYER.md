# 07 — The NVIDIA / AI layer

> **The one-sentence architecture:** *model proposes → optimizer disposes → shield guarantees →
> loop learns.* AI is **always advisory**. Only the deterministic core enacts.

---

## 1. Why the AI sits on top of a deterministic core, and not the other way round

Chase's mandate (2026-07-14) was maximal: *"FRONTIER = state-of-the-art, optimizers and intelligence
engines on **every** decision surface, no hand-tuned heuristics. Leverage NVIDIA and open source.
Seek every ounce of alpha. Never go short or easy — all the way."*

But the architecture that survived adversarial review is **not** "AI everywhere." It is a **3-tier
runtime-assurance stack with strict authority**, because that is the only shape that is both
maximally intelligent *and* diligence-proof for an OEM.

| Tier | What | Authority | Share of enacted decisions |
|---|---|---|---|
| **1 — Deterministic core** | The spine and **sole actuator**. Sub-second reactive loop, L1 shield, per-vehicle stall/DCFC assignment, reservation honour, fault auto-reroute, energy-plan follow. | **Enacts** | ~99.75% |
| **2 — cuOpt** | Batch **planner** for the genuinely NP-hard regime. Off the hot path, on a solve-window cadence. | Advisory | — |
| **3 — Nemotron** | Advisory human-interface / semantic layer: exception triage, coarse regime selection, natural-language explainability, the approval co-pilot. **Never actuates, never produces the optimizer's numbers.** | Advisory | — |

**Every Tier-2 and Tier-3 output is advisory and must pass back through the L1 shield.**

**The honest reason Tier 1 stays dominant:** per-tick stall assignment is a **Linear Assignment
Problem** — polynomial, and greedy is provably near-optimal. An optimizer can only *tie* there.
Shipping a GPU solver at a problem good simple logic already solves optimally would be theatre, and
an OEM engineer will spot it. **cuOpt's real home is where the combinatorics are genuinely hard.**

---

## 2. cuOpt — what it is, what happened, what is true now

**What it is:** NVIDIA's GPU optimization service, used via the **hosted API** at
`optimize.api.nvidia.com/v1/nvidia/cuopt`. **No GPU purchase required.**

**Where it lives in the code:**
- Edge function `ottoq-cuopt-propose` — builds the LP and calls NVIDIA.
- `ottoq_cuopt_refresh(run)` — fires the request via pg_net.
- `ottoq_submit_external_proposal(...)` — writes the result as `pending` into
  `ottoq_external_proposals`. **Structurally incapable of enacting anything.**
- `ottoq_l2_external_proposal(run, context, entity_type, entity_id)` — the consumption seam. Prefers
  `source='cuopt'` within a freshness window.
- Telemetry: `ottoq_cuopt_fire_log`, `cuopt_invocation_log`, `ottoq_cuopt_deferrals`,
  `ottoq_cuopt_first_refusal_arm`, edge fn `ottoq-cuopt-lp-probe`.

### The history, because you will otherwise re-derive it

**Phase 1 — cuOpt fired and answered, and was never once used.** Across **84,000+ decisions,
`proposed_action->>'source' = 'cuopt'` was 0.** Not enacted, not overridden, not deferred. Zero.

**Phase 2 — a wrong root cause was published and later falsified.** The stated cause was *"cuOpt is
async, so proposals arrive stale."* A live probe destroyed that: **cuOpt's hosted LP is
SYNCHRONOUS** — 12/12 calls returned HTTP 200 with `nvcf-status: fulfilled` and a complete inline
solution. At production scale (40×45 = 1,800 vars) it answers in **547–721 ms**; at 4,800 vars,
1,396 ms with the solver itself at **18 ms** (76 simplex iterations). **There is no size in our
reachable range where it goes async.**

Also settled by that probe:
- Our LP payload is **correct** — accepted and solved to proven optimality in every variant.
  Omitting `variable_types`/`row_types`/`client_version` is fine. The LP relaxation came back
  **integral** at 4,800 vars, vindicating the total-unimodularity argument. Empty CSR rows are
  accepted, not an error.
- The parse path `response.solver_response.solution.primal_solution` is **correct**. Three rival
  paths returned nothing.
- ⚠️ **There is NO top-level `reqId` on a 200.** Top-level keys are exactly `["response","notes"]`.
  The request id exists **only** as the `nvcf-reqid` *response header*. Any code keying off
  `result.reqId` reads `undefined`.

**Phase 3 — the real root cause was ORDERING.**
1. `ottoq_cuopt_refresh` uses **pg_net, which only dispatches after COMMIT.** It was called from
   `ottoq_sim_decide_and_dispatch` in the **same transaction** as `decide_tick`. That call site
   **cannot ever work, by construction** — the HTTP request has not been sent when the decider reads
   the seam.
2. By the time cuOpt *was* asked, OTTO-Q's own scheduler had already parked the vehicles and locked
   the stalls, so the edge function hit its early return — which sits **before** it reads the API
   key. **NVIDIA was never contacted.** Measured: free stalls 39→35→21→8→3→2→0 against candidates
   0→0→0→0→0→0→11. **Never both non-zero.**
3. Even a landed proposal was structurally refused: `ottoq_honour_reservation_proposal` **Gate B**
   returned before the external seam was read. **54 of 71 stall assignments (76%) exited there.**

**Phase 4 — cuOpt enacted for the first time (2026-08-01).** A **split-tick** fix to
`ottoq_demo_metronome`: fire cuOpt on one beat and **COMMIT** (the commit is what makes pg_net
actually send), decide on the next. Result: NVIDIA answered 8 times, all HTTP 200, status "Optimal";
19 proposals; **2 enacted with `source='cuopt'`**, and two `begin_charge` commands executed with
cuOpt's exact payloads. **The depot moved on NVIDIA's numbers.** 19 external HTTP receipts
reconciled exactly to 19 DB rows, ruling out in-DB fabrication.

**Phase 5 — 2.8% → 42.4%.** Three further root causes, all measured, all fixed:
1. **The gated candidate set was discarded at the wire.** `ottoq_cuopt_refresh` computed a precise
   candidate list, then posted **only** `{sim_run_id}`. The edge function re-derived the cohort from
   live state ~4.1 s later against a 4.27 s tick — **one full tick late by construction.**
   **Fix: send `candidate_ids` in the body.** Runtime proof: all 54 invocations reported
   `cohort_mode="pinned"`.
   > ⭐ **Prefer a runtime marker like that over source inspection.** An earlier phase "fixed" the
   > edge function while it stayed byte-identical at v22.
2. **The healthy beat almost never ran.** The FIRE beat was throttled by `cuopt_contention_min=2`,
   and two cars are rarely at the gate at once. Removed the veto; stood down the DECIDE-beat fire
   entirely (it is structurally incapable of arriving in time).
3. **`trimmed_by_cap` silently deleted 100% of optimal solutions** — 21 invocations solved to
   `status:"Optimal"` and produced **zero** proposals because an energy cap discarded the answer
   *after* the solve. That is a flat violation of D7 (vehicle-first). **Fix: the cap is advisory.**

**Phase 6 — the current state, honestly.** cuOpt's share has oscillated: 2.8% → **42.4%** → 36.4% →
18.0% raw / 28.8% like-for-like. **Supply is always the bottleneck, not conversion** — when cuOpt
gets candidates and free stalls, conversion has measured **86.7% to 100%**.

**The smoking gun for the supply problem:** `free_stalls_in = 0` on **48 of 83** edge calls (57.8%)
while L2 utilisation was only 39.7% — **~27 chargers were physically free while cuOpt was told there
were none.** The availability predicate counted `released`/`superseded` bookings (86% of the
calendar), which still carry full ~23-minute windows.

**Also fixed along the way:** a **sim-clock vs real-clock domain mismatch** —
`ottoq_sim_advance_tick_world` set `ottoq_ocpp_chargers.last_heartbeat_at` in the **sim** domain
while the edge function filtered healthy chargers with `Date.now() - 90_000` in the **real** domain.
A run whose sim clock sat hours from `now()` saw **every charger as stale**.

### ⚠️ The blocker on every AI claim

`ottoq_sim_auto_dispatch_tick` re-picks vehicles by `soc DESC, seeded_random`, **discarding which
vehicle OTTO-Q / cuOpt / Nemotron chose.** Until that is fixed, our assignment intelligence is not
being tested at all and **no cuOpt or Nemotron contribution is measurable.** Verify whether this is
still true before claiming any AI result.

### Where cuOpt actually belongs

**Not** per-tick stall assignment (that is a polynomial LAP — a category error, and why it ties).
**Yes:** the **compressed overnight wave under charger scarcity** — a resource-constrained
scheduling / flexible job-shop problem where ~100 drained vehicles time-share scarce DCFC/L2/bays/
staff under a cumulative energy cap with **hard morning-deploy due dates.** Greedy first-fit is
arbitrarily suboptimal there: it cannot sequence in time, cannot save DCFC for who needs it, and
cannot shape load.

**The Phase-1 deterministic baseline for that regime is already shipped:**
`ottoq_plan_overnight_wave(run, morning_deploy_at, slot_min, dcfc_kw, l2_kw)` + table
`ottoq_wave_plan` — a real global scheduler (earliest-deadline-first, charger-class selection that
**prefers L2 to save scarce DCFC**, partial-charge-to-target, slot-capacity bookkeeping). Verified:
a 90-vehicle drained wave over a 7 h horizon planned 94 / stranded 0, **68 L2 + 26 DCFC** — the
DCFC-saving a greedy cannot do. Tightened to a 90-minute deadline it correctly reported 67 stranded
and `feasible_all = false`. **This is the honest floor cuOpt must beat.**

---

## 3. Nemotron — the advisory conductor and co-pilot

**Model:** `nvidia/nemotron-3-ultra-550b-a55b`, hosted NIM.

**Where it lives:**
- `ottoq-orchestrator-agent` (v15+) — "OTTO-Q PRIME", the conductor. Reads `ottoq_agent_board(run)`.
- `ottoq-nemotron-copilot` — the copilot.
- `ottoq-approval-copilot` — the supervisor's approval co-pilot.
- `ottoq-ottocommand` — natural-language command. ⚠️ **Its language model is Claude, not Nemotron.**
  Do not describe OTTOCOMMAND as NVIDIA-powered.

### What Nemotron does today
- Fires on a 3-tick heartbeat **or** on `ottoq_orchestrator_trigger(depot)` (gate rush ≥5 / holding
  backlog ≥25).
- Sets policy dials — 6 knobs: `deploy_surge_catchup`, `forecast_horizon_min`,
  `energy_reserve_shave`, `deploy_peak_fraction`, and others. **Never `deploy_floor_soc`.**
- Emits `ops_action`s dispatched by `ottoq_apply_ops_action`. Whitelist auto-executes
  (`raise_deploy_surge`, `extend_forecast_horizon`, `enable_energy_reserve`), each clamped.
  **Anything else routes to `ottoq_ops_approvals` as pending — never a silent drop.**
- Everything is audited into `ottoq_decisions` as `{applied, queued, rejected, rationale}`.

**Engineering notes that mattered:** disabling Nemotron's `<think>` reasoning traces (they overran
`max_tokens`, causing 60% fallback) plus `response_format: json_object` plus a robust multi-span
parser lifted the parse rate from **40% → 83%**.

> 🚨 **A defect worth remembering: `Math.round` destroyed Nemotron's numbers.** **106 of 109 dial
> writes landed on the extreme *opposite* of what the model asked for.** The model was working; the
> plumbing inverted it. **Verify a model's effect at the dial, not at the model's output.**

### The approval co-pilot — the pattern to copy

When OTTO-Q wants a discretionary in-depot reassignment, the D9 wall guard **denies** it and queues
an approval. `ottoq-approval-copilot` then gathers vehicle state, SoC, the proposed change, and
depot context, calls Nemotron, parses strict JSON
`{recommendation: approve|hold|reject, confidence, rationale, risks[]}`, and **merges it into
`payload->'copilot'` — never touching `status`, `decided_by`, or `decided_at`.**

Live-verified: a vehicle `in_service_bay` → guard denied and queued → co-pilot returned `hold`
(confidence 0.65) with a correct rationale (*"in a service bay… moving to DCFC disrupts service
work, benefit unclear at 64% SoC"*) and a risks list, with **status still `pending` and `decided_by`
NULL.**

> **Standing rule: the co-pilot ADVISES; the wall guard and the human ENFORCE.**

**Not yet built:** automatic co-pilot trigger on approval creation, and surfacing the recommendation
in the Pulse/OrchestrAV approvals UI. **Good work.**

### The recommended long-term shape for Nemotron

A 5-agent workshop concluded the **continuous dial-railing conductor should be retired** — it pins
rails while narrating the opposite, which is a diligence liability. Replace it with:
- **coarse, once-per-regime discrete posture selection** —
  `{daytime_churn | pre_wave_prestage | overnight_wave | morning_commit}` → a vetted preset;
- **exception / long-tail triage** (today's exception path is pure seeded-random dice — a real
  opportunity);
- **natural-language explainability** over the signed audit trail;
- **the approval co-pilot** (built).

**Training reality, stated honestly:** cuOpt is **formulation, not training** (hosted API, no GPU
buy). Nemotron is **context engineering plus a graded feedback loop now, fine-tuning later.**
⚠️ **The data-capture gap to close first:** the approval co-pilot writes its recommendation but
**does not store the human's accept/override verdict** — so there is no training signal yet. Build
the `(context → recommendation → verdict → outcome-N-ticks-later)` graded tuple log: few-shot memory
now, an SFT set later. It needs a few thousand **real** (not seeded-dice) tuples and an offline
replay-vs-historical-verdicts harness before any model touches a live run.

---

## 4. The frontier stack (`ottoq-intelligence`)

The Python service that hosts what cannot live in Postgres. FastAPI, deployed to EC2 (App Runner is
closed to new customers). Repo is **public** to avoid git-auth friction on the box.

| Layer | Endpoint | Engine | Status |
|---|---|---|---|
| **Optimize — Energy** | `POST /optimize/energy` | Rolling-horizon **MILP/MPC** (HiGHS): BESS + charge schedule minimising demand-charge ratchet + TOU + wear | **LIVE + tested** |
| **Forecast** | `POST /forecast` | Probabilistic arrivals/load/SoC (TFT / NHITS, GPU) | Stub — FR-2 |
| **Optimize — Assign** | `POST /assign` | cuOpt charger/bay/service assignment + job-shop | Stub — FR-3 |
| **Orchestrate** | `POST /orchestrate` | Nemotron conductor over the specialist optimizers | Stub — FR-4 |
| **Learn** | offline | CIL — offline RL / Bayesian tuning from run outcomes | FR-5 |

**Verified energy alpha** (`tests/test_energy_alpha.py`, realistic overnight wave, same battery):

```
peak no_bess   : 2020 kW   $43,996/mo
peak heuristic : 1250 kW   $27,225/mo   (the live cert_03 rule)
peak MPC       :  771 kW   $16,786/mo
MPC shave vs heuristic: 38.3%   ≈ $125k/yr additional alpha per depot
```

> ⚠️ **A same-day adversarial refutation you must know about:** *"a reserve-aware reactive controller
> matches the LP to the penny."* Branch `fr1b-energy-optimizer-certs`. The MPC's advantage over a
> **well-tuned** reactive baseline is much smaller than over the naive one. **Do not quote 38.3%
> against a straw baseline.**

> 🔑 **KEY ARCHITECTURAL FINDING: Supabase's `pg_net` CANNOT reach a raw EC2 box** (TCP/SSL handshake
> timeout — DB egress is HTTPS/edge-function only). **The fix is an edge-function bridge**
> (`ottoq-energy-mpc`, `verify_jwt=false` + `x-bridge-token`, Deno `fetch` → AWS). **This is THE way
> the twin calls any external compute** — the same path cuOpt and Nemotron use. If you ever need the
> database to reach an external service, build a bridge edge function.

**The async plan pattern:** `ottoq_energy_mpc_replan(...)` fires the request and inserts a `pending`
row in `ottoq_energy_plan`; `ottoq_energy_mpc_ingest(plan_id)` completes it from the landed pg_net
response. Edge-function cold start plus pg_net's batched worker gives 30–60 s latency —
fire-and-ingest tolerates it and matches out-of-band production replanning.
`ottoq_energy_orchestrate` follows `plan.bess_setpoint_kw[current_step]` only when the policy knob
`energy_mpc_follow >= 1` (**default 0**, so production is untouched).

⚠️ **The bridge token and AWS URL are hardcoded** in the edge function and in the replan defaults.
**Move them to Supabase secrets before any real productionisation.**

---

## 5. The frontier core capabilities (all built and proven)

`memory/project_frontier_core.md`

- **FC-0 — Policy-parameter substrate.** `ottoq_policy_params` (scope: run > depot > global),
  `ottoq_policy_get`, `ottoq_policy_param_catalog`. The keystone: the tunable levers MPC varies and
  CIL self-tunes.
- **FC-1 — Twin-in-the-loop MPC.** `ottoq_mpc_lookahead(run, plans, horizon, objective)`.
  **The mechanism is genuinely clever: fork the twin in a PL/pgSQL exception-block SAVEPOINT, apply
  a candidate policy, advance N ticks dry-run, measure, then RAISE to roll back — variables survive
  the rollback.** Verified byte-identical rollback across 4 forked branches.
  **This is the moat: no twin-less orchestrator can do closed-loop lookahead.**
  Dry-run runs set GUC `ottoq.dryrun='on'`; `ottoq_evaluate_rule_core` then *evaluates* the rule
  (safety projection intact) but returns before persisting, and `ottoq_cuopt_refresh` skips (no real
  NVIDIA calls inside forks).
- **FC-2 — Self-improving CIL.** `ottoq_cil_propose` (hill-climb around energy/deploy levers) +
  `ottoq_cil_tick` (A/B every candidate forward through the MPC, **0-unsafe hard gate**, adopt the
  strictly-better winner) + `ottoq_cil_adoptions`. Proven to compound 0.50 → 0.40 → 0.30 (predicted
  peak 1265 → 1015 → 765 kW), each 0-unsafe, correctly rejecting the worse candidate.
- **FC-3 — Uncertainty / risk calibration.** `ottoq_forecast_uncertainty` (0..1 from the coefficient
  of variation of recent EV load) sets the battery reserve floor = `soc_min_floor + uncertainty×20`.
  Proven: erratic ramp → uncertainty 1.00 → floor 10% → 30%; steady → 0.00 → 10%.
- **FC-4 — Agentic natural-language command.** `ottoq-ottocommand` (Claude tool-loop, role-scoped,
  shield-gated) with frontier tools: `get_status_brief`, `set_energy_aggressiveness` (clamped to the
  catalog's safe range), `evaluate_energy_options` (MPC on the active run), `self_improve` (CIL
  tick), `get_improvement_log`.

**The energy control-loop rebuild (the big one).** Building FC-1 surfaced that **OTTO-Q was
disconnected from the battery.** Five root causes: the battery ran an autonomous default and ignored
OTTO-Q's setpoint entirely; the one executor that could connect had **charge/discharge sign
reversed**; active charging was never throttled to the cap; OTTO-Q's own strategy drained the
battery early via an "arbitrage" branch so it was empty at the peak; and `energy_orchestrate` read a
run-agnostic stale snapshot. **All fixed.**

**Re-measured headline (clean on/off A/B, identical forked state at the evening peak, flagship
`service_max` = 2500 kW):** unmanaged baseline **1465 kW** → OTTO-Q default (0.50) **1252 kW
(−15%)** → OTTO-Q aggressive (0.20) **975 kW (−33%)**, all battery-driven, cars not delayed,
0-unsafe.

---

## 6. ⭐ Where the AI layer should go next — the forecast

`memory/project_arrival_forecast_learning_doctrine.md`

Chase's direction, 2026-08-05:

> *"With the arrival forecast, this will be a place where training and almost machine or variable
> learning will be extremely important. It will need to **first look at all variables, and then
> actual scenario or simulation variables, and deduce between the two** when activities or services
> should take place, and then **the actionable part of reserving that within the depot from a stall
> assignment and stall dwell-time perspective.** This could also be a great time to leverage our AI
> tools through NVIDIA or anything else to train agents."*

**The four-step shape:**
1. **All variables** — the vehicle's standing profile: cadences, wear, mileage, SoH, soil, software
   version, fault history, deployment commitments.
2. **Actual scenario / sim variables** — what is true in THIS run right now: current SoC, live ETA,
   real depot contention, staff and lane capacity, live faults, weather.
3. **Deduce between the two** — reconcile the standing profile against live conditions to decide
   **when** each service should happen.
4. **Then act** — *"the actionable part"*: **reserve it**, as a concrete **stall assignment with a
   dwell time**.

> **A forecast that does not end in a reservation is not orchestration.**

**Why this is the right home for the learned layer:** everything else in OTTO-Q is deterministic and
should stay that way. **Forecasting is the one genuinely uncertain part** — when will this car come
back, how long will the work really take, will the bay be free. That is exactly where a learned
model earns its place, and where a wrong answer is safe, because the deterministic layer still vets
the result.

### 🔴 What must exist FIRST — measured, and still true

**You cannot train a forecast on a constant.**

- **There is no arrival side at all.** `ottoq_vehicle_needs_card` has 74 columns and **not one**
  contains "arriv", "eta", or "return". It has `next_deploy_at` (departure) and no
  `next_arrival_at`.
- **The ETA is a hardcoded constant.** `ottoq_return_eta_minutes` is one line:
  `GREATEST(1, COALESCE(ottoq_policy_get(run,'return_eta_minutes',30), 30))`. No distance, no route,
  no traffic. **True forward visibility is a flat 30 minutes**, and only after the vehicle has
  already decided to come home.
- **It destructively overwrites the dispatch plan** — 109 of 112 rows had
  `scheduled_return_at = returning_started_at + exactly 30 min`, so **any plan-vs-actual accuracy
  check is circular by construction.**
- **The dispatch-time plan is useless as a prediction anyway:** 0 of 112 arrivals landed within
  5 minutes of it; median absolute error **68.4 min**, p90 215.5 min, mean bias **+92.4 min late**.
- **`ottoq_book_appointment` cannot reserve a bay.** Its stall search only queries `('dcfc','l2')`
  then `'staging'` — the strings `wash_bay` and `service_bay` **do not appear in it.** Measured:
  **0 of 110 needs had a bay reserved pre-arrival** against 43 bay-requiring atoms.
- **The outbound trip blocks the inbound plan.** `ottoq_plan_visit_itinerary` early-returns if an
  active itinerary with any `planned` leg exists; a deployed vehicle still owns its outbound
  itinerary with an un-executed `depart` leg, so the en-route planning call returns 0 **every time.**
  Measured: 0 of 100 itineraries built more than 1 minute before arrival; median lead **+0.05 min** —
  at the curb.
- **The twin's wear model is half-built.** Only `drive_km_total`, `drive_hours_total`, `soil_index`,
  and `open_dtc_count` accumulate. **Tire tread, brake wear, battery SoH, sensor health, software
  version, and odometer are drawn once at boot and frozen — a tire never wears.** Those are exactly
  the dimensions the founder's named services depend on.

### The recommended build order (before any model is trained)

1. **Give the needs card an arrival side** — `next_arrival_at`, `minutes_to_arrival`,
   `arrival_confidence`. Join `ottoq_approach_band`. **Today no function in any schema joins those
   two objects.**
2. **Stop the ETA overwriting the plan.** Write the refreshed ETA to its own column so plan-vs-actual
   stops being circular and a training signal exists at all.
3. **Make the twin's wear model complete and stateful**, so there is something real to learn from.
4. **Teach `ottoq_book_appointment` to reserve bays**, and unblock the pre-arrival planning path.
5. **THEN learn** — predict arrival time and true dwell/service duration from history, and feed
   cuOpt and Nemotron the reconciled picture. ⚠️ **Nemotron reviews, never decides.**

Steps 1–4 create the labels. Step 5 is meaningless without them.

---

## 7. Future NVIDIA tooling worth pulling in

Validated against primary sources during the build; treat as a menu, not a commitment.

| Tool | What it buys OTTOYARD | Notes |
|---|---|---|
| **cuOpt** (hosted) | Overnight-wave scheduling under charger scarcity; VRP for multi-depot | **In use.** Re-point from per-tick LAP to the wave. |
| **cuOpt self-hosted** | No per-tick API spend, lower latency, **on-prem / air-gapped operation** | Matters for the OEM deployment bar. Revisit once enactment is proven. |
| **cuOpt as a CUDA-X AI-agent skill** | GTC Taipei (Jun 2026) made cuOpt + CUDA-X available as agent skills — **validates our "agent proposes, deterministic solver + shield disposes" pattern 1:1** | Positioning gold. |
| **Nemotron 3** (Nano / Super / Ultra) | Triage, regime selection, explainability, approval co-pilot | Ultra in use. Nano/Super are cheaper tiers for high-frequency advisory work. |
| **NeMo Agent Toolkit** | Multi-agent orchestration scaffolding | Not adopted. |
| **NeMo Guardrails** | Wraps LLM natural language only — **not** a substitute for the deterministic shield | Explicitly scoped: the shield stays separately authored. |
| **Isaac Sim / Omniverse** | The photoreal Tier-B twin | **In use.** Validated by **Cyngn × NVIDIA Isaac Sim (Feb 2026)** running AV fleet management in a warehouse twin with cuOpt-for-Isaac — OTTO-TWIN's exact thesis. |
| **cuOpt-for-Isaac** | Physics-AI motion + optimization inside the photoreal twin | The natural next step once Tier-B carries live motion. |
| **NeuralForecast / Nixtla TFT + quantile nets** | Probabilistic arrival/load/SoC forecasting | The FR-2 plan of record. Open source, not NVIDIA. |
| **OR-Tools CP-SAT via PyJobShop** | The charge → wash → service → stage flow-shop | **Benchmark it against cuOpt.** If CP-SAT wins on the bench, ship CP-SAT and keep cuOpt for the sub-structure — unless NVIDIA-in-the-loop is a positioning requirement. **That is an open founder question.** |
| **NVIDIA Alpamayo** (CES Jan 2026) | The *car's* brain — the far end of our comms layer | **Not** our comms layer. Do not conflate. |

**AWS:** ~$100k NVIDIA Inception credits are available and partially used (the g6e box and the
intelligence service). ~$10k of separate AWS credits are reserved for **Phase 2: IoT Core MQTT +
command API**.

---

## 8. Honest positioning language

Use this. It survived adversarial review; throughput multiples did not.

> *"OTTO-Q handles your entire returning fleet against every constraint — energy, stalls,
> wash/detail/service, deadlines — on materially less charging hardware, with zero unsafe actions
> and a signed, replayable audit trail."*

**Ranked moats:**
1. **Deterministic safety guarantee + signed audit trail** — the #1 durable moat. Regulators, OEMs,
   and insurers need it.
2. **Capital efficiency under scarcity** — the wave scheduler. **Quote magnitude only after the CRN
   certification.**
3. **OEM-agnostic appointment PROTOCOL** — "OCPP for AV depot workflow." A network-effect,
   operating-system moat.
4. **Joint co-optimisation of the whole service graph under one shield** — energy is table stakes;
   lead with energy **in context**.

**AI positioned truthfully:** cuOpt = a shield-validated batch planner. Nemotron = an advisory
co-pilot with a measurable accept rate and explainability. **Never claim cuOpt beats our greedy
until the A/B has actually run** — it has not.
