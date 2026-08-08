---
name: project_cuopt_eligibility_doctrine
description: "⭐⭐CORE DOCTRINE (Chase 2026-08-01): cuOpt re-optimizes FREELY before a vehicle approaches the depot; at APPROACH the itinerary FREEZES into that vehicle's set arrival workflow; only a malfunction/congestion/flag RE-OPENS it. PLUS the architecture mandate: cuOpt is built INTO OTTO-Q, not bolted on."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-01T21:15:45.087Z
---

# ⭐⭐ THE THREE-ZONE ELIGIBILITY RULE (Chase, verbatim intent)

> *"It can override or re-optimize **before the vehicle enters the depot**. Once the vehicle has
> **approached** the depot or has entered, it should have a **set workflow** that it goes through for
> its individual needs for that arrival sequence. Now, if there's an issue, such as **hardware
> malfunction or congestion or flag** for some reason, cuOpt can **reload and be utilized again**. So
> it really shouldn't override unless it's **absolutely necessary** like a malfunction or other cases
> you can predict, or it's **before the vehicle enters the depot** for optimization."*

| Zone | Vehicle state | cuOpt authority |
|---|---|---|
| **A — OPEN** | en route / returning, **has NOT yet approached** | **Full re-optimization.** Reservations here are PROVISIONAL; cuOpt may freely reassign. This is where the optimizer earns its keep. |
| **B — FROZEN** | **approached OR entered** the depot | **No override.** The itinerary IS the vehicle's committed arrival workflow, planned for its individual needs. Leave it alone. |
| **C — RE-OPENED** | frozen, then hit by **malfunction / congestion / flag** | **cuOpt re-engages** for that vehicle. This is the *exception* path, not a routine tier. |

⚠️ **THE BOUNDARY IS "APPROACHED", NOT "INSIDE THE GATE."** I originally proposed freezing at the
depot walls; Chase tightened it to **approach**. The freeze starts EARLIER than the gate. Any
implementation must find the real state-machine transition that means "approaching" and freeze there —
do not silently substitute `arrived_at_gate`.

**Zone C is the push-wake from [[project_runtime_cadence_doctrine]]** ("a random flag for immediate
vehicle return… OTTO-Q can utilise the TEMPORARY STAGING exactly for this purpose"). Same mechanism,
now with a named consumer. It is also the ONLY path that may touch an in-depot vehicle, so it must
route through [[project_indepot_reassignment_gate]] (tech approval) — Zone A needs no such approval
because nothing is committed yet.

**Why this is good engineering, not just policy:** Zone A is exactly where re-optimization is
*cheap* (nothing physical has happened yet) and Zone B is where it is *expensive and confusing*
(a vehicle mid-workflow being re-routed). The doctrine puts the optimizer where its leverage is
highest and its blast radius is smallest. Consistent with [[project_full_service_visit_doctrine]]
(every visit atomic) and [[project_perimeter_hold_doctrine]].

---

# ⭐ ARCHITECTURE MANDATE — cuOpt is built INTO OTTO-Q

> *"cuOpt should be built **INTO** OTTO-Q, so OTTO-Q working literally is **all feature in one**.
> NVIDIA and our own proprietary intelligence."*

**This is not cosmetic — it is the structural fix for the 0-of-84,000 problem.** Today cuOpt is an
*external stranger*: the decider runs its own greedy assignment, then separately invites cuOpt, whose
answer arrives late and can be structurally refused. Two competing paths racing each other is exactly
why it has never won. See [[project_nvidia_layer_truth_2026_07_30]].

**Target shape:** ONE assignment step inside OTTO-Q that uses cuOpt as its engine, with the local
greedy as its **fallback**, not its rival. The L1 deterministic shield (52 safety rules) still
validates every output — that preserves Chase's stated model (optimizer proposes → deterministic
layer allows/denies → dispatch) while removing the ability to skip the optimizer entirely.

**Deployment note (my read, for later):** "built in" here means *product and pipeline integration*,
not necessarily in-process. But self-hosting cuOpt eventually also buys: no per-tick API spend, lower
latency, and on-prem/air-gapped operation — which matters for [[project_deployment_bar]] (an OEM
plugging in). Worth revisiting once enactment is proven.

---

# 📌 STANDING: my role
Chase: *"you were acting as the **senior engineer for fleet research and depot operations and
intelligence**. So when you find a logic question like this, feel free to give your **best input
based on data and analysis from your expert opinion**."*
⇒ On logic/doctrine questions, **lead with a reasoned recommendation grounded in measured data** —
do not merely present options and wait. Reinforces [[feedback_gap_ownership]] and
[[feedback_plain_language]]. Asking is still right when it is genuinely his call (doctrine, business
priority); it is NOT right as a substitute for analysis.

