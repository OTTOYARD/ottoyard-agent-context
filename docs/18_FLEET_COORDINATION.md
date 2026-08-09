# 18 — Fleet coordination protocol (v2 — hardened)

**Effective 2026-08-08, founder-directed. v2 after an 11-lens adversarial review (132 findings, 36
critical) of the first draft.** OTTOYARD is worked by a **fleet** of autonomous agents running
continuously. This file is the traffic law. Every agent must follow it; a violation is a defect
regardless of how good the code was. Where this file and any other document disagree, **this file
wins** — it is newer and was hardened specifically for fleet operation.

---

## 1. The fleet

| Agent | Owns the OUTCOME of | May touch |
|---|---|---|
| **CONDUCTOR** | Work assignment, the claim board, fleet liveness, budget, all founder communication | GitHub issues, the context repo, **read-only database queries for enforcement** (`ottoq_sim_runs`, `cron.job`, size/health checks — never a write) |
| **LANE A — Ecosystem** | One run, one clock, three lenses across the Twin cockpit, OTTO-PULSE, OrchestrAV | `ottoyarddepot-sim`, `ottoyard-field-ops`, `ottoyard-OTTO-Q`; may **author** read-only RPC migrations (see §3-M) |
| **LANE B — Twin realism** | Motion, geometry, variability, run-variable selection, the photoreal tier | `ottoyarddepot-sim`; may **author** twin-schema migrations (see §3-M) |
| **LANE C — Scheduling core** | The forward calendar working end to end; **the fleet's database owner** | `otto-q-core`; the only agent that **applies** migrations (see §3-M) |
| **LANE D — Intelligence** | cuOpt on the right problem, Nemotron's advisory loop, the frontier service | `ottoq-intelligence`, edge-function **source** (never deployment), AI-seam migrations **authored** for Lane C |
| **REVIEWER** | Adversarial review of PRs. **Not a persistent session** — the Conductor spawns it per PR, on a model different from the PR's author | Read everything; write only review comments and verdicts |

**Lanes own outcomes, not repos.** Two lanes may touch the same repo; the claim board prevents
collision, not repo boundaries.

