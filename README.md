# OTTOYARD — Agent Context Repository

**This repository contains no application code.** It is the shared brain for autonomous coding
agents working on OTTOYARD. Clone it, keep it open, and re-read it whenever you are about to touch
something you have not touched before.

**Audience:** Hermes Agent (the autonomous implementation engineer), Claude Code (the reviewing
senior engineer), and any future agent. Written for an AI agent, not a human onboarding doc.

**Author:** Claude Code (Opus), which built the large majority of the OTTOYARD codebase between
April and August 2026, at the direction of Chase Ballenger (founder/CEO).

**Date of this snapshot:** 2026-08-08.

---

## Read in this order

| # | File | What it gives you |
|---|---|---|
| 0 | [AGENTS.md](AGENTS.md) | **Read first, every session.** The operating contract: what you may do, what you must not do, how to ship. |
| 1 | [docs/00_PRODUCT_CONTEXT.md](docs/00_PRODUCT_CONTEXT.md) | What OTTOYARD is as a business, who the customer is, why the software exists, what "done" means commercially. |
| 2 | [docs/01_SYSTEM_MAP.md](docs/01_SYSTEM_MAP.md) | The five systems, every repo, every database, every URL. **The "where do I find X" file.** |
| 3 | [docs/02_OTTOQ_ARCHITECTURE.md](docs/02_OTTOQ_ARCHITECTURE.md) | OTTO-Q: the decision brain. Tick loop, L1/L2/L3, safety shield, the five decision types. |
| 4 | [docs/03_OTTOTWIN_ARCHITECTURE.md](docs/03_OTTOTWIN_ARCHITECTURE.md) | OTTO-TWIN: the hyper-real world model. Variability engine, visual layer tiers, motion. |
| 5 | [docs/04_ORCHESTRA_AND_PULSE.md](docs/04_ORCHESTRA_AND_PULSE.md) | The two cockpits and the **ecosystem tie-in mandate** (your first major lane). |
| 6 | [docs/05_DATABASE.md](docs/05_DATABASE.md) | Supabase projects, schema split, cron, the RPC surface, and the traps that have bitten us. |
| 7 | [docs/06_DOCTRINE.md](docs/06_DOCTRINE.md) | **Binding.** The founder's rulings on how the system must behave. Violating one is a defect, not a preference. |
| 8 | [docs/07_NVIDIA_AI_LAYER.md](docs/07_NVIDIA_AI_LAYER.md) | cuOpt, Nemotron, the frontier stack, and the roadmap of NVIDIA tooling. |
| 9 | [docs/08_DEVELOPMENT.md](docs/08_DEVELOPMENT.md) | How to run, test, migrate, and verify each app. |
| 10 | [docs/09_AGENT_OPERATING_RULES.md](docs/09_AGENT_OPERATING_RULES.md) | Autonomy policy, model-rotation policy, PR protocol, collision avoidance with Claude. |
| 11 | [docs/10_KNOWN_ISSUES.md](docs/10_KNOWN_ISSUES.md) | The live defect and gap register. |
| 12 | [docs/11_BACKLOG.md](docs/11_BACKLOG.md) | Ranked work. **Start here after you have read the rest.** |
| 13 | [docs/12_CREDENTIALS.md](docs/12_CREDENTIALS.md) | What access exists, what you must ask for, and the exact words to ask with. |
| 14 | [docs/13_HISTORY_AND_LESSONS.md](docs/13_HISTORY_AND_LESSONS.md) | Incidents and the rules they produced. Reading this prevents repeating a week of pain. |
| 15 | [docs/14_GLOSSARY.md](docs/14_GLOSSARY.md) | Every term and code name. |
| 16 | [docs/15_UNCERTAINTIES.md](docs/15_UNCERTAINTIES.md) | **What Claude does not know.** Do not treat gaps here as settled. |

---

## `memory/` — the raw distilled corpus

`memory/` holds **79 verbatim memory files** written by Claude Code across four months of building
this system, plus `memory/INDEX.md` which is the one-line index of all of them.

These are the primary sources. `docs/` is synthesis on top of them. When `docs/` and `memory/`
disagree, **check the dates** — the memory files carry `modified:` timestamps in their frontmatter,
and the newer one usually wins.

**Critical caveat, applies to every file in this repo:** these are point-in-time observations. A
memory that says "function X does Y at line 40" was true when written. **Verify against the live
database or the current code before you assert it as fact or build on it.** Several memory files
are themselves records of Claude discovering that its own earlier belief was false — that pattern
is the norm here, not the exception.

---

## The single most important thing to understand

OTTOYARD is not a CRUD dashboard. It is an attempt to build the **operating system for autonomous
vehicle depot infrastructure** — the layer that takes a fleet of robotaxis and manages the entire
depot lifecycle (return → reserve → arrive → charge → clean → service → inspect → redeploy, day and
night) with no human orchestrating each vehicle and nothing breaking.

Everything you build should be judged against: *would this still be the right architecture when it
is running a real Nashville depot with real Waymo vehicles and an OEM engineer is auditing why a
decision was made?*

---

## Keeping this repo current

If you learn something durable — a defect root cause, a founder ruling, a structural fact about the
system — **add it here**. Put durable facts in `docs/`, and open a PR. Do not let knowledge live
only in a chat transcript. That is exactly the failure mode this repo exists to prevent.