---

# ✅ SEQUENCING DECIDED (2026-08-01)
Chase's priority: **prove cuOpt is actually used** first.
**KEY INSIGHT — the first win needs NO doctrine change at all.** Of 71 stall_assignment decisions on
run 9ea73866, **54 (76%) exited at Gate B** (`reservation_honoured`) — but **17 (24%) did not**.
Those 17 are vehicles with no reservation yet, i.e. already Zone A. Fixing the ORDERING alone should
stamp `source='cuopt'` on some of them. So:
1. **Ordering fix only** (split tick) → prove a `cuopt`-stamped decision. Zero doctrine risk.
2. **Then** Zone A/B/C eligibility → opens the remaining volume.
3. **Then** the A/B that answers whether cuOpt actually beats our greedy (never measured).

---

# ✅ 2026-08-01 — cuOpt ENACTED FOR THE FIRST TIME (0 of 84,000 → 2 of 250)
**Split-tick ordering fix applied to `ottoq_demo_metronome`** (snapshot **6163**, pre-md5
`6c1732a6d108d06b4648b2d525444ed5`): fire cuOpt on one beat and COMMIT (the commit is what makes
pg_net actually send), decide on the next. Run **6c6fb2bb**, 43 ticks.
- NVIDIA answered **8 times, all HTTP 200, status "Optimal"**; **19 proposals** produced.
- **19 external HTTP receipts reconcile EXACTLY to 19 DB rows** — rules out in-DB fabrication.
- **2 enacted with `source='cuopt'`** (0.8% of 250 stall assignments); 4 expired, 13 superseded.
- **Physically followed through:** two `begin_charge` commands executed with cuOpt's exact payloads
  (stall 609910b1 @212.5 kW, 765adddb @85 kW). The depot moved on NVIDIA's numbers.
- Provenance checked, not trusted: the edge function can only call `ottoq_submit_external_proposal`,
  which always writes status `pending` — it is **structurally incapable** of writing a decision or
  marking anything `enacted`. So OTTO-Q's own engine enacted it.
- **Evidence preserved in `public.cuopt_enactment_proof_2026_08_01`** (2 decisions + 21 proposals +
  51 NVIDIA receipts). ⚠️ Named WITHOUT the `ottoq` prefix **on purpose** — the purge sweeps every
  `public.ottoq%` table with a `sim_run_id`. The temporary purge keep-list hack was reverted after.

## 🔴 TWO REAL DEFECTS THIS SURFACED
1. **[[project_ledger_transposition_defect]] / task #17 — the booking ledger recorded the
   vehicle→stall pairing TRANSPOSED** vs the executed commands, same transaction, same timestamp.
   Bigger than cuOpt: the reservation book IS the forward availability calendar. **Do not quote any
   booking-derived capacity number until root-caused.**
2. **Task #18 — the split tick halves OTTO-Q's decision cadence and defeats `cuopt_debounce_s`**
   (duplicate BILLABLE NVIDIA solves). The better end-state is the architecture mandate above: fold
   cuOpt into the ONE assignment step with greedy as fallback, which removes the cadence cost.

## 📌 "APPROACHING" DOES NOT EXIST — and the fix is small
MEASURED: `vehicles.current_state` has exactly **17 labels**; none means approaching/inbound-within-N.
`en_route_to_depot` is all-or-nothing — a car 40 min out and one 90 s out are indistinguishable.
**But the ETA clock already exists**, so the band is derivable.
⭐ **RECOMMENDED: a DERIVED read-only band from the existing ETA countdown** — no new state, no change
to the 17 labels, no new writers, nothing can get stranded in it. Days, not weeks. Adding a real
`approaching` state would force ~15 state-reading code paths to learn it and create a stall risk.
⭐ **AND IT IS THE MISSING INPUT, NOT A SEPARATE FEATURE:** cuOpt went quiet after ~90 s because it
was only ever shown cars already AT THE GATE competing for stalls that were already gone (measured:
17, 12, 11, 6, 10 waiting cars against **0** free stalls). Show it the cars 10 minutes out and there
are still stalls to allocate. **The approach band is what makes the optimizer worth having.**

---

