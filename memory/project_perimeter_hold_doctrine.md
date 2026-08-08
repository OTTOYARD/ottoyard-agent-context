---
name: project_perimeter_hold_doctrine
description: "⭐DOCTRINE (Chase, 2026-07-28) — temp staging vs perimeter staging serve DIFFERENT PURPOSES, not overflow tiers; current stall picker sorts them backwards"
metadata: 
  node_type: memory
  type: project
  originSessionId: 432c576c-ab48-4d5f-8c83-c6ce3cb594e6
  modified: 2026-07-28T18:39:21.614Z
---

Chase's stated doctrine for the two staging classes (`stalls.staging_role`), given 2026-07-28:

**Temp spots (`staging_role='temp'`, 24 stalls, interior rows)** — short, transient, mid-workflow:
- quick inspection AFTER all services are rendered for the queuing sequence OTTO-Q issued for that vehicle on arrival
- quick reassignment / disruption inside the original OTTO-Q workflow (e.g. charger faults mid-charge → vehicle temporarily staged here)

**Perimeter spots (`staging_role='long'`, 176 stalls, lot-edge ring)** — longer-term:
- longer-term inspection
- **overnight hold**: after the normal overnight return AND all services are handled, the vehicle is staged in *its specific perimeter space* for a few hours until morning dispatch engages ~4:30–6 AM

**Underlying fleet doctrine:** these vehicles are ALWAYS driving unless they (a) need service, (b) must be held when there is very little ride-hail demand, or (c) there is a major issue (e.g. a city-wide recall). Overnight hold is the zero/low-demand case. See [[project_vehicle_first_doctrine]], [[project_depot_ops_model]], [[project_full_service_visit_doctrine]].

## Verified against the live DB (project gxdrcyphqjzjsuhxuqtg), 2026-07-28

ALREADY BUILT and matching the doctrine:
- `ottoq_visit_needs.urgency = 'overnight_hold'` + `dispatch_due_at` is the hold-until timestamp.
- `ottoq_plan_visit_itinerary:113-120` inserts a `stage` leg running from services-complete to `dispatch_due_at`, tagged `{"kind":"flow_contract","reason":"overnight_hold"}` — i.e. hold AFTER services, exactly as Chase describes.
- `ottoq_decide_tick:68-70` excludes a `staged_for_departure` vehicle from the deploy loop while an open `overnight_hold` need has `dispatch_due_at > clock` — the hold is enforced against redeploy.
- `ottoq_decide_tick:56` scales the deploy target by hour-of-day (America/Chicago); `ottoq_sim_auto_dispatch_tick:132-141` does demand-driven overnight surplus recall (knobs: `overnight_recall_start_hour` 22, `overnight_recall_end_hour` 3, hysteresis, `overnight_holdout_pct`).

THE DEFECT (inverse of doctrine): every staging pick sorts `ORDER BY (s.staging_role = 'temp') DESC` — temp ALWAYS preferred, spill to perimeter. Appears in `ottoq_decide_tick` (x2, lines 165/217), `ottoq_book_appointment:112`, `ottoq_sim_prearrival_contracts:65`. So an overnight hold takes an interior temp spot whenever one is free, consuming the 24-spot quick-turnaround buffer that doctrine reserves for post-service inspection and fault re-staging, before spilling to the 176 perimeter spots where it belongs. The sort encodes capacity overflow; the doctrine wants purpose.

OTHER GAPS: the overnight-hold leg carries NO stall binding (`leg_type='stage'`, no stall id); nothing gives a vehicle a *specific* perimeter space (only TTL reservations via `ottoq_reserve_stall`); and that TTL is decorative — nothing expires reservations (62 of 62 held reservations already expired, newest by 13 hours).

`stalls.zone` is NOT the lever — it is read by exactly one function (`ottoq_twin_depot_layout`, a renderer passthrough) and nothing branches on it.
