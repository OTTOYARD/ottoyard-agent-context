# memory/ — the raw distilled corpus

**79 verbatim memory files** written by Claude Code across four months of building OTTOYARD
(April–August 2026), copied here unaltered. This file is the index; each line points at one memory.

**These are primary sources.** `docs/` is synthesis on top of them. When the two disagree, check the
`modified:` timestamp in the memory file's frontmatter — the newer one usually wins.

> ⚠️ **Every file here is a point-in-time observation.** A memory that says "function X does Y at
> line 40" was true when written. **Verify against the live database or current code before you
> assert it as fact or build on it.** Several of these files are themselves records of Claude
> discovering its own earlier belief was false — that pattern is the norm here, not the exception.

Links written as `[[name]]` refer to the file `<name>.md` in this directory.

---

## How to work with Chase
- [User Profile](user_profile.md) — Chase Ballenger, OTTOYARD founder building AV/EV depot infrastructure
- [Plain Language](feedback_plain_language.md) — ⭐Chase=CEO, Claude=frontier CTO. Plain words, lead with meaning + business value + the trade-off
- [Confirm Every Logic Point](feedback_confirm_every_logic_point.md) — ⭐⭐every logic point Chase states: READ→REASONED→PLANNED→BUILT→CONFIRMED with evidence. Flag contradictions
- [Fix, Don't Just Flag](feedback_fix_dont_just_flag.md) — ⭐⭐finding a defect is HALF the job. Go into the code, fix it, retest, confirm it cleared. Never just call things out
- [Gap Ownership](feedback_gap_ownership.md) — Chase never finds a gap first; verification receipts + a proactive gaps register
- [Realism + Standalone Bar](feedback_realism_standalone.md) — ⭐OTTO-Q must field realistic UNSCRIPTED demand, never pre-programmed; vehicles arrive with varying service manifests
- [GitHub Is Source of Truth](feedback_github_is_source_of_truth.md) — ⭐Chase's ONLY git action is clicking Merge. Never hand him a git command; Desktop clones are Claude's
- [Rebuild, Not Benchmark](feedback_rebuild_not_benchmark.md) — ⭐during rebuild the bar is "it works and moves correctly", NOT beating a baseline. Numbers = evidence, never a score
- [Ask Early](feedback_ask_early.md) — quick clarifying questions UP FRONT, before deep research or building
- [Scout External Tools](feedback_scout_external_tools.md) — search GitHub/APIs/tools at implementation; don't rely solely on internal build
- [UI Audit + Branding](feedback_ui_audit_branding.md) — standing want: full audit of OrchestraAV + OTTO-PULSE to match OTTOYARD branding; no lazy UI
- [UE Console Safety](feedback_ue_console_safety.md) — NEVER cmd+a+Delete in the UE Python console; use triple_click

## Read before quoting any number
- [OTTO-Q's Real Edge](reference_ottoq_real_edge.md) — ⭐⭐READ BEFORE QUOTING ANY COMPARATIVE NUMBER. Baseline was invalid once; never quote vehicles_turned_around
- [Booking Ledger Divergence](project_booking_ledger_divergence.md) — 🚨2026-08-01 FIXED 3/81→79/80. All booking-derived capacity numbers VOID. 📌Never disable cron 12 — it IS the START engine
- [Vehicle-First Doctrine](project_vehicle_first_doctrine.md) — ⭐vehicles/chargers never held back; energy shave only via forecast+BESS. Read before quoting any energy number
- [Ecosystem Positioning](project_ecosystem_positioning.md) — ⭐pitch the WHOLE integrated ecosystem, never a single layer's number

## Core doctrine (what OTTO-Q is)
- [Forward Availability Doctrine](project_forward_availability_doctrine.md) — ⭐⭐orchestration = a forward-time occupancy calendar of every space, not a refusal protocol
- [Forward Service Scheduling](project_forward_service_scheduling_doctrine.md) — ⭐⭐⭐"overdue" is a FAILURE STATE, not a trigger. Work is predicted + reserved before the next arrival. 11 logic points
- [Arrival Forecast + Learning](project_arrival_forecast_learning_doctrine.md) — ⭐⭐the forecast is where ML belongs, and it must END IN A RESERVATION. You cannot train on a constant
- [Needs Draw ALREADY EXISTS](project_needs_draw_already_exists.md) — 🚨2026-08-07 DON'T BUILD ONE. `ottoq_run_boot_draw` already draws tire/brake/health per run, in band. Wash bays were empty from a NIGHT GATE discarding 34 due washes. Seed 424242 is PINNED so every run is identical
- [Needs = Probability Draw](project_needs_probability_draw_doctrine.md) — ⭐⭐needs are a seed-deterministic draw at run start (~1 in 5-10), NOT accumulated wear. Wear modelling is a non-requirement
- [Appointment Depot Doctrine](project_appointment_depot_doctrine.md) — ⭐depot is APPOINTMENT-ONLY; vehicle signals return → OTTO-Q replies with workflow + reserved stalls
- [Full-Service Visit Doctrine](project_full_service_visit_doctrine.md) — ⭐every visit is ATOMIC: ~100% charge + ALL services before redeploy
- [Runtime Cadence Doctrine](project_runtime_cadence_doctrine.md) — ⭐⭐the TICK ≠ OTTO-Q's decision rate. Nothing runs until START; PAUSE freezes; STOP wipes the twin but saves the log
- [Arrival + Tick Doctrine](project_arrival_and_tick_doctrine.md) — ⭐ticks only on explicit start; arrivals are NEVER auto-parked
- [Perimeter Hold Doctrine](project_perimeter_hold_doctrine.md) — ⭐temp vs perimeter staging = different PURPOSES, not overflow tiers
- [In-Depot Reassignment Gate](project_indepot_reassignment_gate.md) — ⭐inside depot walls, no re-route without tech approval
- [cuOpt Eligibility Doctrine](project_cuopt_eligibility_doctrine.md) — ⭐⭐3 zones; itinerary FREEZES at approach. cuOpt belongs INSIDE OTTO-Q as the assignment engine, greedy as fallback
- [Real-Data Ingestion Seam](project_realdata_ingestion_seam.md) — ⭐never cross twin builds into OTTO-Q; real telemetry plugs into the SAME tables/RPCs the twin writes
- [Deployment Bar](project_deployment_bar.md) — ⭐'production' = the most realistic hardware-integration path for an OEM; security deferred until post-funding
- [Depot Ops Model](project_depot_ops_model.md) — Chase's real day/night robotaxi ops; demo = relentless churn
- [Robotics/Automation Direction](project_robotics_automation_direction.md) — staffing = FIXED slider; service-resource model stays resource-agnostic (human ↔ robot)
- [Demo Loop Ops](project_demo_loop_ops.md) — ⚠️bounded demo runs (~1 sim-day), never 30-day marathons

## Architecture + where things live
- [OTTO-Q Project](project_ottoq.md) — OTTO-Q (brain) + OTTO-TWIN (world sim); read Desktop/OTTO-Q V1/RESUME_HERE.md first
- [Architecture Separation](project_architecture_separation.md) — repo `ottoyard-OTTO-Q` is MISLABELED (=OrchestraAV). Read the doc before touching repos
- [OTTO-Q/Twin Boundary](project_ottoq_twin_boundary.md) — ⭐boundary doc lives on an UNMERGED branch; 3 blockers remain
- [Tech Stack](reference_stack.md) — Supabase + TypeScript + Lovable dashboard, OCPP chargers, AV fleet APIs
- [Live Data Model](reference_live_data_model.md) — how real data reaches the cockpits; energy series EMPTY when no live run
- [Twin Variable Backend](project_twin_variable_backend.md) — the ONE eventual source for realistic feeds; current core reads are TEMPORARY scaffolding
- [OTTOYARD Brand](reference_ottoyard_brand.md) — design tokens (dark #06070A, red #C8102E, Chakra Petch/Inter Tight/JetBrains Mono)
- [Parallel Sessions + Graphs](reference_parallel_sessions_and_graphs.md) — ⭐hub-and-spoke operating model; the researched verdict on graph DBs is mostly DON'T

## The AI / optimizer layer
- [NVIDIA Layer — Audited Truth](project_nvidia_layer_truth_2026_07_30.md) — 🚨"cuOpt is async" is FALSIFIED. Real cause = ORDERING (pg_net fires post-COMMIT). Read before any cuOpt work
- [AI Layer Production](project_ai_layer_production.md) — cuOpt fires + drives via the metronome solve-window; Nemotron auto-actions 6 dials
- [NVIDIA/AI Integration](reference_nvidia_ai_integration.md) — cuOpt drives enacted SIM decisions, advisory only in production. No GPU needed (hosted API)
- [Nemotron Approval Co-Pilot](project_nemotron_approval_copilot.md) — Nemotron reviews each approval with a rationale, NEVER decides
- [Frontier Intelligence Architecture](project_frontier_intelligence_architecture.md) — 3-tier runtime-assurance stack; 7 founder questions inside
- [Frontier Stack](project_frontier_stack.md) — mandate: replace every heuristic with a real optimizer/learned model. Repo ~/Desktop/ottoq-intelligence
- [Frontier Core](project_frontier_core.md) — the 4 deep capabilities greenlit before cockpit/visual; FC-0 + FC-1 done

## Build history + open work
- [Orchestration Build 2026-08-01](project_orchestration_build_2026_08_01.md) — ⭐⭐THE BUILD: needs card → atomic enactment → space-agnostic assignment → armed overlap guard
- [Forward Bay Reservation 0011](project_forward_bay_reservation_0011.md) — ⭐the return signal now holds a BAY ~30 sim-min pre-arrival (7/7). Contains 3 FALSIFIED premises: planned_return_at is the DISPATCH plan not the ETA; service_definitions shares 1 of 15 atom names; the outbound-itinerary blocker doesn't exist
- [Depot Layout Unification](project_depot_layout_unification.md) — 🚨2026-08-06: the DB's depot had 54 overlapping stalls + 20 unreachable chargers. Renderer wins; DB is the source. 0010 pending
- [OTTO-Q Logic Completeness](project_ottoq_logic_completeness.md) — ≈90% of decision logic in place + validated 0-unsafe; RLS is the deploy gate
- [A2 Itinerary Contract](project_itinerary_contract_a2.md) — twin-owned timed legs; charge curve measured 2.5× too fast
- [Twin Arrival Cycle](project_twin_arrival_cycle.md) — U6 closed the dispatch→return cycle; linchpin = decide_tick charge-assignment starvation
- [Comms Lines](project_comms_lines.md) — OEM-spec fleet comms live-wired into the tick loop + teleop queue; only a real MQTT broker remains
- [Energy Orchestration Seam](project_energy_orchestration_seam.md) — OTTO-Q reads demand and SIGNALS an external energy system; never actuates
- [E4 Energy-$ Proof](project_energy_e4_charging_fix.md) — fixed charging-load-invisible + charge starvation; numbers now stale
- [Charger-Ratio CapEx](project_charger_ratio_capex.md) — thesis: cut chargers to save CapEx. Real proof = a COMPRESSED overnight wave
- [Black Box Self-Audit](project_blackbox_selfaudit.md) — founder-side flight recorder; backend done, frontend panel remaining
- [Cards Integration Map](project_cards_integration_map.md) — cards build: verified repo maps, card RPC design + honesty rules

## The twin's visual + motion layer
- [Twin Motion Defects](project_twin_render_offmap_bug.md) — ⭐⭐off-map NEVER reproduced; real bug = route() offsetting its own endpoints. Measure BODY OVERLAP, not centre distance
- [Depot Lanes / RAILS](project_depot_lanes_rails.md) — right-of-way painted from the LaneGraph; charge lanes northbound, enter east / exit west
- [Depot Plan Scale](reference_depot_plan_scale.md) — ⚠️1 unit = 0.4785 m in the cockpit; 1.5699 ft/unit in the site-plan export. Mixing them caused a 1.57× outage
- [Depot Motion Timing + Realism](project_depot_motion_timing_realism.md) — real per-vehicle pacing; timing=twin/OTTO-Q, motion=renderer
- [Twin Real-Time Clock](project_twin_realtime_clock.md) — ⭐Chase wants 1 real sec = 1 sim sec + a viewing multiplier; investigated, NOT built
- [Sim Visual + Vehicle Flow](project_sim_visual_movement.md) — realistic movement + congestion-awareness is a key demo
- [Visual Layer Plan](reference_visual_layer_plan.md) — Isaac/Omniverse plan; the streaming viewer + a USD depot ALREADY EXIST and point at a live AWS GPU box

## Hard-won lessons (read before repeating the mistake)
- [Parallel Session Tree Collision](project_parallel_session_tree_collision.md) — 🚨two chats in one repo FOLDER share ONE working tree. `git worktree` per session
- [DB Capacity Ceiling](project_db_capacity_ceiling.md) — ⚠️events table once ate 9.1GB of 11GB; purge has a quadratic ctid defect. ~40 verification findings inside
- [DB Outage 2026-08-05](project_db_outage_2026_08_05.md) — ✅RESOLVED: 14GB→342MB, ticks 48-65s→~1s. How a nearly-full container was recovered without dropping anything
- [leg_type Abort Root Cause](project_legtype_abort_root_cause.md) — 🚨ONE unmapped word aborted every decision while cron reported success. Every seam must be a TOTAL function
- [Tick Cost Root Cause](project_tick_cost_root_cause.md) — ⭐the slow tick was event-write amplification, NOT the shield. Includes the profiling technique
- [Tick-Invariance Cert Defects](project_tick_invariance_cert_defects.md) — ⭐two seeds are contaminated; a funnel figure was a teardown ARTIFACT
- [DB Surgery Lessons](reference_db_surgery_lessons.md) — ⚠️NEVER DROP a function before capturing pg_get_functiondef; verify sub-agent "couldn't find it" claims yourself
- [RLS Public-Policy Hole](project_rls_public_policy_hole.md) — ✅FIXED: 31 tables were world-writable. Read `pg_policies.roles`, NEVER the policyname
- [Mapbox Token Incident](reference_mapbox_token_incident.md) — ⚠️a leaked token cost $1-2k. Policy = MapLibre + free tiles, never metered raster, never a committed token

## Research + reference
- [Service Ops Research](../OTTOQ_SERVICE_OPS_RESEARCH.md) *(doc)* — fact-checked depot servicing: 4 categories; cadences/durations are tunable ASSUMPTIONS
- [AV Depot Research](reference_av_depot_research.md) — headline edge = energy/demand-charge economics; the density pitch is design-target, not fact
- [Calibration Corpora](reference_calibration_corpora.md) — 7 real datasets already ingested + fitted into the twin

## Other
- [ZeroPressure](project_zeropressure.md) — Chase's pressure-washing side business; static SEO site, Vercel, zeropressuretn.com
