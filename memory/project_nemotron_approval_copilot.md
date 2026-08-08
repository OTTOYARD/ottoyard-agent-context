---
name: project_nemotron_approval_copilot
description: "NVIDIA Nemotron as the SUPERVISOR'S CO-PILOT for the in-depot reassignment safety gate (Chase 2026-07-24): reviews each pending approval, writes an approve/hold/reject recommendation + rationale + risks, NEVER decides. Live-verified end-to-end."
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-25T01:59:46.057Z
---

**WHAT + WHY:** ties the AI story to the safety gate ([[project_indepot_reassignment_gate]]) instead of trading them off. When OTTO-Q wants a discretionary in-depot reassignment it's DENIED + queued to `ottoq_ops_approvals` (human owns the call). This co-pilot hands the technician/supervisor a reasoned second opinion; the human still decides.

**BUILD (2026-07-24):** edge fn `ottoq-approval-copilot` (v1, verify_jwt=true). Input `{depot_id?, sim_run_id?, approval_type='indepot_reassign', limit=8, dry_run?}`. Per pending approval: gathers vehicle state/SoC + proposed change + depot context (free DCFC, inbound_forecast.pressure from the run payload), calls Nemotron `nvidia/nemotron-3-ultra-550b-a55b` (reuses the copilot's key-candidate + NIM call pattern), parses STRICT JSON `{recommendation: approve|hold|reject, confidence, rationale, risks[]}`, and **merges it into `payload->'copilot'` — NEVER touches status/decided_by/decided_at.** Conservative system prompt (in-motion/mid-service vehicle should rarely be re-routed).

**LIVE-VERIFIED end-to-end:** vehicle in_service_bay → guard denied + queued approval 7e5176e3 → co-pilot returned `hold` (conf 0.65) with a correct rationale ("in a service bay… moving to DCFC disrupts service work, benefit unclear at 64% SoC") + risks list, model stamped, **status still 'pending', decided_by NULL** (human decision untouched). Cleaned up after.

**REMAINING (not built):** trigger the co-pilot automatically on approval creation (event/cron) + surface the recommendation in the Pulse/Orchestra approvals UI. Standing rule: the co-pilot ADVISES; the wall guard + human ENFORCE. Links: [[reference_nvidia_ai_integration]], [[project_indepot_reassignment_gate]].
