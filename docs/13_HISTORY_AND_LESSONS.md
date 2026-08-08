# 13 — History and hard-won lessons

**Read this once, properly.** Every entry cost real time. Most cost a day or more. Several are cases
where a confident, plausible, *wrong* belief propagated into another session and wasted its opening
hypothesis.

---

## Part 1 — The arc of the build

**April 2026** — OTTO-Q Core scaffolded: schema, a TypeScript engine, adapters, an API layer. Lovable
prompts drive the first cockpit builds.

**May 2026** — Seven real-world corpora ingested and fitted into the twin's variability generators.
The twin becomes calibrated rather than arbitrary.

**2026-06-01 — the audit that reset everything.** A read-only audit proved OTTO-Q was **theatre**:
OTTO-TWIN had been orchestrating the depot itself with greedy heuristics, and **OTTO-Q was never
called in the tick loop.** 0 of 31 brain functions were being invoked. Every "watch OTTO-Q handle
it" demo up to that point was fiction.

Feature work stopped. Frontier research produced a named, citable architecture — **Simplex / Runtime
Assurance** — and Chase's directive: *architect everything thoroughly before building.* Phase 1 was
split into **1a (architect)** and **1b (build)**.

**2026-06-03 — Phase 1a, and a finding bigger than the first.** A 31-agent architecture workflow
produced a 599-line master spec — and a red-team pass proved the 28-rule shield **mostly did not
enforce even when called**: fail-open guards, a dead state-machine vocabulary (3 of 17 states
overlapped), authority-free overrides, dead grid/DR wiring, a clock-frame bug, a missing
`requested_kw`. Honest count of currently-decidable block predicates: **≈14, not 23.**

> **The lesson Chase drew, and it is the reason architecture-first is now the default here:** had we
> built the naive Phase-1 design blind, we would have shipped a shield that **looks real but silently
> passes half its safety rules** — exactly what an OEM auditor destroys you over.

**2026-06-03/05 — Tier 0 and Tier 1.** The enforcement contract goes live. **Three silently-dead
safety rules found and fixed** — EN.005 had dead wiring; EN.001 and EN.004 **crashed their BLOCK
paths on a `printf` format specifier** (Postgres `format()` has no `%.1f`). All three now enforce.
Fail-closed on block-tier evaluator error. The frozen-frame store with sha256 integrity.

**2026-06-04/05 — the cutover.** `ottoq_sim_advance_tick` is repointed at `ottoq_decide_tick`.
**OTTO-Q makes all five decision types through the shield every tick.** Greedy is demoted to a
shadow baseline. The original seam audit now passes: brain in loop ✓, greedy out ✓, 693 rule-evals
per 10 minutes versus 2 ever, 414 decisions recorded.

**2026-06-05 — the A/B under CRN.** Five seeds. OTTO-Q is the **only** policy with 0 unsafe deploys
and 0 breaches, calm or stressed (paired-t, t = 13–19). Under demand surge, greedy collapses (31
unsafe deploys, 9.6 breaches, 0% ready) while OTTO-Q is byte-identical to its calm-day self.

**June 2026** — Frontier core: the policy substrate, twin-in-the-loop MPC, the self-improving CIL,
uncertainty calibration, agentic NL command. The energy control loop is rebuilt after discovering
**OTTO-Q was disconnected from its own battery.**

**Late June 2026** — The visual layer. Tier A motion in Three.js. Tier B: RunPod fails (container
cannot reach the GPU render node), AWS succeeds, a hand-built Kit app segfaults on version skew, and
the prebuilt Isaac Sim container wins. **The depot renders photoreal in the cockpit.**

**July 2026** — The appointment-depot handshake. The comms layer to OEM spec. The forward
availability doctrine. The schema separation (twin / ottoq / public). The intelligence architecture
verdict. And **the certification crisis** (below).

