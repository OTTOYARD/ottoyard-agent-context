---
name: project_legtype_abort_root_cause
description: "🚨2026-08-01 ROOT CAUSE FOUND+FIXED: one unmapped twin service code (interior_inspection) hit a CHECK constraint and aborted the ENTIRE ottoq_decide_tick transaction every tick — zero stall assignments — while cron reported 'succeeded'. The twin's service vocabulary is OPEN; OTTO-Q's leg_type vocabulary is CLOSED; the seam between them must be TOTAL."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-01T16:54:40.155Z
---

# 🚨 The bug that was silently killing every decision

**SYMPTOM:** `ottoq_decide_tick` produced **ZERO** `stall_assignment` decisions, tick after tick,
despite 24 vehicles waiting at the gate and 24 cuOpt proposals pending. Meanwhile
`cron.job_run_details` reported the metronome job as **`succeeded`** every single time.

**FOUND VIA:** `get_logs(postgres)` — Chase's instruction "check the postgres logs for metronome
decide." The warning repeated on every tick:
```
WARNING: metronome decide <run> failed: new row for relation "ottoq_itinerary_legs"
violates check constraint "ottoq_itinerary_legs_leg_type_check"
```

**ROOT CAUSE:** `public.ottoq_plan_visit_itinerary` has **three** leg-writing loops:
| loop | atoms | how it set `leg_type` | safe? |
|---|---|---|---|
| bay (ln 92-111) | `concurrency='bay'` | `CASE … ELSE 'service' END` | ✅ **total** |
| cabin/ext/digital #1 (ln 61-71) | `concurrency IN ('cabin','exterior','digital')` | `v_a->>'svc'` **RAW** | ❌ |
| cabin/ext/digital #2 (ln 76-86) | same | `v_a->>'svc'` **RAW** | ❌ |

The two raw loops worked **only by coincidence** — most twin service codes happen to spell valid
`leg_type` values. `interior_inspection` (100 atoms live) does not. One rejected row →
`ottoq_decide_tick` runs in ONE transaction → **every charging decision in that tick rolled back.**

⚠️ **`perimeter_walkaround` was a SECOND, still-unexploded instance of the identical bug** — found
only because I probed the full `svc` vocabulary instead of stopping at the first offender.

## ⭐ THE DOCTRINE THIS ESTABLISHES
**The twin's service vocabulary is OPEN** — `twin.ottoq_sim_generate_service_manifest` can mint any
code. **OTTO-Q's itinerary vocabulary is CLOSED** — 22 values in
`ottoq_itinerary_legs_leg_type_check`. **Therefore every seam that carries a twin code into an
OTTO-Q column MUST be a total function.** A partial mapping is not a style issue here; it is a
whole-decision-loop outage waiting for the twin to add one service.
This is a concrete instance of [[project_realdata_ingestion_seam]] — and the same rule applies when
REAL telemetry replaces the twin, because real fleets will emit codes we never enumerated.

## ✅ FIXED 2026-08-01
`public.ottoq_svc_to_leg_type(text)` — IMMUTABLE, service_role only, backup at
`ottoq_schema_snapshots` snapshot_id **6161** (md5 `62fe5639df8d43f26ebb67dcf504407d`).
Passes through the 10 codes that are already valid leg types; maps `exterior_wash`→`wash`,
`interior_deep_clean`→`detail`, `interior_inspection`→`inspect`; **`ELSE 'service'`** catches
everything else. The migration guards on the source md5 and asserts all 4 replacements landed,
so a drifted source aborts instead of half-patching.

**Losslessness:** the true twin code is preserved in `ottoq_itinerary_legs.duration_basis->>'atom'`
(⚠️ the column is **`duration_basis`**, NOT `payload` — my migration comment says `payload`, which
is wrong; the code is correct).

**VERIFIED on live run 9ea73866 (seed 424242), 7 ticks:**
- mapper is **total**: 18/18 probe values pass the CHECK, including `NULL` and an invented code
- **53 legs written** across 7 leg types (was: 0, transaction aborted)
- **27 `inspect` legs carrying `atom='interior_inspection'`** — each one would have killed a tick
- no further `metronome decide … failed` warnings
- STOP contract verified end-to-end: `archived=true, blackbox_ready=true, seed 424242 replayable,
  depot_reset_to_empty=true`

## 🔴 THE REAL LESSON — silent degradation, again
`ottoq_demo_metronome` wraps decide in `EXCEPTION WHEN OTHERS THEN RAISE WARNING`, so a
**100%-failing decision loop looked like a healthy green cron job.** This is the same failure shape
as: signing that never signed, audit flags green because the data was deleted, and a purge that ate
its own archive. **`cron.job_run_details.status='succeeded'` proves the procedure returned — NOT
that it did anything.** Always read `get_logs(postgres)` for WARNINGs before trusting a green run.

## ⏭️ STILL OPEN (found while verifying, NOT fixed)
1. **cuOpt LP still not consumed.** v20 is deployed and fast (6s → 0.3-2.6s), returns HTTP 200 —
   but the only proposals recorded were `source='greedy_constrained'` (the edge function's
   FALLBACK) and both went `superseded`. The LP path is returning nothing. Next step: read the
   function's console output, not just the request log. See [[project_nvidia_layer_truth_2026_07_30]].
2. **`ottoq_purge_prior_runs` can NEVER delete events** — `WARNING: table ottoq_events failed:
   ottoq_events is append-only. Operation DELETE rejected`. The retention worker has been unable to
   touch the 9 GB table this whole time. Directly relevant to [[project_db_capacity_ceiling]].
3. **`purge_prior_runs: table ottoq_active_sim_runs failed: cannot delete from view`** — the generic
   sweep tries to DELETE from a view every run.

Links: [[project_ottoq_twin_boundary]], [[project_runtime_cadence_doctrine]],
[[project_db_capacity_ceiling]], [[reference_db_surgery_lessons]].
