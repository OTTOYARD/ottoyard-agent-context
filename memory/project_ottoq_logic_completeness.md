---
name: project_ottoq_logic_completeness
description: OTTO-Q decision-logic completeness ≈90% (2026-06-22) + production-tightness state — the scorecard before migrating to the twin/integrations
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

**Chase's bar (2026-06-22): get ≥90% of OTTO-Q's decision LOGIC in place + production-tight BEFORE migrating to the twin + integrations there.** Current state on otto-q-core (`gxdrcyphqjzjsuhxuqtg`):

**Logic completeness ≈ 90%, all validated at 0-unsafe.** Built + verified: L1 52-rule safety shield · charger-aware stall/charge assignment (cuOpt L2) · energy orchestration with **proven ~42% demand-charge cut (~$10k/mo/depot)** · day/night-aware deploy (TLC curve, overnight pre-stage + AM commute surge) · service manifests (must-do fast-turn + deferred **overnight drain** via separate tech-crew capacity) · **full exception triad** — charger-fault auto-reroute (S3a) + vehicle-fault tech-approval (S3b) + bay-fault auto-reroute (S3c) · **predictive arrival forecast + forecast-aware BESS pre-position** (P1/P2 — the "know SoC+ETA in advance" edge). Surface: ~50 OTTO-Q brain fns + 65 twin-sim + 12 substrate.

**Production tightness:**
- ✅ **Logic integrity tight** — health smoke + extensive multi-hour runs this session: 0 unsafe, 0 error/critical events (the one "critical" seen = a legit `vehicle.exception_auto_staged`, not a bug), decisions logging (~423/6-tick run), 14+ event types flowing. The brain is coherent after ~22 session migrations.
- ⛔ **Security is the real deploy gate (OQ-7, needs Chase's policy sign-off — do NOT enable RLS blind, it'll break the live cron/sim):** security advisor = 18 ERROR / 483 WARN — **13 tables RLS-disabled**, 34 always-true RLS policies, **228 SECURITY DEFINER fns reachable by anon/authenticated**, 219 mutable `search_path`. This gates EXTERNAL exposure, not internal logic.

**Remaining to fully close the logic (the last ~10%, all enhancements not blockers):** S2b-arrival (generate the service manifest at ARRIVAL not charge-complete, for full upfront sequencing) + deeper decide_tick §4/§5 manifest routing; broaden predictive pre-position beyond the conservative BESS band; an integrated full-day capstone scorekeeper. **Integration surface that rightly comes WITH the twin migration:** OQ-1 (canonical domain model + dialect adapters for real OEM telemetry), OQ-6 (25-tool action API + UI contracts for the thin-client UIs), OQ-7 (RLS). See OTTOQ_VISUAL_LAYER_PLAN.md §7 + [[project_energy_e4_charging_fix]], [[project_depot_ops_model]], [[feedback_realism_standalone]].

**Patch technique that worked all session for editing proven functions safely:** server-side `replace()`/`regexp_replace()` on `pg_get_functiondef(oid)` + guarded `EXECUTE` (occurrence-count guard, abort if ≠1) — zero transcription risk on the 234-line `decide_tick`. Gotchas hit + fixed: event `actor_type` must be in the catalog set (use `command_center_operator`, not `depot_technician`); `data_source` ∈ {production,twin,replay,shadow}; new `event_type`s must be INSERTed into `ottoq_event_types_catalog` (FK); Supabase `execute_sql` runs multi-statement as ONE txn (a trailing error rolls back the whole batch incl. ticks) + returns only the LAST statement's rows.