**August 2026** — The orchestration build: needs card → atomic enactment → space-agnostic assignment
→ armed overlap guard. cuOpt enacted for the first time. The depot-layout discovery. The database
outage and recovery.

---

## Part 2 — The certification crisis, and what it taught

This is the most important section in this file.

**2026-07-26 — the baseline itself was invalid, and every comparative number ever produced was
void.**

`ottoq_sim_auto_charge_assign_tick` — reached **only** via `ottoq_greedy_tick`, i.e. the **baseline
arm** — tested stall occupancy with `cs.status::text = 'in_progress'`. The enum
`ocpp_session_status` has **no such label** (active / completed / faulted / cancelled). The `::text`
cast turned what would have been a hard enum error into a predicate matching **zero rows forever**,
so the LEFT JOIN never found an occupant and **every stall always looked free.**

**Measured: 54,274 overlapping session pairs across all 21 greedy runs / 33 stalls. Maximum 100
vehicles on ONE charger simultaneously.** otto_q and fifo: zero.

**The baseline was not naive. It had unlimited chargers.**

Fixed and **structurally prevented** by a partial unique index
(`ocpp_sessions(stall_id) WHERE status='active'`) that makes double-booking impossible from any code
path. The naive *character* was deliberately preserved — only the physics was corrected.

**Then the re-certification was itself wrong.** A same-day adversarial audit refuted 3 of 4 claims,
each of which was then re-verified against the live database:

| First reported | Truth after verification |
|---|---|
| Grid peak **−41.1%**, 3/3 | **~−10.5%, 2/3.** 81.7% of the gap was a **BESS artifact** |
| Productive deploys **+9.7%**, 3/3 | **−6.2%, OTTO-Q loses 3/3** once orphan deploys are removed |
| Throughput −11.3% = "the cost of the doctrine" | −11.3% is real but caused by **a bug**, not doctrine |
| "otto_q had ZERO overlapping sessions" | **False** — 2 otto_q runs overlapped via a *second* path |

**The BESS artifact:** the baseline issues no energy commands, so its battery ran an uncontrolled
default whose "off-peak top-up" branch charged at **750 kW right on top of the depot's own overnight
EV wave** — and that *set* the greedy peak in all three seeds. **A second strawman of exactly the
same kind as the phantom chargers.** A real BESS controller would never charge into its own peak.

**The orphan deploys:** OTTO-Q routed 148 of 263 deploys through `en_route_to_deployment` and **43
never completed.** Root cause: an architecture violation — the brain was writing `vehicles.state`
directly while the twin's rate-limited dispatcher actually performs departures.

### The rules this produced

1. **Interrogate the baseline before you believe a win.** Twice, a "win" was the baseline being
   secretly broken in our favour.
2. **Always state your denominator.** 98.8% and 68.1% were both true of the same run: one is
   conditional on a booking existing, the other is end-to-end.
3. **Never quote final-frame instantaneous state counts** — `vehicles_turned_around`,
   `fleet_ready_pct`, `gate_backlog`. They structurally penalise OTTO-Q, because the full-service
   doctrine keeps more cars mid-service at any instant.
4. **A vacuous guard is worse than no guard.** The overlap constraint was scoped to states no row
   ever had (`held`/`active`) while every booking was born `done` ⇒ **every prior "zero
   double-bookings" pass was vacuous**, and 22 vehicles had been booked into one service bay.
5. **A metric can improve by LYING or by FORGETTING.** Eight of 24 bay "dones" lasted under 5
   minutes — two lasted **43 seconds**. Mid-service ejections were scored as completed work. And
   `ottoq_reopen_visit_atoms` was **dead code**, so interrupted work silently vanished. **A metric
   that improves by forgetting outstanding work is worse than an inflated one.** Report done /
   interrupted / legacy side by side.
6. **Measure occupancy as a union of intervals clipped to the run window.** A naive sum of booking
   minutes reported service-bay utilisation at **242%**.
