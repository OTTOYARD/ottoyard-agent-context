---
name: project_nvidia_layer_truth_2026_07_30
description: "🚨AUDIT 2026-07-30: cuOpt FIRES but 0 of its proposals were EVER enacted (0 of 84k decisions); Nemotron-3-Ultra DOES write live but Math.round destroys its numbers — 106 of 109 dial writes land on the extreme OPPOSITE of what it asked. OTTOCOMMAND's language model is Claude, not Nemotron."
metadata: 
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-01T17:43:56.338Z
---

> ## ⛔ 2026-08-01 CORRECTION — READ THIS BEFORE THE REST OF THIS FILE
> **The "cuOpt is ASYNC ⇒ proposals arrive stale ⇒ only LP re-posing fixes it" root cause below is
> FALSIFIED.** A 10-agent workflow (`wf_c3189056-9de`) with a **live probe** settled it empirically:
> - **cuOpt's hosted LP is SYNCHRONOUS. 12/12 calls returned HTTP 200 with `nvcf-status: fulfilled`
>   and a complete inline solution — ZERO 202s**, at 1 var, at production scale (40×45 = 1,800 vars,
>   547-721 ms, 6/6), and at 4,800 vars (1,396 ms; solver itself **18 ms**, 76 simplex iterations).
>   There is no size in our reachable range where it goes async.
> - **Our LP payload is CORRECT** — accepted and solved to proven optimality in every variant.
>   Omitting `variable_types`/`row_types`/`client_version` is fine; the LP relaxation came back
>   **integral** (value set exactly {0.0, 1.0}) at 4,800 vars, vindicating the total-unimodularity
>   argument. Empty CSR rows (an inlet matching nothing) are accepted, not an error.
> - **v20's parse path is CORRECT**: `response.solver_response.solution.primal_solution`. Three rival
>   paths were tested and all returned nothing.
> - ⚠️ **There is NO top-level `reqId` on a 200** — top-level keys are exactly `["response","notes"]`.
>   The request id exists ONLY as the `nvcf-reqid` **response header**. Any fix keying off
>   `result.reqId` reads undefined.
>
> **THE REAL ROOT CAUSE: cuOpt is asked LAST, and usually not asked at all.**
> 1. `ottoq_cuopt_refresh` uses **pg_net**, which only dispatches **after COMMIT**. It is called from
>    `ottoq_sim_decide_and_dispatch:36` — the *same transaction* as `decide_tick:45`. That call site
>    **cannot ever work, by construction**: the HTTP request has not been sent when the decider reads
>    the seam.
> 2. By the time cuOpt is asked, OTTO-Q's own scheduler has already parked the vehicles and locked
>    the stalls, so the edge function hits its early return
>    `if ((cands.length===0 && enrouteCands.length===0) || freeStalls.length===0) return json({source:"none"})`
>    — which sits **BEFORE** `Deno.env.get("NVIDIA_API_KEY_CUOPT")`. **NVIDIA is never contacted.**
>    MEASURED on run 9ea73866, 7 fires: free stalls 39→35→21→8→3→2→0 vs candidates 0→0→0→0→0→0→11.
>    **Never both non-zero.**
> 3. Even a landed proposal is structurally refused: `ottoq_honour_reservation_proposal` **Gate B**
>    (`IF v_resv_stall IS NOT NULL THEN … RETURN 'reservation_honoured'`) returns before the external
>    seam is read at L53-55. **54 of 71 stall_assignments (76%) exited there.**
> 4. cuOpt DID solve **11 times, optimally, in 10-34 ms, producing 169 proposals** on 2026-08-01
>    16:37-16:38 (verbatim in `net._http_response` ids 64563-64586). Those rows are gone because
>    **`ottoq_purge_prior_runs` sweeps `ottoq_external_proposals`** when the next run starts — which
>    is why the table's `max(created_at)` reads 2026-07-23 despite solves today. **Artifact, not
>    contradiction. Capture cuOpt evidence DURING a run or it is destroyed.**
>
> **⇒ The LP rewrite was not wasted (it is correct and fast) but it was never the fix. The fix is
> ORDERING.** Two options: a *blocking* `extensions.http` call (pgsql-http v1.6 **is installed**)
> moved ahead of assignment, or the safer **split tick** (beat N fires + commits, beat N+1 decides).
> ⚠️ Blocking HTTP inside the tick transaction risks a second instance of the
> [[project_legtype_abort_root_cause]] pattern — it MUST have a ≤2.5 s timeout and swallow all errors.
> ⚠️ **Gate B override needs Chase's explicit doctrine sign-off** — overriding a reservation a vehicle
> already holds is exactly the [[project_indepot_reassignment_gate]] case requiring tech approval.
> ⚠️ **cuOpt has NEVER been enacted, so we have NEVER measured whether it beats our greedy.** The
> −41% grid-peak headline is OTTO-Q's own logic and does not depend on cuOpt. Claim nothing until the
> A/B runs.

