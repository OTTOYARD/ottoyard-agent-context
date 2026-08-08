---
name: reference_parallel_sessions_and_graphs
description: "Operating model for multi-session work (hub-and-spoke, NOT permanent topic chats) + the researched 2026 verdict on 'graphs' for agents: mostly DON'T adopt — markdown+wikilinks already IS the graph, and Anthropic REMOVED code-graph indexing from Claude Code."
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
  modified: 2026-07-26T15:27:38.994Z
---

## PARALLEL SESSIONS — the operating model that worked (2026-07-26)

**HUB-AND-SPOKE, not permanent topic chats.** One main session is the integrator (owns doctrine, memory, merges, the final story). Spokes are TASK-SCOPED sessions spawned for one question, which end when answered.

**PROVEN 2026-07-26:** two spawned sessions (`local_b3b38afc` funnel-gap, `local_f1c3569b` tick-cost) each went deeper than the main session had, and **caught 4 substantive errors in the main session's own work** (the retracted 910100 cert, the retracted metronome 35.7%, a mis-evidenced fence test, a wrong funnel tell). **A fresh session auditing the main session's claims is the single highest-value use of parallelism.**

**CONVERGENCE IS FREE — DO NOT ASK CHASE TO WORK INSIDE EACH CHAT.** `mcp__ccd_session_mgmt__list_sessions` / `search_session_transcripts` / `list_events` let the hub read any other session's transcript and conclusions directly. Read them, correct memory, report once.

**HARD CONSTRAINTS that make permanent topic chats a bad idea:**
1. **ONE shared Supabase project** (`gxdrcyphqjzjsuhxuqtg`). Two sessions doing the `pg_get_functiondef` → `replace()` → `EXECUTE` splice on the SAME function = silent lost update, last writer wins.
2. **`ottoq_one_running_run_per_depot`** — only ONE running sim run per depot, and there are only 2 depots. Two sessions running sims collide (hit repeatedly 2026-07-25/26).
3. **MEMORY.md + memory files are a single shared surface** — parallel writers fragment the one thing that is working.

**RULES:**
- **Migrations serialize through the hub.** A spoke investigates freely and proposes DDL; the hub applies it — unless explicitly handed exclusive ownership of a named function set.
- **Any spoke that runs a sim MUST use `run_by='cert_harness'`** (the only value the metronome excludes) and must stop+reset what it starts.
- **Disjoint surfaces CAN run truly parallel:** the TypeScript renderer repo (`~/Desktop/OTTOYARD/ottoyarddepot-sim`) shares nothing with the DB. That is the one safe standing split.
- Every spoke prompt must be SELF-CONTAINED: project id, measurements already taken, hypotheses ALREADY REFUTED (so it doesn't redo them), the DB-surgery rules, cleanup obligations.

## "GRAPHS" — RESEARCHED VERDICT (2026-07-26): mostly DON'T

**"Loops are out, graphs are in" is NOT accurate for a coding agent.** Four different things get called "graphs":

1. **Graph ORCHESTRATION (LangGraph 1.0 GA 2025-10-22; MS Agent Framework GA 2026-04).** Real and mature, but earns its keep only with multi-agent roles, inspectable branching, or pause/resume. A single coding agent looping is still correct — that is what Claude Code itself does. ⚠️ **Genuine narrow fit for OTTO-Q's OWN dispatch pipeline**: a state graph can make "every reassignment must call the guard" structurally impossible to skip, vs today where every code path must remember. Worth considering for the PRODUCT, not for how we work.
2. **Graph MEMORY (GraphRAG, Zep/Graphiti, Mem0).** **DO NOT ADOPT.** Our markdown + `[[wikilinks]]` **already IS a hand-built knowledge graph** — files are nodes, links are edges — and because we author edges deliberately it sidesteps the #1 documented failure of automated graph memory: hallucinated relationships. Mem0's OWN paper (arXiv:2504.19413) shows its graph variant beats its non-graph variant by only ~2 points. Vendor benchmarks are self-graded. Our corpus is dozens of files, far under any break-even.
3. **CODE GRAPHS (call/AST/symbol graphs).** ⚠️ **Strongest anti-signal: Anthropic BUILT this into Claude Code and REMOVED it.** Boris Cherny (built Claude Code): early versions used RAG + a local vector DB; agentic search "generally works better... simpler and doesn't have the same issues around security, privacy, staleness, and reliability." **On an actively-rewritten codebase a STALE graph is worse than none — it confidently asserts wrong structure, i.e. re-creates the drift problem one layer down.** (Aider's tree-sitter repo-map + LocAgent arXiv:2503.09089 show graphs DO help *localization* — a different problem from ours.)
4. **Traceability / provenance graphs** — doctrine↔code drift detection. **EXACTLY Chase's problem, and NOT a shipped product** (survey arXiv:2606.04990; Cognition "Agent Trace" RFC 0.1.0 Jan 2026 tracks what an agent wrote, not whether it matches doctrine). Watch, don't adopt.

**WHAT THE RESEARCH VALIDATES THAT WE ALREADY DO:** Anthropic's context-engineering guidance (2025-09) + Chroma's independent "Context Rot" study (2025-07, 18 frontier models: accuracy drops 30-50% well before the advertised context limit; safe usable context often 4-10× smaller than advertised) both endorse **a short index pointing at focused files** over one giant unified store. MEMORY.md + per-topic files is the correct pattern, not something to replace.

**CONCLUSION: the countermeasure to drift is not a graph technology — it is the discipline in [[feedback_confirm_every_logic_point]] + [[feedback_gap_ownership]], plus ADVERSARIAL AUDIT BY A FRESH SESSION, empirically the thing that caught 4 real errors on 2026-07-26.** Institutionalize the audit; do not buy a graph database.

Links: [[feedback_confirm_every_logic_point]], [[feedback_gap_ownership]], [[reference_db_surgery_lessons]], [[project_twin_realtime_clock]].