7. **Capture evidence only after the run is stopped.** A mid-run capture understated assignments
   6→10 and overstated cuOpt share 20%→33%.
8. **Run ≥139 sim-minutes or certify nothing.** Two phases were wasted on n=3 and n=6.
9. **Finishing the run IS the deliverable.** One phase shipped code and never finished its run —
   every number read NOT MEASURED and nothing was preserved.
10. **State your own denominator, and never silently reuse a number you cannot rebuild.**

---

## Part 3 — The seam failures

**The recurring shape of failure in this system is not a logic error. It is two correct components
with an interface between them that silently loses information.**

### The `leg_type` abort — one word killed every decision

**Symptom:** `ottoq_decide_tick` produced **zero** stall_assignment decisions, tick after tick,
despite 24 vehicles waiting at the gate and 24 cuOpt proposals pending. Meanwhile
`cron.job_run_details` reported the metronome as **`succeeded`** every single time.

**Found via** `get_logs(postgres)` — on Chase's instruction to check the Postgres logs.

**Root cause:** `ottoq_plan_visit_itinerary` has three leg-writing loops. One used a total
`CASE … ELSE 'service' END`. **Two used `v_a->>'svc'` raw.** They worked only by coincidence — most
twin service codes happen to spell valid `leg_type` values. `interior_inspection` (100 live atoms)
does not. One rejected row → `decide_tick` runs in ONE transaction → **every charging decision in
that tick rolled back.**

⚠️ **`perimeter_walkaround` was a second, still-unexploded instance** — found only because the full
`svc` vocabulary was probed instead of stopping at the first offender.

> ### The doctrine this established
> **The twin's service vocabulary is OPEN.** `twin.ottoq_sim_generate_service_manifest` can mint any
> code. **OTTO-Q's itinerary vocabulary is CLOSED** — 22 values in a CHECK constraint.
> **Therefore every seam that carries a twin code into an OTTO-Q column MUST be a TOTAL function.**
> A partial mapping is not a style issue — it is a whole-decision-loop outage waiting for the twin
> to add one service.
>
> **And the same rule applies when real telemetry replaces the twin**, because real fleets will emit
> codes we never enumerated.

### The other seam failures, for pattern recognition

| Failure | The seam |
|---|---|
| **`Math.round` inverted Nemotron** | 106 of 109 dial writes landed on the **extreme opposite** of what the model asked. The model worked; the plumbing inverted it. **Verify a model's effect at the dial, not at its output.** |
| **Sim clock vs real clock** | `last_heartbeat_at` written in the **sim** domain; the edge function filtered with `Date.now() - 90_000` in the **real** domain. A run whose sim clock sat hours from now saw **every charger stale**. |
| **pg_net after COMMIT** | `ottoq_cuopt_refresh` was called in the same transaction as the decider. **The HTTP request had not been sent when the decider read the seam.** That call site could never work, by construction. |
| **The candidate set discarded at the wire** | cuOpt's refresh computed a precise candidate list, then posted **only** `{sim_run_id}`. The edge function re-derived the cohort ~4 s later — **one full tick late by construction.** |
| **Two uncoordinated stall searches** | `decide_tick` picked a stall and reserved it; `ottoq_book_workflow` ran its **own** search and booked a different one. Agreement was **3 of 81 (3.7%)**. Not a race — **guaranteed wrong by construction.** And the availability oracle excluded only maintenance/closed stalls, **never occupied ones**, so the ledger was wrong in both directions at once. |
| **`enum::text` defeating a partial index** | 2,449 ms → 0.83 ms once fixed (~2,900×). A shield rule was scanning 28,963 rows on every call, ~33× per tick. |
| **`REVOKE … FROM anon` while PUBLIC held the grant** | A no-op that *succeeded*. All 48 SECURITY DEFINER world-state writers stayed callable with the publishable anon key — including `ottoq_sim_seed_fleet` (blanket-updates every vehicle) and `ottoq_purge_prior_runs` (deletes the black box). |
| **A policy named "Service role full access" that was `TO PUBLIC`** | 31 tables world-writable while every name-based audit said they were fine. **Read `pg_policies.roles`, never `policyname`.** |
| **The retention purge's swallowed error** | `ottoq_events_block_mutation` allows DELETE only when a GUC is set. The purge never set it, every delete was refused, **the error was swallowed, and the job reported success** — for months. |
| **A uniqueness index enforcing the opposite of its name** | `idx_stalls_one_vehicle_per_stall` is `UNIQUE ON stalls(current_vehicle_id)` — one **stall per vehicle**. It structurally cannot detect two vehicles claiming one stall. |
| **A wash gate whose escape hatches were unreachable** | 34 vehicles due, 0 atoms emitted. Max `soil_index` 0.4417 against a 0.75 override; `cycles_since_wash` drawn 2–5 against a backstop of 9. **Idle by construction.** |
| **A decorative column** | `service_cadence_policy.seed_phase_max` holds the exact constants hardcoded inside the seeder — **and the seeder never reads it.** Tuning it changes nothing while appearing to. |

