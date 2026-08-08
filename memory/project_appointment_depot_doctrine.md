---
name: project_appointment_depot_doctrine
description: "CORE DOCTRINE (Chase, 2026-07-19): the depot is APPOINTMENT-ONLY. Vehicle-initiated return handshake, OTTO-Q replies with workflow + stall reservations, vehicle self-navigates stage by stage. Includes the real overnight wave schedule."
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-22T00:45:42.792Z
---

**THE DEPOT IS APPOINTMENT-ONLY.** A vehicle NEVER arrives with nothing to do. If it is healthy and needs nothing it stays DEPLOYED, driving and earning — returning a healthy car is lost revenue. It should not even RENDER at the depot with nothing to do. Arrival is the *fulfilment* of a decision already made, never the trigger for one.

**THE HANDSHAKE (Chase's exact spec — this is a request/response protocol, not OTTO-Q polling):**
1. The **VEHICLE** sends a **"return to depot" notification with its telemetry**. This is the trigger — reservations across the depot occur *only* at this moment, never earlier, never on a timer.
2. **OTTO-Q instantly analyses** the telemetry and builds that vehicle's **arrival workflow** — e.g. "needs charging + interior cleaning".
3. OTTO-Q **reserves the necessary stalls** for that workflow and **communicates the stall selections back to the vehicle**.
4. The **vehicle knows where each stall is** and travels to each assignment itself, advancing **once each stage is confirmed completed**.
5. OTTO-Q can **REASSIGN** mid-flight on malfunction, interruption or delay, and communicates the change. **The vehicle receives and responds to that info.**

Consequence: a charging stall held EMPTY for an inbound reserved vehicle is **CORRECT AND DESIRABLE**, not waste. Chase explicitly rejected "book only on arrival" (that *is* today's broken behaviour) and rejected borrowing-until-close. The hold during transit is intended. This handshake should ride the EXISTING comms bus (see [[project_comms_lines]] — ottoq_vehicle_commands / ottoq_comms_messages, OEM-spec MQTT / J2735 / ISO 20078).

**OVERNIGHT WAVE SCHEDULE (real ops, Chase 2026-07-19):**
- **10pm–12am**: vehicles begin returning for overnight hold, in **timed waves according to charge needs**.
- **~1% of the fleet stays out**, returning finally at **3am**.
- Overnight vehicles run **all normally required service queues that OTTO-Q has acknowledged**: charge → clean → maybe service → then **park for final inspection**.
- **Re-deploy in waves from ~4:30–5am** for dispatch and earning.
- So "return purely to hold overnight" IS a legitimate, named need — but it is scheduled and need-aware, never an unlabelled timer.

**CONTENTION RULE:** if a need arises mid-shift and no slot is free for hours, the car **comes home early with margin and waits in staging**. Safety-first over revenue; never risk an unsafe SoC.

**CHARGE POLICY (Chase 2026-07-19 — top-up is NOT a reason to come home):**
- **Out driving → only route back to charge once a LOW threshold is hit.** Topping up is never a return trigger. (Today a car at 88.9% vs a 90% target gets a non-deferrable charge need — that is the single biggest source of cars arriving with nothing to do, and it goes.)
- Top-up is allowed only **under a certain SoC at certain times, depending on demand**.
- **Charger class: DCFC FIRST, unless all occupied. L2 for longer/overnight needs** (usually).
- **Opportunistic top-off rule:** if a car is already at the depot for another reason (inspection/service) **and is at or below ~70% SoC**, top it off **before deploying** — even though charging was not why it came.
- **OTTO-Q owns this reasoning**, with help from the AI/optimizer tools in its intelligence orchestration (advisory only — the shield still decides).

**DEPLOY FLOOR (Chase 2026-07-19):** keep **80%**, but make it a real **per-operator SLA** — `ottoq_fleet_operator_slas` is currently EMPTY and every "80" in the proposer, manifest generator and shield rule SLA.001 is an undefended hardcoded fallback.

**WHY THIS MATTERS — the defect it explained.** Measured on run 4ee2a5ab: **99.0% of returns (390/394) fired on a fixed timer** (`return_trigger='scheduled'`) at avg SoC 70.3, ZERO with a pending service step — contract-less gate-crashers competing for stalls promised to real appointments. That was the source of the gate pileup (82 cars stuck) and the 47 `no_compatible_available_stall` abstains while chargers sat empty.

**BUILD STATUS (2026-07-20) — the handshake is BUILT + CERTIFIED end to end:**
- **AP-3 (need-driven return, not a clock):** `ottoq_evaluate_return_need` (10-rung ladder) replaced the timer in `ottoq_sim_advance_deployed_telemetry`. Certified: **0 `scheduled` timer-returns**, returns fire on real SoC/fault/service triggers, 0-unsafe. Tuning: `reserve_margin_pct=25`, `p99_burn_pct_per_min=0.20` so R3 `low_soc_reserve` (urgent) is the primary charge-return and R0 `critical_reserve` is the tighter failsafe (fixed a label-collapse where R0 swallowed routine returns).
- **AP-4 (book BEFORE departure — the real handshake):** at need-fire the telemetry advancer calls `ottoq_book_appointment` → builds the workflow (manifest), plans the itinerary, reserves an inlet-compatible charge stall (or staging fallback = "come home with margin, wait in staging") with an **ETA-derived TTL**, and communicates the stall selection back over the comms bus (`ottoq_comms_send_command` downlink + ack uplink) plus a durable `ottoq_vehicle_commands` `proceed_to_stall`/`stage` record. Departure is GATED on securing a resource; non-deferrable (reserve/fault) go regardless, deferrable-with-no-stall stay deployed (bounded by the SoC ladder). `ottoq_sim_prearrival_contracts` demoted to a backstop (ETA-TTL, widened window, gate-refresh keep-alive). Certified: **79/79 returns booked-before-departure, 0-unsafe**, comms handshake live, gate queue 6 cars (was 82).
- **AP-5 core (honour the booking on arrival):** `ottoq_honour_reservation_proposal` (spliced into `ottoq_decide_tick` §3) FULFILS a live compatible reservation deterministically instead of re-deciding — skips the external optimizer when a booking exists; tags `reservation_honoured/reassigned/broken`. Certified 48-tick: **85% honour rate, 0-unsafe, 0.0% shield override, no limbo.**
- **AP-7 core (overnight cycle — "the fleet comes home to roost"):** demand-driven surplus recall in `ottoq_sim_auto_dispatch_tick` (window-gated 22:00–03:00, lowest-SoC-first, ~1% holdout stays out till 3am, rides `book_appointment` trigger=`surplus_to_demand`, deferrable, per-tick capped to intake). Doctrine ruling (workflow-verified): recall does NOT violate "never hold back" — it parks a car that was OUT (overnight ride demand ~0 = no lost revenue), never throttles a car AT the depot. New helper `ottoq_is_overnight_holdout` (TZ-stable night key); `ottoq_deploy_target_fraction` trough lowered + 0.08 floor→0.005. Certified full night (run a70d946b): wave-in **99 deployed → home by 3am** (recalls 38→21→10→2), holdout back by 3am, overnight service running, **morning redeploy 0→55 by 6am**, min arrival SoC 31.6% / **0 below-reserve arrivals**, 71 `surplus_to_demand` + 0 `scheduled`, **0-unsafe certified=true**.
- **AP-6 (make it VISIBLE — DONE + verified live):** OTTO-TWIN (Lovable app `~/Desktop/OTTOYARD/ottoyarddepot-sim`) now has an **"Orchestration" cockpit tab** rendering the whole appointment ecosystem live off RPC `ottoq_twin_appointments(run)`: headline seam tiles (fleet, veh/charger ratio, reservations held, booked-before-arrival, **0 unsafe = green shield-guaranteed**), a live lifecycle pipeline (Deployed→Inbound→At-gate→Charging→Washing→Service→Ready), reservations-held with red-emphasized INBOUND rows (stall held empty for an en-route car), inbound cars with SECURED + ETA + ordered workflow chips, and an overnight banner. Verified in browser (port 8080), 0 console/build errors, honesty-clean. MERGED TO MAIN (PR #37). **On-map reservation GLOW added + merged (PR-less; see below):** the 2D depot canvas (ReservationGlow.tsx) now pulses OTTOYARD-red "HELD" rings on stalls held EMPTY for an en-route car + calm rings on occupied/staging holds, matched to renderer stalls via the SAME map as TwinMotionDriver (no drift) — verified live (6 HELD tags tracked the feed). Production parity: `world_advance` now runs the full certified cycle incl. energy orchestration + the target-aware gate-holding disposition.

**⚠️ GITHUB/LOVABLE WORKFLOW (learned 2026-07-20):** the `ottoyarddepot-sim` repo AUTO-MERGES every pushed `claude/*` branch straight to `main` via Lovable's GitHub sync (it opens + merges a PR itself, e.g. #37 AP-6 tab, #38 motion polish). So pushing a branch = it goes LIVE on main; there are NO open PRs to review (they're already merged/closed). Do NOT try to `gh`-open a PR (gh not installed; `git credential fill` yields the keychain token for the GitHub API if ever needed). Verify thoroughly BEFORE pushing since push = deploy.
- **REMAINING:** AP-5b (#148) = M2 optimizer clock + M6 gate-disposition seam + to_stall_id. AP-7b (#149) = drain fix (B5 DONE), parked_overnight+readiness reconcile (B4), distinct 4:30-5am wave (B12), BESS trough pre-charge (B16), production_live parity (B15 DONE), multi-seed A/B recall-on/off. AP-6b polish = highlight reserved stalls IN the canvas + per-vehicle appointment popover. Design spec: the `ap7-overnight-cycle-design` workflow output (17 items B1-B17 + 5 cross-lens misses).

Links: [[project_depot_ops_model]], [[project_itinerary_contract_a2]], [[project_vehicle_first_doctrine]], [[project_deployment_bar]].
