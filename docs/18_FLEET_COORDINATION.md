# 18 — Fleet coordination protocol

**Effective 2026-08-08, founder-directed.** OTTOYARD is now worked by a **fleet** of autonomous
agents running continuously, not a single agent. This file is the traffic law. Every agent in the
fleet must follow it; a violation is a defect regardless of how good the code was.

---

## 1. The fleet

| Agent | Owns the OUTCOME of | May touch |
|---|---|---|
| **CONDUCTOR** | Work assignment, the claim board, all founder communication | GitHub issues, the context repo |
| **LANE A — Ecosystem** | One run, one clock, three lenses: the Twin cockpit, OTTO-PULSE and OrchestrAV depicting the same live truth, with OTTO-Q visibly the orchestrator | `ottoyarddepot-sim`, `ottoyard-field-ops`, `ottoyard-OTTO-Q`, read-only RPCs in `otto-q-core` |
| **LANE B — Twin realism** | The hyper-real world model: motion, geometry, variability, start-of-run variable selection, the photoreal tier | `ottoyarddepot-sim`, twin-schema work in `otto-q-core` |
| **LANE C — Scheduling core** | The forward calendar actually working: bay binding, arrival forecast prerequisites, completion write-back, the appointment loop | `otto-q-core` (this lane is the primary DB owner) |
| **LANE D — Intelligence** | cuOpt pointed at the right problem, Nemotron's advisory loop, the frontier service | `ottoq-intelligence`, AI-seam work in `otto-q-core`, edge functions |
| **REVIEWER** | Adversarial review of every substantive PR before it is offered to the founder | read everything, write nothing but review comments |

**Lanes own outcomes, not repos.** Two lanes may touch the same repo; the claim board (below) is
what prevents collision, not repo boundaries.

## 2. The pipeline — the graph every task moves through

No stage may be skipped. A task that fails a gate goes back, it does not go around.

```
INTAKE      what exactly is the outcome? what is the evidence it will need?
PLAN        smallest honest slice; which files; which models per step
CLAIM       post the claim (see §3) BEFORE touching anything shared
BUILD       cheap/open-source models by default (see §5)
SELF-TEST   run it: build, tests, sim run, real queries — the agent's own proof
ADV-REVIEW  a DIFFERENT model than the author tries to refute the work
EVIDENCE    assemble the numbers/screenshots/row counts that prove it
PR          open it with the evidence, what was NOT verified, and blast radius
REPORT      one message to the Conductor; the Conductor decides if Chase hears
```

**The ADV-REVIEW gate is not optional and never uses the authoring model.** On this project,
three of four adversarial reviewers once correctly rejected a plan that two independent planners
had both approved.

## 3. The claim board — how agents avoid colliding

Claims are **GitHub issues**, because every agent can read and write them and they survive any
session dying. Post the claim **before** touching the shared thing, close it when done.

| Shared thing | Claim | Where |
|---|---|---|
| A migration number | Issue titled `CLAIM MIGRATION 00NN — <slug>` | `otto-q-core` issues |
| A live sim run | Issue titled `CLAIM SIM-RUN — <lane> — <purpose>` | `otto-q-core` issues |
| A file two lanes both need | Issue titled `CLAIM FILE — <path>` | that repo's issues |
| A work item | Issue titled `LANE <X>: <outcome>` — the Conductor assigns these | context repo issues |

**Hard database traffic rules — these are absolute:**

1. **One live sim run at a time, fleet-wide.** Before starting: check
   `select id, status, run_by from ottoq_sim_runs where status in ('running','paused')` **and** the
   claim board. If a run exists that is not yours, you wait. `ottoq_start_demo_run` **aborts** other
   runs — starting one over someone else's claim destroys their evidence.
2. **Migration numbers are claimed before they are written.** Two agents authoring `0023` is a
   guaranteed conflict.
3. **Only Lane C applies migrations, and only to preview branches.** Nobody applies to production —
   the founder merges, then application happens from the merged file.
4. **Never touch cron job 12.** It is the run engine.
5. **Certification runs cost database headroom and real money** (cuOpt is billable per solve). Note
   the cost in the claim.
6. **Branch naming carries the lane:** `hermes/<lane>/<slug>` — e.g. `hermes/a/run-context-banner`.

