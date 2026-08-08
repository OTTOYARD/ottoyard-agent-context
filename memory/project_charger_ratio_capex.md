---
name: project_charger_ratio_capex
description: "Chase's CapEx thesis (2026-07-20): the depot may have too many chargers; cut to ~30 (10 DCFC + 20 L2) and prove OTTO-Q still runs the fleet. Scarcity is where orchestration wins — but the proof needs the COMPRESSED overnight wave + a service-completion metric, not raw throughput."
metadata:
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-21T03:01:39.980Z
---

**THE THESIS (Chase 2026-07-20):** the flagship has 45 charge stalls for 116 vehicles (2.6:1). Maybe that's over-built. Cut to **30 chargers (10 DCFC + 20 L2)** → 3.9:1, saving 15 L2 units of CapEx. *"If there are ample chargers, there is no need for orchestration from that standpoint. When we're trying to optimize for as few chargers to as many vehicles as possible, that's when orchestration comes into play, especially with other constraints and variables. Maybe there's a ratio we're ultimately aiming for — like 1 charger for 3-5 vehicles or more."* This proves OTTO-Q's intelligence *stronger* (scarcity forces smart sequencing) AND is a clean investor CapEx headline.

**FIRST PROBE (single-seed, 2026-07-20) — a REFINING result, not a confirmation:** reconfigured the benchmark depot to 10 DCFC + 20 L2 (convert 15 L2 stalls→staging, keeping ocpp_charger_id as the revert marker; survives benchmark_reset; reverted after) and ran the cert_arm shift A/B. Cutting 45→30 chargers **NARROWED** OTTO-Q's throughput edge (1.33×→1.08×; all three policies converged to ~1.2-1.3/hr). **WHY:** the cert_arm shift scenario SPREADS returns over 6-10h, so 30 chargers aren't the *time*-binding constraint — everyone hits the same physical charger cap and converges. So **throughput_per_hr is the WRONG metric and the spread shift is the WRONG scenario** to prove this.

**THE RIGHT EXPERIMENT (Chase already named it):** the COMPRESSED **overnight wave** — 100-120 AVs returning 22:00-03:00 in a tight window, double-constrained by few chargers AND little time. There the differentiator is NOT raw throughput (still capped) but: **(a) % fleet fully charged+serviced by ~5am, (b) 0 stranded cars, (c) 0 under-charged AM deploys, (d) energy peak w/ BESS.** FIFO under a compressed wave charges whoever arrives first (often a higher-SoC car) while low-SoC cars strand; OTTO-Q's lowest-SoC-first recall + staging→charge→staged cycling + reservation honour should service *everyone*. BUILD (task #150): run the AP-7 overnight-wave scenario (clock-pinned ~20:00, full fleet, recall) against 30 chargers, OTTO-Q vs FIFO, add a `fleet_serviced_by_morning` metric, and **sweep 45→30→24→20 chargers to find where naive strands cars but OTTO-Q holds** = the recommended design ratio + the CapEx headline.

Links: [[project_ecosystem_positioning]], [[project_appointment_depot_doctrine]] (the overnight recall this rides), [[reference_ottoq_real_edge]] (metric-honesty: don't quote deploys_total).