---

## Part 4 — The database incidents

### The capacity ceiling and the 14 GB outage

`ottoq_events` reached **11 GB of a 14 GB database** — 4.05–4.33M rows, of which **7 GB was live
TOAST**, not bloat (dead tuples were 1.6%). **~1.7 KB of JSON per event.** The disease was that
**events were far too fat**, not that cleanup was skipped.

Supabase reported `ACTIVE_HEALTHY` while the database **could not reliably accept a connection with
zero simulation runs active.** Trivial indexed reads timed out. The metronome went from 0.1–1.2 s to
**10–84 s**; `advance_tick_world` measured **48–65 s** against a normal ~1 s.

**Resolved: 14 GB → 652 MB, ticks back to ~1 s.**

**Two hard truths from it:**
1. **Deleting rows does not shrink the file.** VACUUM frees space *inside* the file and only
   truncates trailing pages; the deletable rows are the oldest = the head of the heap. Real reclaim
   needs `pg_repack` or time partitioning + `DROP PARTITION`. **Never `VACUUM FULL`.**
2. **Probe a deletion boundary directly; never trust a computed gap.** A proposed cutoff based on a
   "clean write gap" would have deleted ~7,126 rows only **6.6 days old**. The agent that probed the
   real boundary found the true edge and added a redundant guard.

**And an honest self-assessment from that session, worth internalising:**
> *"This session added ~3 GB to that table via repeated full-length certification runs. The work was
> real, but I kept filling a container I already knew was nearly full. Future certification runs must
> be budgeted against remaining headroom, not treated as free."*

### The tick-cost investigation

**The answer: event-write amplification.** `ottoq_record_event` cost ~28 ms/call × 60–120 calls/tick
= the entire tick. It is called from *inside* every step, so each step absorbs only a slice and
**nothing looks slow individually** — which is exactly why per-function profiling found nothing.

Root cause: index maintenance on a table that no longer fit in cache. Controlled test in an aborted
transaction: insert into `ottoq_events` (**18 indexes**) = **4.03 ms**; identical insert into an
unindexed clone = **0.065 ms**. **62×.**

**Three hypotheses refuted — retained so nobody re-tests them:**
1. **The safety shield.** All rule evaluations summed to **2.7 s across a 12-minute window**; one
   17.7 s tick had **zero** evaluations.
2. **The pg_net HTTP post on the hot path.** pg_net is async — measured 119 ms.
3. **Metronome lock contention.** Fencing changed nothing.

⚠️ **And a retraction:** "the metronome is 35.7% of DB time" was a **measurement artifact**.
`pg_stat_statements.track` is `top`, which rolls all nested work into the caller. That 35.7% **is**
the tick cost seen from outside, not a second 35% to reclaim.

### The quadratic safety check that flattened the box

