---
name: project_deployment_bar
description: "STANDING (Chase 2026-07-19): what 'production' means for OTTO-Q right now — a defensible real backend with the most realistic hardware-integration path possible, NOT a hardened or secured system. Security explicitly deferred."
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-23T19:05:49.331Z
---

**THE BAR, in Chase's words (2026-07-19):** *"We need the most realistic hardware integration path possible, so that when an OEM or an Investor says 'yeah, let's go,' we can plug the vehicles in — or our system can receive the most realistic lines of communication possible. This allows us to say we're ready and our software is fully built, not that it's just some type of simulation and visual layer and the entire backend still has to be thoroughly built out."*

**The test to apply to every decision:** would a skeptical OEM integration engineer, poking at this for an hour, conclude it is a real backend with real communication seams — or a visual shell? Chase's exact fear: *"Nothing vapor or just visual."* And: *"everything works start to finish as we describe and portray our technology to be. Nothing falls short in a major way, and everything defensibly works and shows output from a very substantial standpoint."*

**Explicitly NOT the bar right now:**
- **Security is DEFERRED by founder decision.** *"I don't give a crap about security right now. No one is on our backs... when we get to that point we'll have engineers that fully harden the backend. Put that permanently in your back pocket."* Re-activates only on OEM/investor technical due-diligence, real operator data, multi-tenant work, or a real vehicle connecting. State captured in task SEC-1 so it needs no re-discovery. Do NOT re-raise it as a blocker.
- **Bulletproof engineering is not the goal.** Chase expects that *"once we have funding and actual engineers, they might tear it apart and systematically rebuild the architecture."* So optimise for CREDIBLE ARCHITECTURE AND HONEST CAPABILITY, not for code that survives forever.
- **Multi-tenant SaaS comes LATER** — after pilot, and after the V1 investor/OEM-ready demo. Do not build tenant isolation now.

**Sequence Chase named:** pitchable demo + V1 investor/OEM visualizations → OEM/investor sign-off + funding → pilot with real vehicles → multi-tenant SaaS.

**⭐ SECURITY-TIMING RE-CONFIRMED (Chase 2026-07-23, in response to the OEM-diligence audit's 2 security BLOCKERS — anon DML on 28 tables + 199 anon-EXEC SECDEF fns + open blackbox export + audit-trail-tamperability):** *"Not currently worried about safety. Just wanna make sure all functionality works and we're building to production. And then once we are ready for final demos and everything works great including motion, Isaac Sim, full NVIDIA AI operation and functionality, then we will worry about safety and security right before pitching and OEM conversations."* → DO NOT touch security/RLS/anon-grants now, even the audit-integrity carve-out. The gate re-activates ONLY at "right before pitching/OEM," AFTER motion + Isaac Sim + full NVIDIA AI all work great. Disclose honestly if asked; never spend build cycles on it now. **Build order he set 2026-07-23:** finish the real-telemetry CUTOVER completeness FIRST (service-completion ingest stream + split service-pipeline fabrication from orchestration + energy TWIN-% leak + one proven external-mode run w/ receipt), THEN A/B re-cert the numbers under the fixed CRN stack. Functionality → production; safety/security last.

**What this prioritises:** real protocol shapes and integration seams (OCPP, OEM/fleet APIs, telemetry ingest, comms bus, EMS signalling) that a real counterparty could connect to; end-to-end flows that actually complete; and defensible outputs with real numbers. **What it de-prioritises:** auth/RLS hardening, tenant isolation, and polish that is only visual.

**PRODUCTION-PATH PARITY (2026-07-20, AP-7b B15):** the live production tick `ottoq_world_advance` (cron-driven, real wall-clock, `run_by='production_live'`) now runs the IDENTICAL full certified cycle as the demo tick loop `ottoq_sim_advance_tick_world` — the appointment/overnight brain (AP-3/4/5/7), **energy orchestration (the #1 demand-shave edge, previously ABSENT from production)**, admission, comms, service/overnight advancers, exception/fault handlers, and decide+recall — each in a defensive BEGIN/EXCEPTION so one subcall never aborts the tick. Verified on a live production run: 91 telemetry + 11 charge + 38 decisions + 8 energy_commands / 4 ticks, ZERO subcall warnings. So "the demo proves X" and "production does X" are now the SAME code path — the direct answer to "nothing vapor." The one honest caveat: `ottoq_sim_advance_deployed_telemetry` still SIMULATES vehicle motion; that is exactly the seam a real OEM telemetry feed replaces at true integration (a swap of the telemetry source, not a rebuild of the brain).

**COROLLARY — the honesty stakes are HIGHER here, not lower.** Because Chase will repeat what I tell him to an OEM, any capability I overstate becomes his credibility loss in a due-diligence room. Grade seams as REAL / PARTIAL / COSMETIC / ABSENT and never let "the table exists" pass as "it works". See [[feedback_gap_ownership]] and the fabricated-provenance precedent in [[project_appointment_depot_doctrine]] (`ottoq_visit_needs.source` was being relabelled to imply a vehicle handshake that never happened).

Links: [[project_appointment_depot_doctrine]], [[project_comms_lines]], [[project_ottoq_logic_completeness]], [[project_architecture_separation]].
