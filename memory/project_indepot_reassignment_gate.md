---
name: project_indepot_reassignment_gate
description: "⭐FOUNDER SAFETY DOCTRINE (Chase 2026-07-24): inside the depot walls a vehicle may NOT be re-routed / switch workflow queue without technician-supervisor approval. Enforced via ottoq_indepot_reassignment_guard + re-opt timing policies. Resource faults still auto-reroute (exception triad)."
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-29T02:15:38.517Z
---

**THE RULE (Chase, verbatim intent):** the re-optimizer's re-learning is exactly what he wants — but with timing foundations: *"The vehicle can't get re-routed mid depot unless a supervisor/technician approves it... a vehicle can't switch workflow queuing once it is inside the depot walls without technician/supervisor approval."*

**Why:** physical safety + predictability — a vehicle already moving/parked inside a constrained yard must never receive a surprise re-route from an optimizer; humans own in-yard exceptions.

**How it's ENFORCED (shipped 2026-07-24, migrations `indepot_reassignment_gate_and_reopt_timing_v2` + `approvals_allow_indepot_reassign`):**
- **`ottoq_indepot_reassignment_guard(vehicle, run, reason, payload)`** — THE reusable gate for ANY reassignment path (re-optimizer today; Nemotron actions, orchestrate moves, future paths MUST call it):
  - outside walls (`deployed/en_route_to_depot/en_route_to_deployment/offline`) → `allowed, outside_walls`;
  - inside + `reason='resource_fault'` → `allowed, resource_fault_auto_reroute` (keeps the exception-triad doctrine: broken charger/bay = necessity);
  - inside + discretionary → **DENIED**, queues `ottoq_ops_approvals` type `indepot_reassign` (30-min expiry, no duplicate pendings) → tech/supervisor approves in the ops queue. CHECK constraint extended to allow the type.
  - **VERIFIED:** all 3 branches + no-dup behavior tested live.
- **Timing policies (registered in ottoq_policy_param_catalog, clamped):** `reopt_min_eta_min` (5; never rebook a final-approach vehicle), `reopt_cooldown_min` (20; one booking change per vehicle per window), `reopt_max_per_tick` (6; bounded wave).
- The reservation-book re-optimizer (see [[reference_nvidia_ai_integration]]) filters to en-route vehicles AND calls the guard belt-and-braces; each decision receipt records `wall_gate` mode.

**STANDING RULE FOR FUTURE BUILDS:** any new code path that changes an in-depot vehicle's stall/queue/workflow MUST call `ottoq_indepot_reassignment_guard` first — never bypass. Links: [[project_appointment_depot_doctrine]], [[project_vehicle_first_doctrine]].

---

## ⚠️ AMENDMENT (Chase, 2026-07-28) — auto-advance while there are no technicians

Chase, verbatim: *"the one note about reassignment should flow automatically for now (since we don't have technicians). Let the stages auto advance once their designated timing at that specific stage is complete, but let it be 'marked' as complete, as a technician would approve on OTTO-PULSE once a task in a stage is complete. So for now, it will just auto-advance through stage approvals or completions since there is no human to press complete and 'advance' it to the next OTTO-Q assignment the vehicle was designated."*

**This is NOT a bypass — it is an auto-approver.** The gate is still called and an approval row is still written, with the same shape a technician's would have; only `decided_by` differs. Swapping in real humans later is a config change, not a code change.

**BUILT 2026-07-28:** `ottoq_stage_advance_approval(vehicle, run, depot, stage, next_stage, visit, clock, payload)` → jsonb. Writes `ottoq_ops_approvals` (`approval_type='tech_greenlight'`). Reads run knob `tech_approvals_required` via `ottoq_policy_get` (default **0** = no techs → auto-approve, `decided_by='auto_advance_no_tech'`). Set it to 1 and the identical call returns `approved=false, mode='awaiting_tech'` with the row left `pending`. Both modes verified live.

**⚠️ AUDIT FINDING 2026-07-28 — the original doctrine was NEVER actually enforced.** Exactly one function called `ottoq_indepot_reassignment_guard`, passing the argument that makes it auto-approve, and never read the result. The highest-volume in-depot move (staging→charging in `ottoq_decide_tick`, **61,378 enacted**) bypassed it entirely; only one approval row exists in the system's whole history. So the "shipped + VERIFIED 2026-07-24" claim above covers the guard function itself, not its adoption. Wiring every reassignment path to the guard is outstanding work. See [[project_forward_availability_doctrine]].
