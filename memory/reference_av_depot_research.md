---
name: reference_av_depot_research
description: "Fact-checked AV-depot research conclusions that reorder OTTO-Q's demo strategy + twin calibration"
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

Deep-research findings (2026-06-21, 104 agents, 20 claims verified / 5 refuted) — full doc: **`~/Desktop/OTTO-Q V1/OTTOQ_AV_DEPOT_RESEARCH.md`**.

**Strategic reframe (important):** OTTO-Q's biggest PROVABLE demo edge is **energy economics (demand-charge peak-shaving)** — demand charges are **30-74% of a commercial/DCFC bill** (~$10/kW on the 15-min peak), the single strongest quantified prize. Next: **charger-reliability-aware dispatch** (public DCFC field-tested 72.5% functional; maintained private far higher → contrast is fault-ROUTING) and **guaranteed-SoC-by-deployment / 0 unsafe deploys**. Then utilization/packing (45-82× $/kWh swing low- vs high-use). Verified savings are MODEST (3-13%); do NOT headline big throughput/turnaround multipliers.

**The density pitch (100+ vs 50-80 vehicles/depot) is NOT externally evidenced** — keep it as a *demonstrated-in-our-calibrated-twin design target*, never a cited industry stat (won't survive diligence). Real anchor topology: Waymo 201 Toland ≈ 38 DCFC ports (~60 kW, dual-port) for ~100+ vehicles ≈ **1 charger per 3 vehicles** — chargers are the scarce binding constraint (that scarcity is WHY orchestration matters).

**GOOD NEWS — OTTO-Q already has the top-3 provable edges built:** energy co-optimizer (OQ-4, grid+PV+BESS = #1), charger-aware proposer + recovery ([[project_twin_arrival_cycle]] U6b/c/f = #2 reliability-routing), L1 shield 0-unsafe (#3 SoC safety). The demo must MEASURE + SHOWCASE these, not chase a throughput race.

**Baseline for A/B = uncontrolled "dumb" charging:** ignores TOU/demand peaks, treats all chargers healthy (no fault-routing), no SoC-by-deployment scheduling.

**DO NOT CITE (refuted):** Paren Reliability Index trajectory; "80% fast-charge knee"; "Waymo charges at 30 kW". Arrival curves / dwell / cleaning cadence / returns-per-day / the 50-80 cap are unsupported by public evidence → calibrate from our ingested corpora ([[reference_calibration_corpora]]) + OEM telemetry. See [[project_depot_ops_model]].