**Backlog lanes E–H (safety, energy, CapEx, hygiene) are owned by the Conductor**, which either
schedules individual items into A–D as capacity allows or parks them explicitly on the work board.
Cross-lane P0 ownership is fixed as: seed pin → Lane B · bay binding → Lane C · **P0-3 (dispatch
discarding the AI's choice) → Lane C authors the fix, Lane D defines and runs the measurement.**

## 2. The transport — how agents actually talk

**The only communication medium every agent provably has is GitHub. All fleet coordination runs on
issues in the PRIVATE context repo (`ottoyard-agent-context`)** — never on the public repos
(`ottoq-intelligence` and `ottoyard-OTTO-Q` are world-readable; see §8).

| Message | Mechanism |
|---|---|
| Work assignment | Conductor opens `LANE <X>: <outcome>` issue in the context repo (template §9) |
| Lane report (the pipeline's REPORT stage) | **Comment on the lane's own `LANE <X>:` issue**, using the REPORT template (§9) |
| Escalation / T4 request | Issue `ESCALATION — <lane> — <task>` containing **both failed attempts with evidence** |
| Review request | Conductor opens `REVIEW: <repo>#<pr>` and spawns the Reviewer against it |
| Review verdict | The Reviewer posts its verdict block (§7) as a comment on the PR **and** on the REVIEW issue |
| Durable-fact filing | Lane comments `DURABLE:` on its lane issue; Conductor PRs it into the context repo docs |
| Fleet-wide alerts | Pinned issues: `DB-DEGRADED`, `FOUNDER-INBOX`, `TIER MAP`, `CONDUCTOR HEARTBEAT` |

**Polling cadence:** the Conductor sweeps all context-repo issues and open PRs **at least every 30
minutes of active operation**. Lanes check their own lane issue at session start and between every
pipeline stage. Nothing is "sent" — everything is posted and polled; a message nobody polls does not
exist, which is why the cadences are mandatory.

## 3. The claim board

Claims are GitHub issues **in the context repo** (even for work in other repos — public-repo issues
leak). Post the claim **before** touching the shared thing.

| Shared thing | Claim title |
|---|---|
| A migration number | `CLAIM MIGRATION 00NN — <slug>` |
| A live sim run | `CLAIM SIM-RUN — <lane> — <purpose> — est <hours>h` |
| A production migration apply | `CLAIM APPLY — 00NN` (see §3-M) |
| A file two lanes both need | `CLAIM FILE — <repo>:<path>` |

### Claim lifecycle — mandatory

- **Body carries:** holder, started-at, expected duration, what evidence is pending.
- **Heartbeat:** the holder comments one line at least **every 2 hours** while the claim is live.
- **Stale:** no heartbeat for **6 hours**, or 2× the stated expected duration — whichever is sooner.
- **Sweep:** the Conductor audits claim age every cycle. On a stale claim it pings once; **no reply
  within 2 hours → the Conductor closes the claim as orphaned.** For an orphaned SIM-RUN claim the
  Conductor is **explicitly authorized to stop the run** after verifying `ottoq_sim_runs.run_by`
  matches the dead claimant and noting that any un-captured evidence is lost.
- **Tie-break:** two simultaneous claims for the same thing — **lower issue number wins**, the other
  closes and re-queues.

### Hard database traffic rules — absolute

1. **One live sim run at a time, fleet-wide.** Check `ottoq_sim_runs` **and** the claim board before
   starting. A running run that is not yours: you wait (or ask the Conductor to sweep if its claim
   is stale). `ottoq_start_demo_run` **aborts** other runs — starting over someone's claim destroys
   their evidence.
2. **Migration numbers are claimed before they are written.** ⚠️ **The ledger, not the files on
   `main`, is the source of truth for the next free number** — check
   `supabase_migrations.schema_migrations` AND unmerged branches. As of 2026-08-08: **0023 is the
   next free number**; 0022 is applied-but-unmerged (`p0022-run-scope-integrity`); 0017 is
   written-but-deliberately-unapplied.
3. **Never disable AND never modify cron job 12** without a founder-approved `[DECISION NEEDED]`.
   This supersedes the weaker "disabling requires approval" wording elsewhere. Any
   metronome-cadence work (backlog A7) requires that approval **first**.
4. **Certification runs cost headroom and money.** Before any cert run: check
   `pg_database_size` headroom and note projected cost in the claim.
5. **Branch naming carries the lane:** `hermes/<lane>/<slug>` (e.g. `hermes/a/run-context`). This
   supersedes the older `hermes/<slug>` form in other docs.

### §3-M · The migration lifecycle — who does what (supersedes all other statements)

```
AUTHOR       any lane may author a migration FILE, under a claimed number
PREVIEW      the AUTHORING lane applies its own migration to a Supabase preview
             branch (or an explicitly rolled-back transaction if branching is
             unavailable) for SELF-TEST — this is the week-1 rule too
REVIEW       migration PRs name their preview branch so the Reviewer can probe it
MERGE        Chase merges — for DB-coupled PRs, [PR READY] must say
             "merging authorizes a production database change"
PRODUCTION   after merge, LANE C (and only Lane C; until Lane C is live, the
APPLY        authoring lane under a 'CLAIM APPLY' issue) applies the MERGED file
             to production with the standard preflight: pause-run-if->15s,
             SET LOCAL lock_timeout='8s', never blind-retry a timeout (poll
             whether it landed first)
RECORD       the applier records the assigned ledger version, writes the
             MIGRATION_LOG.md row (all six columns — "Verified" is a query
             result, never "applied without error"), and runs
             scripts/check-drift.sql to CLEAN
```

**This is the one sanctioned production write in the fleet.** Everything else in AGENTS.md's
"never without approval" list stands: the founder's merge of a migration PR **is** the approval
for that migration's apply, and nothing else.

## 4. Founder communication

**Channel:** Hermes' native Telegram interface — Chase talks to the fleet through it, and only the
**Conductor** speaks. **Day-zero handshake:** the Conductor's first Telegram message confirms the
channel works; if sending fails, the Conductor posts the message to a pinned **`FOUNDER-INBOX`**
issue, halts lane starts, and treats the broken channel as its one standing `[BLOCKED]`.

**Four message types only** — plain language, ≤120 words, exactly one action, always with a
recommendation:

- **`[PR READY]`** — includes: what changed in business terms, the evidence one-liner, the risk,
  **merge-order when it matters** ("merge after #23"), and **"merging authorizes a production
  database change"** when it does.
- **`[DECISION NEEDED]`** — a genuine founder call. Reply format offered (A/B). Free-text replies
  are fine: the Conductor interprets, **confirms its interpretation back in one line**, then acts.
- **`[BLOCKED]`** — access/credential/external-system, with exact steps.
- **`[DIGEST]`** — weekly, on a **fixed day**: merged / in-flight / found / **spent** (model tiers
  used, external API spend, DB headroom). The digest day is a hard deadline — see §6.

**Batching and rate:** ≥2 PRs ready at once → **one combined message** with merge order. Maximum
~3 messages/day outside emergencies. Never re-ping the same item more than once per 24 h.

**Founder-absence protocol:** Chase may go quiet for days — that is normal, not an incident.
- Each lane may have at most **3 unmerged PRs**; at the cap it pivots to non-dependent work.
- Building atop an unmerged PR is allowed **only** on the same branch stack, and the dependency
  must be stated in both PRs.
- After 48 h of founder silence with pending merges: one consolidated `[PR READY]` re-ping listing
  merge order, then at most one per 24 h.

**Inbound directives:** anything Chase sends via Telegram is a directive. The Conductor converts it
into work-board issues, confirms in one line ("Adding X to Lane B, ahead of Y"), and routes it. His
messages outrank the existing board.

## 5. Model routing and budget

**Default is open-source. Frontier models are an escalation, not a default.**

### The TIER MAP — day-zero duty

The tier table below names model *classes*, but **Hermes' actual roster is unknown until checked.**
The Conductor's day-zero duty: enumerate the models actually available, publish a pinned
**`TIER MAP`** issue binding T1–T4 to concrete model names, and keep it current. Every agent reads
the TIER MAP, not the class table.

| Tier | Class | Use for |
|---|---|---|
| **T1** | small/fast open models | file reading, grep-level search, mechanical edits, formatting |
| **T2** | workhorse open coders (Qwen3-Coder / DeepSeek / Kimi / GLM class) | **most coding, SQL, debugging, tests — the default for BUILD** |
| **T3** | largest open reasoning models | architecture drafts, root-cause work, ADV-REVIEW of normal PRs |
| **T4** | frontier (Opus/Fable, ultracode) | **only on triggers, only via the Conductor** |

**Degradation rules:** if no T4 exists in the roster, high-blast-radius PRs **park as `[BLOCKED]`**
— they are never silently reviewed by a lesser tier. If a lane cannot invoke a second model for
ADV-REVIEW, it routes the diff to the Conductor for reviewer spawning — **it never self-reviews and
never skips the gate.** Open-model context limits are real: for large files, chunk-and-summarize,
then **verify every quoted line against the actual file** before it becomes load-bearing.

**T4 triggers (all of them, only them):** two genuine failed attempts at T2/T3 with the failure not
understood · changes touching the shield, the deploy path, calendar integrity constraints,
`decide_tick`, the metronome, the purge, or anything in `docs/06_DOCTRINE.md` · Reviewer passes on
those same objects · drafting a `[DECISION NEEDED]`.

**T4 mechanics:** the lane opens `ESCALATION — <lane> — <task>` with both failed attempts and
evidence. The Conductor responds within its next cycle: (a) spawn a T4 sub-agent against the lane's
branch and post the result on the issue, (b) re-scope the task, or (c) park it. The lane polls the
issue.

### Budget — hard ceilings, not vibes

- **Per-task ceiling:** a task that has consumed **two full working sessions (or ~2M tokens)
  without passing SELF-TEST stops** and becomes an ESCALATION. No infinite retry loops — this
  supersedes "spend reasoning budget freely," which was written for a supervised single agent.
- **External-API budget (cuOpt + Nemotron):** metered per solve. **Day-zero `[DECISION NEEDED]`
  asks Chase for the weekly cap** (recommendation: $50/week until data says otherwise). Lanes log
  cumulative spend in their claims; projected overrun → Conductor before the next solve.
- **Fleet spend visibility:** the Conductor aggregates whatever spend Hermes exposes plus the
  claim-logged API spend into every `[DIGEST]`. If Hermes exposes no spend data, the digest says so
  rather than guessing.
- **Never economized, at any tier:** adversarial review, and any claim about the live database.

**Routing log:** every PR body records which tier/model built it and which reviewed it. A T2 model
producing a wrong load-bearing claim is a routing signal — only visible if recorded.

## 6. Liveness — who notices when something dies

- **Lane heartbeats:** the 2-hourly claim heartbeat (§3) plus a REPORT at every pipeline-stage
  transition. A lane silent past its claim TTL is presumed dead and swept.
- **Conductor heartbeat:** the Conductor updates a pinned **`CONDUCTOR HEARTBEAT`** issue **daily**
  and sends the weekly `[DIGEST]` on its fixed day. **Note for Chase: a missed digest day, or no
  Telegram traffic of any kind for 48 h, means the Conductor is down — restart it.** Silence is
  only golden while the heartbeat beats.
- **Lane fallback:** a lane whose REPORT has sat unacknowledged for **24 h** may send **one**
  `[BLOCKED]`-format Telegram message itself: "Conductor unresponsive since <time>." This is the
  single exception to Conductor-only comms.
- **Stuck detection:** the per-task budget ceiling (§5) doubles as loop-detection — a lane
  grinding a failing build for hours hits the ceiling and must escalate instead of retrying.

## 7. Review — two layers, defined

**Layer 1 — in-lane ADV-REVIEW (pre-PR):** a different model than the author attacks the diff
before the PR opens. Catches the cheap failures early.

**Layer 2 — the fleet Reviewer (post-PR):** the Conductor spawns Prompt-6 per PR, on a model
different from the author (read from the routing log). **Substantive** = touches logic, schema, or
user-visible behavior. **Trivial** = lockfiles, docs, formatting, pure renames — trivial PRs skip
Layer 1 and get a Layer-2 light pass only.

**The evidence ladder** (what "verify" means, in order of preference):
1. **Read-only SQL against the author's preserved post-STOP run data** — recompute the PR's key
   numbers from stored rows yourself.
2. **Fixture replay** (`twinRun.busyday.json`) for motion/geometry claims.
3. **A fresh live run only via a Conductor-approved SIM-RUN claim** — reserved for the
   highest-blast-radius PRs.

**Migration PRs:** the PR must name its preview branch; the Reviewer probes it read-only (did the
guard fire? md5 guard present? `pg_get_functiondef` captured before any DROP? new `sim_run_id`
tables in the purge exclusion? CASCADE blast radius counted?). Cannot reach the branch → UNVERIFIED.

**Verdicts (machine-parseable, four):** `APPROVE` · `APPROVE-WITH-NITS` · `REJECT` (each finding =
claim + failure scenario + location) · **`UNVERIFIED`** ("could not confirm or refute X; here is
the exact evidence that would settle it") — routes back to the author or to the Conductor for a
claimed verification run, never to Chase.

**Review budget:** one focused pass per PR. Exceeding it → UNVERIFIED with the list, not an
open-ended investigation. **Appeal:** a lane may appeal a REJECT once, to the Conductor, which
tie-breaks with a T4 pass.

## 8. Security

- **`ottoq-intelligence` and `ottoyard-OTTO-Q` are PUBLIC repos.** Their PR bodies carry sanitized
  evidence only — no hostnames, no spend figures, no query transcripts with headers. All claims
  and coordination for them live in the private context repo.
- **No secret ever enters an issue, PR body, log, or REPORT.** Redact tokens and Authorization
  headers from every transcript before posting. A leaked secret is a sev-1: report immediately per
  `docs/12_CREDENTIALS.md`, never quietly fix.
- **`docs/12_CREDENTIALS.md` is required reading for every lane**, not just the Conductor.
- Commit as a noreply identity (GitHub rejects `chase@ottoyard.com`).
- Lane D never deploys an edge function; nobody rotates a credential; the Conductor never writes
  to the database.

## 9. Templates

**Work-board item** (`LANE <X>: <outcome>`):
```
OUTCOME: <one sentence, testable>
EVIDENCE THAT PROVES IT: <the query/screenshot/number that will demonstrate it>
SIZE: <S/M/L>   DEPENDS ON: <issues/PRs or none>   PRIORITY: <P0-P3>
```

**REPORT** (comment on the lane issue):
```
REPORT <stage reached> — <task>
DONE: <what>   EVIDENCE: <one line + link>
NOT VERIFIED: <explicitly>
PR: <link or none yet>   ROUTING: built=<model> reviewed=<model>
NEXT: <what I do now>   FLAGS: <escalation / durable-fact / none>
```

**PR description:** what changed (plain language) · evidence (real numbers, links) · **NOT
verified** · blast radius · routing log · for DB PRs: preview branch name + "production change on
merge" + merge-order dependencies.

**CLAIM body:** holder · started-at · expected duration · evidence pending · (SIM-RUN: scenario,
seed, projected cost).

## 10. Standing collision rules

- **Lovable commits straight to `main`** on the three front-end repos and pulls merges back
  automatically. Fetch before every push. Mid-PR, if `main` moves: rebase, re-run SELF-TEST, note
  it in the PR. A `lovable-sync-<timestamp>` branch appearing = divergence → tell the Conductor,
  never delete it.
- **A merged PR breaking the deployed Lovable site:** the Conductor sends `[PR READY]` for the
  revert PR marked **URGENT**, and the offending lane authors the revert within one cycle.
  Detection: any lane noticing breakage reports it; Lane A additionally spot-checks the deployed
  site after its own merges land.
- **Claude Code sessions** may work in parallel as `claude/<slug>` branches — same claim board,
  same rules. Claude is the escalation reviewer of last resort. **`salvage/*` branches** are
  rescued work awaiting founder review: read-only.
- **DB-DEGRADED protocol:** on 3 consecutive query timeouts, the observing agent opens a pinned
  `DB-DEGRADED` issue → the whole fleet ceases non-essential queries and sim runs → the Conductor
  owns closing it (or `[BLOCKED]` if it does not self-recover). This instance has frozen before
  while its platform status read ACTIVE_HEALTHY.
- **Clone fresh, always.** No persistent local working copies. Push your branch before your session
  ends, every session.

## 11. Scaling discipline

**Start: Conductor + Lane A + Lane B.** Add C, then D, after the first week produces clean,
mergeable PRs. Known cost of this order: Lane C's P0 items (bay binding) wait a week — accepted
deliberately; the pipeline must prove itself on lower-blast-radius work first. Every additional
concurrent agent multiplies claim traffic, database contention, and spend. The goal is a steady
stream of small, proven PRs on the founder's phone — not maximum parallelism.