# ⚠️ 2026-08-01 — THE BAND IS BUILT AND HALF-WORKING. Read before trusting it.
**Objects:** `public.ottoq_approach_zone(...)` + view `public.ottoq_approach_band`. Derived, as
decided — **no new `vehicles.current_state` label** (still 17), no new writers, nothing can strand.
Threshold is a real policy knob (`approach_freeze_minutes`=10, horizon=30). "approached" is its own
derived boundary, **NOT** silently substituted with `arrived_at_gate`. Rollback snapshots 6170/6171.

**✅ THE INPUT STARVATION IS FIXED — this was the band's whole purpose.** MEASURED across 25 ticks:
cuOpt was offered **164 vehicle-ticks of Zone A cars still en route**, and **free stalls never fell
below 11** (range 11-132). Before the band it faced the 0-free-stalls wall (17,12,11,6,10 waiting
cars vs **0** stalls). It is now being handed a solvable problem while inventory still exists.

**🔴 BUT TWO REAL DEFECTS — the band is not yet doing its job:**
1. **Zone C over-fires catastrophically on a FALSE signal.** MEASURED zone_reason counts:
   `reopened:hardware_malfunction` = **539** vehicle-ticks vs `reopened:flag` = **10**. Normal
   charger occupancy is being read as a hardware malfunction. Zone C is the *exception* path
   (founder: *"shouldn't override unless absolutely necessary"*) — at 539 it is the default, which
   inverts the doctrine. **Fix the malfunction predicate before relying on any zone number.**
2. **The ETA is QUANTIZED TO THE 30-MINUTE TICK GRID**, so a 10-minute freeze boundary is crossed
   only **3 times in 1,850 observations** and the "cuOpt window" collapses to a single point at
   exactly 30.00 min. ⇒ the knob is real but has almost nothing to bite on.
   ⭐ **THE FIX IS THE CLOCK, NOT THE BAND:** in `live` playback mode each tick advances by real
   elapsed time, so ETA becomes continuous. See [[project_runtime_cadence_doctrine]] /
   [[project_twin_realtime_clock]] — Phase A (1:1 ratio) is already done and verified exact.

**🔴 AND THE PAYOFF IS NOT VISIBLE: cuOpt enactments FELL 8 → 2** (proposals 78 → 48; 34 superseded,
11 expired). Better input, fewer outputs. INFERRED (not isolated): the proposal-vs-greedy race —
the local path still out-runs the proposal inside the tick. **This is now the highest-value next
investigation**, and it is the same suspect as [[project_nvidia_layer_truth_2026_07_30]].

**Calendar work shipped alongside (snapshots 6167/6168/6169):** breadth widened — wash 2→40,
charge_dcfc 76→129, charge_l2 38→82, perimeter_hold 11→27, service 3→9, detail 4→6. **Staging is
STILL never booked (0→0).** ⚠️ **But end-to-end coverage REGRESSED: 99 of 209 enacted assignments
never reach the calendar at all.** ⭐ **STRUCTURAL FINDING THAT CHANGES THE PLAN:** all 209 enacted
assignments were CHARGING (128 L2 + 81 DCFC) — **the decision engine only ever emits charging
assignments.** Wash/detail/service/staging/holds never pass through it; they reach the calendar via
a separate writer (`ottoq_book_workflow`). ⇒ "teach the decision path to book non-charging spaces"
**cannot** close the doctrine gap. That is a design decision, not a patch.

**✅ SAFETY CLEAN — no revert.** Zero physical charging conflicts in both runs (charge sessions rose
165→272). Starvation **improved**: baseline left **2 vehicles permanently stranded**, post-change
**0**. Gate B byte-identical (witness snapshot 6175). `ottoq_events` untouched. cron 12 restored ACTIVE.

**📌 NUMBER RECONCILIATION (I verified this myself — an agent claimed 98.8% "did not reproduce"):**
It reproduces exactly. The three figures are different QUESTIONS, all true:
| question | baseline | after |
|---|---|---|
| of decisions that produced a booking, is the stall right? | 3/81 = **3.7%** | 79/80 = **98.8%** |
| of ALL decisions with a stall, is it right? | 3/185 = **1.6%** | 79/116 = **68.1%** |
**Always state which denominator.** 98.8% is CONDITIONAL on a booking existing; 68.1% is the honest
end-to-end number. Quoting 98.8% alone overstates calendar health.

Links: [[project_booking_ledger_divergence]], [[project_nvidia_layer_truth_2026_07_30]], [[project_indepot_reassignment_gate]],
[[project_runtime_cadence_doctrine]], [[project_ai_layer_production]], [[project_deployment_bar]].
