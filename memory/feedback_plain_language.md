---
name: feedback_plain_language
description: "STANDING — Chase (CEO) wants plain, descriptive language from Claude (frontier CTO); minimal jargon/acronyms he won't know, because heavy technical language makes his decision-making harder"
metadata:
  node_type: memory
  type: feedback
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

Chase (2026-06-23): **"You are the frontier CTO, and I'm the CEO. Talk to me in descriptive but not too technical language, no acronyms I won't understand — that makes decision-making difficult due to the technical language barrier."**

**Why:** Chase makes the business/product calls but isn't a database/ML engineer. When choices are framed in raw technical terms (table names, function names, acronyms like BESS/MPC/CIL/grid_import_kw), he can't weigh them well. He needs the *decision* and the *trade-off* in plain English to steer.

**How to apply:**
- Lead with what it MEANS and what it's WORTH (business/demo impact), not how it's wired.
- Translate every term: "the big on-site battery" not "BESS"; "the depot's peak power draw that the utility bills extra for" not "demand charge / grid_import_kw peak"; "OTTO-Q simulates a few options forward and picks the best" not "twin-in-the-loop MPC."
- When asking him to decide, give 2-3 plain options + a clear recommendation + the trade-off in one line each. Don't make him parse acronyms to choose.
- Code/function names are fine in the *background* (so the work is traceable) but never the headline of an explanation.
- He still wants rigor + honesty (real numbers, what failed) — just delivered readably. See [[feedback_ask_early]], [[feedback_realism_standalone]].