**5-agent audit + web research, workflow `wf_2b03e82a-981`. Every number below is measured from the live
DB, not inferred.** Supersedes the optimistic framing in [[reference_nvidia_ai_integration]] and
[[project_ai_layer_production]] ("cuOpt FIRES+DRIVES", "Nemotron auto-actions 6 dials").

---

## 🚨 cuOpt — FIRES, GENUINELY ANSWERS, AND IS **NEVER USED**
- **NVIDIA really does respond.** 12 rows in `ottoq_cuopt_fire_log`; HTTP **200**s in `net._http_response`;
  **18 rows** in `ottoq_external_proposals` with `source='cuopt'` (that literal is set only on a *successful*
  NVIDIA fetch — a failure downgrades to `cuopt_fallback`). Seam B reached NVIDIA too:
  `"charge_vehicles":10,"charge_assigned":10,"charge_source":"cuopt"`. **The API key works.**
- **⭐ CONSUMPTION IS ZERO.** `ottoq_external_proposals` by source/status: **cuopt/expired 17,
  cuopt/superseded 1 — never once `enacted`.** The local heuristic `greedy_constrained` enacted **14**.
  `ottoq_decisions` by `proposed_action->>'source'` over **84k+ rows**: `greedy_constrained` 15,
  `reservation_honoured` 10, `reservation_reassigned` 4, **`cuopt` = 0.** Not one enacted, overridden or deferred.
- **`ottoq_recommendations`: 83,812 rows, `executed_at IS NOT NULL` = 0.** Nobody has ever executed one.
- **Seam B's "enacted" label is FALSE.** `ottoq_shield_and_log` contains no UPDATE of vehicles/stalls — it
  INSERTs a decision row and emits a recommendation. 55,016 stall_assignment + 27,282 task_start rows carry
  `outcome_status='enacted'` with `sim_clock IS NULL`. Nothing was enacted.
- **⭐⭐ ROOT CAUSE — A SIM-CLOCK vs REAL-CLOCK DOMAIN MISMATCH.** `ottoq_sim_advance_tick_world:87` sets
  `ottoq_ocpp_chargers.last_heartbeat_at = v_new_sim_clock` (**sim** domain). The edge fn
  `ottoq-cuopt-propose` filters healthy chargers with `.gte("last_heartbeat_at", Date.now() - 90_000)`
  (**real** domain). A run whose sim clock sits hours from now() therefore sees **every charger stale** ⇒
  `healthy` empty ⇒ all 45 charge stalls dropped ⇒ body `{"proposed":0,…,"stalls":0,"source":"none"}` ⇒
  **early return, NVIDIA never called.** Verified now: **0 of 90 chargers** have a heartbeat within 90 s.
  ⇒ **This bug HEALS as sim clock approaches real clock — i.e. 1:1 `live` mode partly fixes cuOpt.**
- **⭐ THE 4-SECOND BLOCK CANNOT EVER SUCCEED.** The metronome blocks `LEAST(6.0, cuopt_solve_window_ms/1000)`
  = **4 s by default**, but NVIDIA's own `solver_config.time_limit = 5 s`. **The window is shorter than the
  solve.** So the world clock freezes 4-6× the cost of the entire rest of the tick (twin ~914 ms + decide
  ~181 ms) and then gives up before an answer could arrive. Pure cost, zero benefit, by construction.
- **A second timing defect:** `ottoq_sim_decide_and_dispatch` calls `ottoq_cuopt_refresh` in the **same
  transaction** that then runs `decide_tick`. **pg_net only dispatches after COMMIT**, so on that path the
  answer can never arrive for the tick that asked. Earliest possible use is the NEXT tick.
- **Observed miss:** run `d9216f30` — 17 decisions at 12:36:17 (8 short-circuited to `reservation_honoured`,
  9 `noop_no_candidate`); cuOpt's proposals for those very vehicles landed **85 s later**; no further tick; all expired.
- **DEAD EDGE FUNCTIONS, ZERO CALLERS:** `ottoq-assign-optimize` (**contains a real cuOpt call**),
  `ottoq-sequence-optimize`, `ottoq-energy-optimize`. Only `ottoq-orchestrate-tick` is referenced (by `ottoq_cron_tick`).

## 🚨 Nemotron — FIRES AND IS CONSUMED, BUT ITS REASONING IS THROWN AWAY
- **Model confirmed: `nvidia/nemotron-3-ultra-550b-a55b`.** Chase's "Nemotron 3 Ultra" is real and is what's wired.
- **It writes to the live brain:** **462 run-scoped rows** in `ottoq_policy_params` stamped
  `updated_by='ottoq_prime'` across **123 runs**, 2026-07-15 → 2026-07-30. Live HTTP 200s on 2026-07-30 22:58-22:59.
