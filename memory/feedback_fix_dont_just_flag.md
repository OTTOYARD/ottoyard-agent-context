---
name: feedback_fix_dont_just_flag
description: "⭐⭐STANDING: identifying a defect is only half the job — go into the code/schema, FIX it, then retest and confirm the fix cleared"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 432c576c-ab48-4d5f-8c83-c6ce3cb594e6
  modified: 2026-07-29T14:55:14.690Z
---

Chase, 2026-07-29: *"Make sure that just because you identify something you don't stop there with just calling it out. Once you identify something, I need you to go into the repo or schema and code to fix it so that we can then retest and confirm the fix has been cleared with your coding. We can't just call things out and then leave them. Make sure we're actually accomplishing things through code and build out."*

**Why:** he is paying for shipped behavior, not audit reports. A findings list that ends at "here's what's broken" transfers the work back to him — the opposite of what a CTO is for. Several of my turns had ended with a well-evidenced defect list and no code change.

**How to apply:**
- Every defect I surface gets a fix in the same or the very next turn, in the actual schema/repo — not a recommendation.
- Then **retest and show the delta** (before → after numbers from the live system), the way the gate-queue fix went 62 → 0.
- If a fix genuinely must wait (blocked, needs his decision, or too risky mid-run), say so explicitly and say when it lands — don't let it drift into a standing "outstanding" list.
- Batch related fixes rather than reporting them one at a time.

Related: [[feedback_gap_ownership]] (never let him find the gap first), [[feedback_confirm_every_logic_point]] (READ→REASONED→PLANNED→BUILT→CONFIRMED — "BUILT" is the part this feedback is enforcing).
