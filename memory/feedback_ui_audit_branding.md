---
name: feedback-ui-audit-branding
description: Chase wants a future full UI audit + design/branding-consistency pass on both OTTO-PULSE and OrchestraAV
metadata: 
  node_type: memory
  type: feedback
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

Chase (2026-06-17): while wiring the current UIs to OTTO-Q is fine for now, **at some point do a full AUDIT and adjustment of both OrchestraAV and OTTO-PULSE** — streamline, clean up, build out, and optimize the overall design AND function. Explicitly: hunt for "laziness or missed design/functional elements," and make the visual design **consistent with OTTOYARD's HTML branding — the same system used for the website**.

**Why:** these are the two production product UIs (field/depot techs = PULSE; fleet owners = OrchestraAV). They were Lovable-generated and have drift (dead buttons, mock fallbacks, split backends, ~70% simulated OttoCommand tools — see [[otto-q-otto-twin-status]] OQ-6 analysis + Desktop/OTTO-Q V1/OQ6_UI_WIRING_ANALYSIS.md). He wants them to feel polished, coherent, and on-brand, not just functional.

**How to apply:**
- Treat this as a standing quality bar on all UI work — don't ship lazy/half-wired UI; flag gaps proactively.
- The dedicated audit is a SEPARATE pass to schedule after the OQ-6 functional wiring is further along (don't derail current wiring for it unless Chase asks).
- For the audit: first OBTAIN the OTTOYARD website's branding/design system (colors, type, components, logo usage) — ask Chase for the website repo/URL if not available — then reconcile both apps' Tailwind theme + components to it. Repos: OTTO-PULSE = ottoyard-field-ops; OrchestraAV = ottoyard-a4359174.
- Audit scope: dead/unwired controls, mock-vs-real surfaces, inconsistent components, missing states (loading/empty/error), accessibility, responsive, and brand consistency. Produce a findings list + fixes, like the OQ-6 contract-first analysis.