- **⭐⭐ THE DEFECT — `Math.round` COLLAPSES EVERY FRACTIONAL DIAL TO AN EXTREME.** `ottoq-orchestrator-agent`
  v15: `if (cd.hi - cd.lo <= 1) { v = Math.min(cd.hi, Math.max(cd.lo, Math.round(Number(a.value)))) } else { /* ±30% MAX_DRIFT */ }`.
  **5 of the 6 dials have range width ≤ 1**, so they take the `Math.round` branch — meaning **the ±30% drift
  limiter, the actual guardrail, never runs on them.** A fractional request rounds to 0 or 1, then clamps to
  floor or ceiling.
  **Paired proposed-vs-applied proof:** asked `deploy_peak_fraction 0.7` → applied **1.0** (opposite extreme);
  asked `0.8` → **1**; asked `0.85` → **1**; asked `energy_demand_factor_peak 0.65` → **0.9**; asked `0.35` → **0.3**.
  **Distribution:** `deploy_peak_fraction` **106/109 = 1.00**; `energy_demand_factor_peak` 107 = 0.9 (117/121 at an
  endpoint); `energy_demand_factor_expensive` 73 = 0.20 (80/81 at an endpoint); `energy_reserve_shave` **98/98 = 1**.
  ⇒ **We pay for 550B-parameter reasoning and enact a coin flip between "off" and "all the way on."**
- **⚠️ IT POINTS AT THE ENERGY HEADLINE.** `energy_demand_factor_peak` feeds `ottoq_energy_orchestrate:63`
  `v_demand_target := v_service_max * energy_demand_factor_peak` — **higher = LESS shaving**, and it is pinned
  to the ceiling 88% of the time. `deploy_peak_fraction` feeds `ottoq_decide_tick`,
  `twin.ottoq_sim_auto_dispatch_tick`, `twin.ottoq_sim_advance_service_flow`.
  ⚠️ Caveat flagged by the auditor and **not yet run to ground**: when `energy_reserve_shave=1` (which Nemotron
  also sets, 98/98) line 67 may override the demand target. **Verify before re-quoting the −41% grid-peak number**
  ([[reference_ottoq_real_edge]]).
- **Ratchets that are NOT the model:** `deploy_surge_catchup` 0.455/0.5915/0.76895 = ×1.30 each time, and
  `forecast_horizon_min` 45/60/75 = +15 — both come from `ottoq_apply_ops_action`'s SQL, not from Nemotron.
- **The approval half has NEVER fired.** `ottoq_ops_approvals` = **0 rows, ever**; every audited call returns
  `queued: []`, `rejected: []`. The routing code exists but is unreachable — the system prompt hands the model
  the legal names, so it never emits an illegal one.
- **SAFETY GATE, honestly:** name whitelist (6 params) = **real containment** — Nemotron cannot move a vehicle,
  open a stall or dispatch. Range clamp = real. Move cap 3/call = real. **±30% drift limiter = DEAD on 5 of 6
  dials.** **The 52-rule L1 shield is NOT in this path** — a dial change isn't a physical effect, so the shield
  never sees it. ⇒ Chase's "suggestions pass through a deterministic safety gate" is **true for cuOpt's
  *physical* proposals, but NOT for Nemotron's policy writes.**

