---
name: calibration-corpora
description: "The 7 real-world datasets already ingested + fitted into OTTO-TWIN's variability generators (ACN charging, CA DMV AV, EIA grid, NYC TLC, NOAA, NREL, charger reliability). No live API keys stored."
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

OTTO-TWIN's randomized variability is CALIBRATED to real-world data (NOT raw replay): real datasets → fitted distributions (16) + correlations (1) + profiles (8) in the `ottoq_calibration_*` tables; the sim generators sample from these. Ingested 2026-05-22..05-28 via `twin_ingest/ingest_calibration.py`. This IS the real data Chase referenced ("UCLA/CA DMV/charging API data") — UCLA ≈ Caltech ACN + UC Berkeley charger study.

The 7 corpora (`ottoq_calibration_datasets`):
- **acn_data** — Caltech ACN-Data, 130k+ real EV charging sessions (4k ingested, 2018) → charge_duration, energy_delivered, dwell_time, soc-vs-time curves, dwell~energy corr. (ev.caltech.edu/dataset; registration)
- **ca_dmv_av** — California DMV AV disengagement + collision reports (20k encoded, 2022-24) → incident_rate, disengagement_freq, incident_type_mix, edge_case_library. (public record)
- **eia_grid** — EIA Hourly Grid Monitor, TVA region for Nashville (4367 hourly, 2024 H1) → grid_demand_profile, carbon_intensity, tariff_window_shape. (eia.gov; LIVE needs an API key)
- **nyc_tlc** — NYC TLC trip records (3.5M real trips, Jun 2024) → arrival rate by hour/DoW, trip duration. (public domain)
- **noaa_nws** — NOAA NWS api.weather.gov, Nashville KBNA (500 obs) → ambient_temp, weather_severity, precip, wind. (public, NO key)
- **nrel_fleet** — NREL Fleet DNA (15k) → daily_energy_kwh, duty_cycle, service_interval. (public)
- **charger_reliability** — UC Berkeley / JD Power / ChargerHelp composite (40k) → charger_fault_rate, fault_mode_mix, MTBF. (escholarship.org)

⭐ DECISION (Claude's call, 2026-06-19, Chase deferred): keep HYBRID — calibrate randomized generators TO these real corpora (real params + logic, but randomizable for novel stress) NOT verbatim replay (replay can't be perturbed). For live real-time fidelity, POST real APIs through the **U4 `ottoq-ingest`** seam (brain unchanged). NO live API keys in Vault (only `ottoq_anon_key`) — the offline ingest used the sources directly; LIVE pulls (EIA / ElectricityMaps / ACN registration) need keys → Chase provisions, then Claude wires live pollers → ottoq-ingest. Do NOT need new data to proceed; fitted corpora already ground the variability.

⭐ FRONTIER DEEPENING (Phase 3, fold into U5 scorekeeper): ingest MORE records (full ACN 130k, full-year NOAA via CDO, fuller NYC TLC/DMV) and ESPECIALLY more cross-variable CORRELATIONS — only 1 fitted now = THE gap. Independent random knobs ≠ realistic stress; reality CO-MOVES (heat wave → AC load↑ + charge-rate↓ + grid price↑ + arrival shift). Correlated variability = what makes the twin a statistical clone + the OTTO-Q-beats-FIFO/greedy proof credible for the demo. Relates to [[otto-q-otto-twin-status]], [[scout-external-tools]], [[sim-visual-and-vehicle-flow]].