A schema-move migration ran one `SELECT count(*) FROM pg_proc WHERE prosrc LIKE '%public.<name>%'`
**per candidate** — ~79 of them, each a ~21 s sequential scan over every function body. **~25 minutes
of saturated compute**, 137-second checkpoints, pg_cron "job startup timeout" on every job, statement
timeouts everywhere, the MCP connector dead for minutes.

**The migration itself completed correctly. The damage was the safety check, not the change.**

⇒ **One pass, not N.** Build all reference checks into a single scan. And **treat any migration
expected to exceed ~15 s as an outage risk.**

### The near-miss that nearly deleted the ledger

Migration 0010 (depot layout) was briefed as "5 stalls removed, zero references." The file actually
retired **19 per depot**, and **13 carried real booking history**.
`ottoq_stall_bookings.stall_id` is **ON DELETE CASCADE** ⇒ applying it would have **silently
vaporised ledger rows with no error.**

**The apply halted at preflight. Correctly. That preflight is why the ledger still exists.**

---

## Part 5 — Lessons about how to *work* here

### Re-verify a stored "established fact" before passing it on

The "off-map render bug" was recorded as **"DEFINITIVE: the motion complaints are a RENDERER bug"**
and handed to a fresh session as a premise. It **does not reproduce** — 0 sightings across a captured
116-vehicle run. **It cost that session a wasted starting hypothesis.**

The same file's real finding — `LaneGraph.route()` displacing its own endpoints by 3.20 units against
a 5.7-unit stall pitch — was the actual bug, and it explained **both** symptoms at once.

### Prefer a runtime marker over source inspection

An earlier phase "fixed" an edge function **while it stayed byte-identical at v22.** The later fix
was only provable because invocations began reporting `cohort_mode="pinned"` at runtime.

### Guess less; pick the right instrument

The clock-ratio investigation raised **three hypotheses and refuted all three** before finding the
answer. The answer did not come from a better guess — it came from realising that
`sim_clock_current` and `last_tick_at` are written by the **same statement** and are therefore
phase-aligned by construction, while a continuously-running `now()` is not.

> **Never sample a step-advancing clock against a continuous one.**

### Do not trust a sub-agent's "I couldn't find it"

An investigation reported that a rule evaluator "lives in a TypeScript edge function outside scope,
never found." It is an ordinary SQL function, **one `pg_proc` query away.**

### Adversarial review beats independent planning

Two independent plans for the rule-1 inversion both self-rated ATTEMPT_WITH_CONDITIONS. **Three of
four adversarial reviewers returned REJECT — and they were right.** The one fact that decided it
(the actuation window is 30 sim-minutes, not 1–3) invalidated every risk rating in both plans.

Separately: **only 1 of 7 generated split plans was safe as written** — two were truncated
mid-statement, one had a NULL-hour window inversion, one made an unsanctioned predicate change.

### Two chats in one repo folder share one working tree

Use `git worktree` per session, or clone into your own directory. This has actually caused collisions.

### Verify a claim about your own tools

`ALTER ROUTINE … SET search_path` applied blanket to all 396 routines **broke the metronome** — a
routine with a `SET` clause cannot `COMMIT`. Cron job 12 failed two consecutive beats.

### Patch technique that worked reliably

For editing proven functions safely: server-side `replace()` / `regexp_replace()` on
`pg_get_functiondef(oid)` plus a guarded `EXECUTE` with an occurrence-count guard that aborts if the
count is ≠ 1. **Zero transcription risk** on a 234-line function.

Gotchas hit and fixed along the way: event `actor_type` must be in the catalog set (use
`command_center_operator`, not `depot_technician`) · `data_source` ∈ {production, twin, replay,
shadow} · new `event_type` values must be INSERTed into `ottoq_event_types_catalog` (FK) · Supabase
`execute_sql` runs multi-statement input as **ONE transaction** (a trailing error rolls back the
whole batch including ticks) and returns only the **last** statement's rows.