## ⭐ OTTOCOMMAND — the language model is CLAUDE, not Nemotron
`ottoq-ottocommand/index.ts` calls **`api.anthropic.com`, `claude-sonnet-4-6`, 16 tools**. Nemotron's only
natural-language job is `ottoq-nemotron-copilot`, an after-action report behind one button in
`TwinCopilotTab.tsx`. **So: language = Claude; Nemotron = silent dial-turner. The exact inverse of Chase's
mental model.** Both UIs do share one brain-side agent — PULSE `ottoyard-field-ops/src/hooks/use-ottocommand.ts`
(`role:'technician'`), OrchestraAV `ottoyard-OTTO-Q/src/components/OttoCommand/OttoCommandPanel.tsx:284`
(`role:'manager'`).
**⚠️ SECURITY:** role separation exists **only as advice in the system prompt** ("a manager REQUESTS; a
technician EXECUTES"). **No server-side permission check** — a manager session can call every tool a
technician can.

---

## 🚀 SHIPPED 2026-07-31/08-01 — cuOpt now genuinely produces proposals (half the fix)

**1. Removed the blocking solve window** (migration `cuopt_fire_and_forget_remove_blocking_solve_window`).
The metronome now FIRES cuOpt and moves on; the proposal is consumed by a later tick via the existing
L2 seam. Verified: `still_polls_cuopt=false, still_has_deadline=false, still_fires_cuopt=true`.
**⚠️ CORRECTION TO THE RESEARCH'S ADVICE:** it recommended "set `cuopt_solve_window_ms = 0`, a one-line
policy change." **That would have DISABLED cuOpt entirely** — metronome line 78's `IF v_window_ms > 0`
wraps the `ottoq_cuopt_refresh` call **as well as** the poll loop. Verified against source before acting.
The knob now means ENABLE (>0), not "how long to wait."

**2. Fixed the clock-domain bug** (`ottoq-cuopt-propose` **v19**). One line:
`new Date(Date.now() - 90_000)` → `new Date(new Date(clock).getTime() - 90_000)`, where `clock` is the
run's `sim_clock_current` (already used two lines below for reservation expiry). `production_live` sets
`sim_clock_current = now()`, so it is correct on both paths. Also added `chargers_healthy`,
`heartbeat_since`, `sim_clock` to the response for observability.

**MEASURED BEFORE → AFTER (run `d3cc7ecf`, seed 700011):**
| | before | after |
|---|---|---|
| chargers passing freshness | **0 of 90** | **45** |
| edge-fn response | `{"stalls":0,"source":"none"}` | `{"proposed":2,"candidates":2,"stalls":45,"source":"cuopt","cuopt_error":null}` |
| cuOpt proposals created | 0 for the run | **43 over 6 ticks** |
⇒ **NVIDIA is now actually being called and returning real assignments.**

**⚠️ HALF THE PROBLEM REMAINS — CONSUMPTION IS STILL ZERO.** `cuopt_enacted = 0`,
`decisions_from_cuopt = 0` (of 5 decisions over 6 ticks). Proposals sit `pending` then go `superseded`.
**Do not claim cuOpt "drives decisions" yet — it produces, nothing consumes.** Next: trace
`ottoq_l2_external_proposal` ← `ottoq_honour_reservation_proposal` ← `ottoq_decide_tick:155` and find
why a `pending` cuopt proposal for a gate vehicle is not picked up. Caveat on this measurement: the
world was near-idle (2 gate candidates, 5 decisions), so absence of enactment is **suggestive, not
conclusive** — re-test under a primed fleet before concluding the consumer is broken.

## 🚀 SHIPPED — `ottoq-orchestrator-agent` **v16**: Nemotron's numbers are no longer replaced
**THE BUG, precisely:** v15 used `if (cd.hi - cd.lo <= 1) { Math.round(value) }`. That test was meant
to ask *"is this a whole-number dial?"* but it measures **RANGE WIDTH** — and 4 of the 6 dials are
FRACTIONS with narrow ranges: `deploy_peak_fraction` 0.5-1.0 (w=0.5), `energy_demand_factor_peak`
0.3-0.9 (0.6), `energy_demand_factor_expensive` 0.2-0.8 (0.6), `deploy_surge_catchup` 0.1-1.0 (0.9).
`Math.round(0.7)=1` → clamp → **1.0, the ceiling.** And because those dials took the `if` branch, the
±30% `MAX_DRIFT` limiter (living in the `else`) **never ran on them.** Only `forecast_horizon_min`
(w=80) reached the limiter; only `energy_reserve_shave` (0|1) legitimately wanted rounding.

**FIX (v16):** integer-ness is now an **explicit per-dial `int: true` flag** (`forecast_horizon_min`,
`energy_reserve_shave` only). New `clampDial()` applies **drift-limit → range-clamp → round-only-if-int**,
so the limiter now runs on **every continuous dial**. Binary switches skip the drift limiter (a `current`
of 0 would otherwise pin them at 0 forever — real edge case, guarded). The audit row now records BOTH
`requested` (the model's number) and `limited_by` (`none|drift|range|drift+range`), so a future audit can
see at a glance whether the model's judgement survived. **Also added real `propose_latency_ms` /
`total_latency_ms`** (were hardcoded 0, so nobody could tell if it fits the 0.5-1 s window). System prompt
now tells the model fractional dials accept fractional values.

**⚠️ DEPLOYED BUT NOT YET OBSERVED LIVE.** Run `b524b9d6` (seed 700012) reached 5 ticks with
**0 orchestrator calls** — the gate is `ottoq_orchestrator_trigger(depot_id)` inside
`ottoq_sim_decide_and_dispatch`, and a near-idle world does not trip it. **The fix is verified by code
reading, NOT by a live paired proposed-vs-applied row. Do not claim it works until one is observed** —
re-test under a primed/loaded fleet and confirm a fractional request lands as a fractional value with
`limited_by` populated.

## 🎯 2026-08-01 — THE REAL BLOCKER IS UPSTREAM OF cuOpt ENTIRELY. START HERE NEXT SESSION.

**❌ MY "ORDERING BUG INSIDE decide_tick" HYPOTHESIS WAS WRONG — there is no ordering bug.** Read the
source: `ottoq_decide_tick:155` calls `ottoq_honour_reservation_proposal` (which reads the L2 external
proposal seam) **BEFORE** any local assignment, and **`:198 v_action := v_proposal`** — the decider
already *prefers* cuOpt's answer and enacts it at `:199-203`. The plumbing was always correct.

**SHIPPED ANYWAY (and it worked, migration `cuopt_fire_after_decide_so_answer_waits_for_next_tick`):**
moved the cuOpt fire from BEFORE `decide_and_dispatch` to AFTER it, so the answer lands during the
inter-tick pacing gap (2-6 s) instead of racing a ~181 ms decide. **Result: pending cuOpt proposals went
from ~0 to 55, then 24 held steady against 24 gate vehicles** — proposals are now genuinely waiting when
the next tick starts. Bonus: firing post-decision means cuOpt only proposes for vehicles still unserved.

**🚨 BUT ENACTMENT IS STILL 0, AND THE MEASURED REASON IS THAT `decide_tick` NEVER RUNS ITS
STALL-ASSIGNMENT LOOP.** Run `1f8d764d` (seed 700015), 6 ticks, **24 vehicles at `arrived_at_gate`
with SoC<85, 24 cuOpt proposals pending** — and `ottoq_decisions` contained **ZERO `stall_assignment`
rows**. All 9 rows were `task_start` from the Nemotron orchestrator agent (a different edge function).
**Not even the `noop_no_candidate` rows the loop writes at `:158-159` when it abstains.** Zero decisions
of ANY kind from decide_tick ⇒ **the cursor returned no rows, or decide_tick was never reached.**

**RULED OUT ALREADY (do not re-test):**
- The `charging_staff` LIMIT at `:143-151` — measured `twin.ottoq_sim_lane_capacity(run,'charging_staff',45) = 45`,
  and `45 >= 45` takes the `2147483647` branch ⇒ **unbounded**. `now_charging = 0`.
- The cuOpt seam, the clock domains, and the LP solver — all verified working this session.

**PRIME SUSPECT: `ottoq_sim_decide_and_dispatch` is throwing before it reaches `ottoq_decide_tick`, and
the metronome swallows it** — it wraps the call in `EXCEPTION WHEN OTHERS THEN RAISE WARNING 'metronome
decide % failed: %'`. A `RAISE WARNING` does NOT appear in `cron.job_run_details` (it shows `succeeded`),
so this failure mode is invisible everywhere except the Postgres log.
**FIRST ACTION NEXT SESSION: `get_logs(postgres)` and grep for `metronome decide`.** If that is silent,
call `public.ottoq_sim_decide_and_dispatch('<run>')` directly in a rolled-back `DO` block with a live run
and read the error. Second suspect: the cursor's `AND v.current_soc < COALESCE(visit_needs.target_soc, 85)`
— an open `ottoq_visit_needs` row with a LOW `target_soc` would exclude a vehicle my `< 85` check counted.

⇒ **Everything cuOpt-side is now proven working. The remaining gap is a decide_tick execution failure that
has been masked by a swallowed exception — which is why it never showed up as an error anywhere.**

## 🚨 DOCS-vs-REALITY: NVIDIA's PUBLISHED SPEC SAYS LP IS UNAVAILABLE. IT IS AVAILABLE. TRUST THE PROBE.
A 3-agent web-research pass concluded **`hosted_lp_available: NO`**, citing NVIDIA's own OpenAPI at
`docs.api.nvidia.com/nim/reference/nvidia-cuopt` — whose `action` enum contains ONLY
`cuOpt_OptimizedRouting` / `cuOpt_RoutingValidator` — plus a third-party post claiming `cuOpt_LP`
returns *422 "Linear Programming is disabled"*. It recommended self-hosting `cuopt-server` on GPU.

**THAT IS WRONG, AND THE LIVE SERVICE PROVES IT.** Measured twice on 2026-08-01 against
`optimize.api.nvidia.com/v1/nvidia/cuopt` with our existing key:
1. the endpoint's OWN 422 validator lists `cuOpt_LP` among the legal actions;
2. a `cuOpt_LP` probe returned **HTTP 200**, `problem_category:"LP"`, `primal_solution:[0,1]`
   (the correct optimum), `solver_time` 0.00095 s;
3. production v20 returned `status:"Optimal"`, `solve_ms:70.14` with real assignments.

**WHY THE DOCS ARE WRONG: the published spec is pinned at `"NVIDIA API for cuOpt-24.03"` while the
self-hosted spec is at 26.06 — the managed reference page is ~2 releases stale.** The research
surfaced that fact and still concluded NO.
**⭐ LESSON (this is the third time today the empirical check beat the plausible answer): for a live
API, a probe outranks documentation, a changelog and a blog post combined. Had I trusted the research
I would have told Chase to stand up GPU infrastructure he does not need.**

**⚠️ ONE REAL RISK THE RESEARCH DID SURFACE — a 200 KB request-body ceiling.** Bodies over 200 KB
require the NVCF asset path (`NVCF-INPUT-ASSET-REFERENCES` header + presigned S3 upload + `"data": null`).
Our worst case is 40 vehicles × 120 stalls = 4,800 variables / ~9,600 nonzeros ⇒ roughly **110-130 KB**
of JSON. Under the limit, but **not by much**, and it grows with the fleet. If the candidate cap or
stall count ever rises, this becomes a silent failure mode — the solve would start erroring and the
code would fall back to the greedy heuristic while still reporting a nominal path.
**Mitigations, in order of preference:** keep the candidate cap; drop `variable_bounds` (defaults are
[0, inf], so only `upper_bounds` is strictly needed); shorten/omit `variable_names`; or adopt the
asset-reference path.
**Other genuinely useful options the research confirmed (not yet adopted):** `solver_config.method`
(e.g. `"Dual Simplex"`), `infeasibility_detection: true`, `variable_names` (makes the response's `vars`
map human-readable for the "why"), and `warmstartId` for incremental re-solve — the last one is the
real enabler for a continuous "constant sorter". Self-hosted `cuopt-server` (POST `/cuopt/request`,
GET `/cuopt/solution/{id}`, port 5000) remains the path to **MILP** if genuine either/or logic is ever
needed; the hosted plane is LP + routing.

## ⭐⭐ SHIPPED 2026-08-01 — cuOpt **LP** is live (v20). 5 s → **70 ms**, provably optimal, self-explaining.

**⭐ THE HOSTED ENDPOINT DOES SUPPORT LP — verified EMPIRICALLY, not from docs.** I deployed a throwaway
probe (`ottoq-cuopt-lp-probe`, since deleted) that POSTed candidate actions to the endpoint we already
use. The **422 error was the gift** — NVIDIA's own validator enumerated the legal actions:
`'cuOpt_OptimizedRouting','cuOpt_RoutingValidator','cuOpt_LP','cuOpt_LPValidator','cuOpt_Solver','cuOpt_Validator'`.
A `cuOpt_LP` probe then returned `{"status":"Optimal","primal_solution":[0,1],"dual_solution":[1],
"solver_time":0.00095}` in **206 ms round-trip**.
**⭐ REUSABLE TECHNIQUE: to learn an unknown API's contract, send a deliberately WRONG enum value — the
validation error often returns the complete legal set.** Far faster and more reliable than doc-hunting.

**VERIFIED LP PAYLOAD (`action: "cuOpt_LP"`, same endpoint/auth as routing):**
```
data: { csr_constraint_matrix:{offsets,indices,values},
        constraint_bounds:{lower_bounds,upper_bounds},
        objective_data:{coefficients,scalability_factor,offset},
        variable_bounds:{lower_bounds,upper_bounds},
        maximize:false, solver_config:{time_limit} }
```
Response: `response.solver_response.status` (`"Optimal"`) and `.solution.{primal_solution,
dual_solution, reduced_cost, solver_time, lp_statistics, milp_statistics}`.

**THE MODEL (v20 `cuoptAssign`):** one 0/1 var per **compatible** (vehicle, stall) pair — incompatible
pairs get **no variable at all**, so they are structurally unselectable (v16 used `BIG=1,000,000`, which
a solver may still legally pick when everything else is worse). Rows `0..n-1` = "vehicle used ≤ once",
rows `n..n+m-1` = "stall used ≤ once", all bounds `[0,1]`. **The assignment polytope is totally
unimodular ⇒ the LP optimum is already integral — no MILP needed.** Objective `cost − R` with
`R = maxCost+1` so every assignment has a negative coefficient ⇒ assign as MANY as possible,
cheapest-first. `pairCost()` preserves the exact v16 policy.

**MEASURED LIVE (run `a4ea97c6`, seed 700014):**
`"solver":{"kind":"LP","status":"Optimal","solve_ms":70.14,"binding":{"stall_type":"dcfc","shadow_price":1}}`
⇒ **provably optimal in 70 ms** (was a 5 s metaheuristic that always burned its full time limit), and the
**dual values immediately identified DCFC scarcity as the binding constraint** — the "why" the cockpit
and OEM reviews need, delivered on the first solve.

**⚠️ BUT CONSUMPTION IS STILL 0 — speed alone did NOT beat the race, and here is why.**
`ottoq_cuopt_refresh` uses **pg_net, which only dispatches AFTER COMMIT**. The metronome does
`refresh → COMMIT → decide_and_dispatch`, so the HTTP request does not even leave until the commit, then
races a `decide` that takes ~181 ms. A 70 ms *solve* still loses because edge-function + network overhead
sits on top. And the tick-N proposal is still unusable at tick N+1 because tick N's local greedy already
reserved those stalls. **⇒ The remaining blocker is ORDERING INSIDE `ottoq_decide_tick`, not solver speed.**
The fix is to make `decide_tick` consult a pending cuOpt proposal **before** its local greedy path claims
inventory (or have the proposal carry the reservation). That is surgery on the 29k-char `decide_tick` and
should be done with the same review discipline as the rule-1 inversion — see task #13's findings, especially
the age-cap lesson (an unbounded in-flight guard can freeze vehicles permanently).

## 🎯 ROOT CAUSE OF ZERO CONSUMPTION — FOUND 2026-08-01. It is a RACE, not a clock bug.
Traced by replicating each guard of `ottoq_l2_external_proposal` against a live pending proposal:
```
stall_empty=true   charger_ok=true   reservation_ok=FALSE   vehicle_state=arrived_at_gate
```
**All 3 pending cuOpt proposals targeted stalls ALREADY RESERVED BY A DIFFERENT VEHICLE**, each
reservation live for another ~40 sim-minutes (`reserved_by` ≠ proposed vehicle, `expires 03:15` vs
`sim_now 02:35`). So the consumer is behaving **correctly** — it refuses to hand a vehicle a stall
someone else holds. cuOpt's answer is simply **stale on arrival**.

**WHY:** `ottoq_cuopt_refresh` fires, and in the SAME tick `ottoq_decide_tick` runs its local
`greedy_constrained` path and **reserves those very stalls first**. pg_net only dispatches after
COMMIT, so cuOpt's proposal cannot land before the local decider has already claimed the inventory.
By the time the proposal exists, every stall it names is taken. **This is the structural reason
cuOpt has 0 enactments across 84k decisions — not the clock-domain bugs (those were real and are
fixed, but they were masking this).**

**⇒ THE FIX IS NOT ANOTHER PATCH — it is the re-posing already recommended in the strategic read:**
1. **Reserve-then-propose, or propose-against-a-reservation-free view.** cuOpt must either be given
   the inventory the local decider has *not* claimed, or its proposals must carry the claim.
2. **Better: make cuOpt the assigner rather than a second opinion.** With the LP re-posing
   (sub-second, provably optimal) cuOpt returns *inside* the tick, so there is no window for the
   local heuristic to race it. The heuristic becomes the FALLBACK it was always meant to be, not
   the incumbent that always wins.
3. Interim mitigation if needed: have `decide_tick` skip local reservation for vehicles with a
   `pending` cuopt proposal younger than N seconds (an in-flight guard) — but note the age-cap
   lesson from the rule-1 review: without a cap, a stalled cuOpt would freeze those vehicles forever.

**STATUS OF THE TWO CLOCK FIXES (both real, both verified, both necessary-but-insufficient):**
`ottoq-cuopt-propose` v19 — chargers seen 0→45, proposals 0→43, `source:"cuopt"`, no error.
`ottoq_l2_external_proposal` — `charger_ok` now returns TRUE (was FALSE for every stall). Confirmed
by the trace above: that guard passes now; only `reservation_ok` fails.

---

## ⭐⭐ STRATEGIC READ (web research, same workflow) — BOTH TOOLS ARE POINTED AT THE WRONG JOB

**The symmetry: cuOpt has the right capability aimed at the wrong problem SHAPE; Nemotron is the right
model family aimed at the wrong TASK.**

### cuOpt — we use the routing solver for an assignment problem, and never send it the routing problem
- **Stall assignment is min-cost BIPARTITE MATCHING = an LP.** cuOpt LP (GA) solves it to **provable
  optimality sub-second** (ref: 69k constraints / 17k vars in <0.3 s on H100) and, because assignment is
  totally unimodular, the LP answer is already integral. **We instead send it to cuOpt Routing (VRP), an
  anytime metaheuristic that BURNS ITS FULL TIME LIMIT, ALWAYS — that is the 5 s.**
  ⇒ **Switching to LP is what makes Chase's "constant sorter" physically possible** (terminates on
  optimality, not on a clock).
- **FREE EXPLANATION LAYER: LP returns DUAL VALUES** — *which constraint bound me* (energy cap, connector
  mismatch, DCFC scarcity). That is exactly the "why" the cockpit and any OEM demo needs, and we get it for nothing.
- **We fake hard compatibility with `BIG = 1,000,000` cost penalties** — a solver may still legally pick
  them if every alternative is worse. Real constraints (`order_vehicle_match`) are unused.
- **We pre-truncate `cands.slice(0, freeStalls.length)`**, discarding candidates before cuOpt ever sees
  them, instead of using **prize collection** (drop a task for a declared penalty).
- **No time model at all:** every task ships `[0, 86400]` with `service_times: 1`. **That is why service
  bays, wash and tech work are entirely absent from optimisation.**
- **⭐ The one genuinely routing-shaped problem we own — SERVICE-BAY / WASH SCHEDULING (bay = vehicle, job
  = task, real durations + time windows) — is never posed to cuOpt at all.** Exactly backwards.
- Also unused: **warm start** (`initial_ids`/`upload_solution`) — every call solves from scratch, the
  opposite of a constant sorter; `BatchSolve()` (26.02) for what-if fan-out; gRPC remote exec (26.04);
  solution callbacks (take a good-enough incumbent instead of waiting out the clock); MILP (beta) for
  genuine either/or logic; QP for a smooth quadratic peak-shaving objective.
- **Also missing: the 202 path.** `cuoptAssign()` never handles an async `requestId` + `/v1/status/{id}`
  poll, so every long-poll timeout **silently downgrades to the greedy heuristic** while still reporting
  a nominal path.

### Nemotron — right family, wrong size, wrong job
- **Family (verified 2026-07-31): Nemotron 3 Nano 30B-A3B · Super 120B-A12B · Ultra 550B-A55B.** All
  hybrid Mamba-Transformer MoE, 1M context. Ultra released 2026-06-04 under **OpenMDW-1.1**.
- **⭐ Reasoning vs instruct is a RUNTIME FLAG, not a model choice:** `chat_template_kwargs:{enable_thinking:false}`
  (Super adds `low_effort`, Ultra adds `medium_effort`). **Most teams downgrade the model when they should
  just flip the flag.**
- **⭐ WE RUN EVERYTHING ON ULTRA ($0.50/$2.20 per M) TO TURN 6 DIALS.** Nano is $0.05/$0.20, Super
  $0.085/$0.40. **Nothing currently asked of Nemotron needs Ultra** — ~10× overspend.
- **⭐⭐ WE ARE NOT USING ITS ACTUAL DIFFERENTIATOR: long-horizon AGENTIC TOOL-CALLING over hundreds of
  steps with 1M context.** Both call sites are single-shot, no tools, no multi-turn. **Nemotron is given
  ZERO tools anywhere in the codebase** — all 16 tools live on the Claude path in OttoCommand.
- Not using `nvext.guided_json` (NVIDIA's own XGrammar-backed schema constraint) — we use the weaker
  `response_format:{type:'json_object'}`. No server-side `jsonschema` validation (NVIDIA calls this
  non-negotiable). No `enable_thinking:false` on the approval co-pilot (the orchestrator already learned
  that lesson — v15 comment: `<think>` traces overran max_tokens ⇒ **~60% fell to fallback**).
- **No latency telemetry at all** — `ottoq_decisions` hardcodes `propose_latency_ms:0`/`total_latency_ms:0`,
  so we cannot tell whether Nemotron fits the 0.5-1 s thinking window.
- Approval co-pilot loops **up to 20 sequential HTTP calls** against a ~40 RPM free-tier limit; everything
  is `stream:false` including human-read output.
- **Silent-failure shape:** an unparseable response becomes `recommendation:"hold"`, indistinguishable
  from a genuine hold. Should be `null` + `status:"model_unavailable"`.
- **Available and unused: Nemotron 3.5 Content Safety 4B** — a cheap guardrail to screen operator
  free-text before it reaches a tool-calling loop. (There is no general-purpose "Nemotron 3.5" LLM;
  3.5 so far is only ASR and Content Safety.)

### The pattern both tools need (matches Chase's own doctrine)
**Never on the tick's critical path.** Tick writes a request row and returns; a separate worker calls NIM/
cuOpt off-clock and writes to an advisory table with `valid_for_tick` + `expires_at`; the next tick reads
the latest STILL-VALID advisory or proceeds deterministically. A stale advisory is **dropped, not applied
late** — the software form of the temp-staging pressure valve. Immediate no-brainer: **set
`cuopt_solve_window_ms = 0`** (a policy row, not a code change) to delete the 4 s freeze that can never succeed.

Links: [[reference_nvidia_ai_integration]], [[project_ai_layer_production]], [[project_nemotron_approval_copilot]],
[[reference_ottoq_real_edge]], [[project_runtime_cadence_doctrine]], [[project_energy_orchestration_seam]].