## 4. Communication protocol — the founder's phone is the constraint

Chase checks Telegram periodically. He is non-technical on code. **Everything he receives is
plain-language, under ~120 words, and ends with exactly one action for him.** Only the Conductor
messages him.

Only four message types exist:

**`[PR READY]`** — a PR passed review and needs his merge.
> *[PR READY] otto-q-core #24 — Vehicles now keep their assigned bay reservation instead of
> giving it up mid-charge. Verified on a full 3-hour sim: 11 of 14 holds now bind (was 0 of 18,
> ever). No behaviour changes outside the replanner. Risk: low — one function, md5-guarded.
> → Merge when ready.*

**`[DECISION NEEDED]`** — a genuine founder call: doctrine, money, product behaviour, or
irreversibility. Always with a recommendation.
> *[DECISION NEEDED] cuOpt vs CP-SAT: on our benchmark the free open-source solver beats NVIDIA's
> paid one for overnight scheduling. Keep NVIDIA in the loop for the pitch story, or ship the
> better solver? My recommendation: ship CP-SAT, keep cuOpt where it genuinely wins. → Reply A
> (CP-SAT) or B (cuOpt).*

**`[BLOCKED]`** — the fleet cannot proceed without him (access, credential, an external system).
Exact steps included.

**`[DIGEST]`** — a weekly summary: merged, in flight, found, spent. Nothing needing action.

**Everything else is silent.** Progress, retries, internal reviews, model switches — none of it
reaches him. Clarifying questions are front-loaded: ask everything at task intake, not mid-build.

## 5. Model routing — the standing economic policy

**Default is open-source. Frontier models are an escalation, not a default.**

| Tier | Models (route to whatever Hermes offers in this class) | Use for |
|---|---|---|
| **T1 — cheap/fast** | small Qwen / DeepSeek / GLM class | file reading, grep-level search, mechanical edits, lockfiles, formatting, first-draft tests |
| **T2 — workhorse** | Qwen3-Coder / DeepSeek-V3 / Kimi K2 / GLM-4.5 class | **most coding, SQL, debugging, refactors, test suites — the default for BUILD** |
| **T3 — heavy open** | the largest open reasoning models available | architecture drafts, tricky root-cause work, ADV-REVIEW of normal PRs |
| **T4 — frontier (Opus/Fable, ultracode)** | Claude Opus 5 / Fable 5 | **Only on triggers, never by default** |

**T4 triggers — all of them, and only them:**
- Two genuine attempts at T2/T3 failed and the failure is not understood.
- A change touches the safety shield, the deploy path, the booking calendar's integrity
  constraints, or anything in `docs/06_DOCTRINE.md`.
- ADV-REVIEW of a high-blast-radius PR (migrations that replace functions, anything touching
  `ottoq_decide_tick`, the metronome, or the purge).
- A `[DECISION NEEDED]` is being drafted — the framing that reaches the founder is worth the best
  model.

**Two things never economised, at any tier:** adversarial review, and any claim about the live
database. Both have already cost this project real time.

**Log the routing.** Each PR notes which tier built it and which model reviewed it. When a T2 model
produces a wrong load-bearing claim, that is a signal to adjust the routing, and it is only visible
if recorded.

## 6. Standing collision rules with the other actors

- **Lovable commits straight to `main`** on the three front-end repos as `gpt-engineer-app[bot]`,
  and pulls merges back automatically. Fetch before pushing, always. A `lovable-sync-<timestamp>`
  branch appearing means a divergence — tell the Conductor, do not delete it.
- **Claude Code sessions** may also be working, as `claude/<slug>` branches. Same claim board, same
  rules. Claude is additionally the escalation reviewer of last resort.
- **Salvage branches** (`salvage/*`) are rescued work awaiting founder review. Read-only.
- **Clone fresh, always.** No persistent local working copies exist, by founder decision. Push your
  branch before your session ends, every session — six branches were nearly lost to this once.

## 7. Scaling discipline

**Start with the Conductor plus two lanes (A and B). Add C and D only after the first week
produces clean, mergeable PRs.** Every additional concurrent agent multiplies claim-board traffic,
database contention, and spend. The goal is a steady stream of small, proven PRs on the founder's
phone — not maximum parallelism.
