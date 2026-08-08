---
name: project_energy_orchestration_seam
description: "OTTO-Q is the energy ORCHESTRATOR (reads demand, signals an external energy system); it does NOT actuate energy itself"
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

Chase's architecture for the energy layer (2026-06-21), the energy analog of the OCPP/telemetry seam:

- **OTTO-Q = intelligence/orchestration layer only.** It **reads the demand cycles** (real-time, as load builds, AND historical patterns), analyzes (agentic AI / the layered L1-L3 brain), and **sends control SIGNALS to an external energy system** — e.g. **Schneider EcoStruxure** (per Chase's Schneider discussions): "pull this much now," "switch off," "discharge the battery," "shift to off-peak." OTTO-Q does **NOT** move energy (grid↔battery) itself. It's the brain that commands the energy hardware.
- **OTTO-TWIN holds the energy physics + grid-demand signals** (the randomized energy variables: grid demand, LMP/TOU prices, solar, BESS, weather) and feeds the real demand signals to OTTO-Q. For the DEMO, the twin also runs a **basic middleware "energy system" (an EcoStruxure stand-in)** that RECEIVES OTTO-Q's commands and EXECUTES them on the simulated hardware (grid pull, BESS charge/discharge, charger throttle/shed). Eventually this middleware is replaced by real Schneider/EMS hardware OTTO-Q communicates with.
- **Separation holds:** twin = energy physics + the executing energy system; OTTO-Q = read demand → decide → signal. Same pattern as twin-holds-vehicle-telemetry / OTTO-Q-decides.

**Build (the energy-economics demo, the #1 provable edge per [[reference_av_depot_research]] — demand charges = 30-74% of the bill):**
1. ✅ **seam** (U7) — `ottoq_energy_commands` (OTTO-Q→EMS signal log) + `ottoq_sim_energy_controller` (mock EcoStruxure that executes) + `ottoq_active_charge_cap_kw`. Verified.
2. ✅ **OTTO-Q energy brain** `ottoq_energy_orchestrate` (U7b) — reads latest site demand + grid LMP + BESS SoC → demand_target = service_max × (0.35 expensive / 0.50 else), posts charge-cap + BESS-dispatch. Wired into advance_tick (U7c) for OTTO-Q policies only (NULL/otto_q); baselines uncontrolled. Verified live.
3. ✅ **Enforcement** `decide_tick` §3 (U7d) — defers charges that would push EV load past the active cap (running v_ev_committed_kw tally). SAFE (deferring a charge can't be unsafe; shield still gates every charge). Verified: 0-unsafe preserved, depot flows, cap binds. **BONUS** (U7e): auto_dispatch/prime were dispatching SoC≥60 (under the 80 deploy floor) → 24 under-charged deploys; fixed to ≥80 → **unsafe 24→0**.
4. ⏳ **A/B $ scoring** — demand-charge $ (peak-15min-kW × $/kW) + TOU $ per policy → "$X saved/day". GATED ON: a high-charging-load scenario so the cap binds hard + uncontrolled baselines spike (normal_day peak ~800kW is under the ~1200 cap → little shave shown; need the overnight wave / lower cap). Also fix `ottoq_score_run.energy_peak_kw` (unfiltered depot-wide max → run-filter it like v_peak) + add the $ columns.

NOTE on tuning: energy_orchestrate caps daytime too (service_max×0.5) — may be slightly aggressive vs daytime fast-turnaround; consider day/night-aware cap (loose daytime, tight overnight) when building E4 + the day/night deploy fix.

See [[project_depot_ops_model]] (day/night: overnight = the energy-stress + cheap-TOU window), [[reference_calibration_corpora]] (EIA grid / LMP already ingested).
