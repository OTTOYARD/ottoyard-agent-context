# 15 — What I do not know

Written by Claude Code, 2026-08-08. **Chase's instruction for this package was explicit: identify
anything you are uncertain about rather than inventing information.** This is that list.

**Treat nothing here as settled.** Where a gap matters to your work, verify it yourself or ask.

---

## 1. Things I could not verify at all

| # | Unknown | Why it matters | How to resolve |
|---|---|---|---|
| U1 | **Where seed `424242` is pinned.** | Until found, every run is identical and no variability work can be judged. | Grep every caller of `ottoq_start_demo_run` / `ottoq_start_busy_run` across the DB, the edge functions, and all three front ends. Check the operator console defaults and any cron/harness. |
| U2 | **Whether the branches listed in `11_BACKLOG.md` H7 are merged.** | Several backlog items may already be done. | `git branch -a` per repo and compare against `main`. |
| U3 | **Whether `docs/OTTOQ-TWIN-BOUNDARY.md` is on `main`** in `ottoyarddepot-sim`. | It was previously one branch deletion from gone. It exists on the local clone; I did not confirm the branch. | `git log main -- docs/OTTOQ-TWIN-BOUNDARY.md` |
| U4 | **What migration `20260808182226` did.** | It was applied to production **today**, after my last full inventory. Someone (a parallel session, or Chase) changed the brain. | `select * from supabase_migrations.schema_migrations where version = '20260808182226';` |
| U5 | **The current live state of the AI enactment path.** | The cuOpt share figures span 2.8% → 42.4% → 18%, and P0-3 (dispatch discarding the AI's choice) may or may not still hold. | Run a `busy_day` run of ≥139 sim-min, stop it, then group `ottoq_decisions` by `proposed_action->>'source'`. |
| U6 | **Whether the refusal path now fires** (P1-11). | The pre-flight work should have changed this; I did not re-measure. | `select status, confirmed_by, count(*) from ottoq_vehicle_commands group by 1,2;` |
| U7 | **Whether the phantom-booking count is still 0** (P2-18). | It was 5 of 108, then 0 of 211. Phases moved. | `ottoq_booking_provenance_audit(run)` on a fresh run. |
| U8 | **Business numbers.** Depot CapEx, revenue model, pilot raise size, unit economics. | I deliberately did not extract these from the PDFs on the Desktop, and they are not in any repo. | Ask Chase, or read `~/Desktop/OTTOYARD/OTTOYARD Fleet Depot Economics 12.7.25.pdf` and the company brief if you have filesystem access. |
| U9 | **Whether the `ottoq-intelligence` EC2 service is currently up.** | The frontier layer is dead without it. Its IP has changed at least once. | `curl <url>/health` — and the URL itself is in the edge function `ottoq-energy-mpc`. |
| U10 | **The current state of the Isaac Sim box.** | It may be stopped (correctly, to save cost) and its public IP will have changed. | Ask Chase; there is no AWS console access. |

---

## 2. Things where the record contradicts itself

| # | Contradiction | My read |
|---|---|---|
| C1 | **Is the wear model supposed to be stateful?** D5 says needs are a probability draw and **wear modelling is an explicit non-requirement**. But P1-14 flags that tire tread, brake wear and SoH are frozen — and D4 requires completion to write back. | These are reconcilable: the *draw* sets the starting condition (that stays), and *completion* must advance last-done (D4 requires it). What is genuinely unresolved is whether elapsed time / mileage should degrade a condition between runs. **Ask Chase before building.** |
| C2 | **The safety claim.** Memory variously says "deterministic shield ⇒ 0 unsafe" and "the shield is not on the deploy path." | The 2026-07-23 adversarial audit is the later and better-evidenced account: it is an **inline deterministic precondition**, not a rule-engine shield, on the deploy transition specifically. `02_OTTOQ_ARCHITECTURE.md` §4 states it this way. |
| C3 | **Charger counts.** The flagship is variously described as 45 charge stalls, 10 DCFC + 35 L2, and 10 DCFC + 20–30 L2. | The **live DB** is the answer, and it changes with the layout work. Query `stalls` grouped by `stall_type` for the depot you care about. Do not quote a number from any document. |
| C4 | **Fleet size.** 116, 163, 220, and "~100+" all appear. | Scenario- and depot-dependent. Always scope your query to a depot and a run. |
| C5 | **Function counts.** "118 sim/twin functions" (the world contract doc) vs 387 non-extension routines vs my measurement of twin=71 / ottoq=55 / public=1088. | Mine are live and current, but `public`'s 1088 includes ~744 PostGIS functions. The "118" figure is a scoped number reused as an unscoped total. **Always state the scope.** |
| C6 | **The DB outage status.** The memory file body says UNRESOLVED and paused by Chase; the index line says RESOLVED. | **Resolved.** I verified: the database is 652 MB and `ottoq_events` is 147k rows / 195 MB. The file body is simply older than its own index line. |

---

## 3. Open questions for Chase that were never answered

These were put to Chase and, as far as the record shows, not resolved.

| # | Question | Context |
|---|---|---|
| Q1 | **Is NVIDIA-cuOpt-in-the-loop a positioning requirement**, or may we ship OR-Tools CP-SAT if it beats cuOpt on the bench? | Materially changes the Lane-D architecture. |
| Q2 | **What is the real per-vehicle morning-deploy due-time distribution**, and the true wave arrival/SoC shape, to certify the overnight-wave scheduler against? | Without it the wave cert is calibrated on assumptions. |
| Q3 | **What charger-sweep endpoints should the CapEx proof target** — ~20 chargers per 100 vehicles? Score against the two real depots' inventory? | This is the CapEx headline. |
| Q4 | **Approve retiring the continuous dial-railing Nemotron conductor** in favour of discrete regime selection? | It is a diligence liability as-is. |
| Q5 | **OD-31: the real Nashville grid interconnection ceiling in kW.** | A real number is needed; everything energy-related is currently modelled against an assumed cap. |
| Q6 | **OD-13: the OEM-timeout policy** — deploy on partner silence, or hold? auto_accept vs hold? | An OEM-facing contract decision. |
| Q7 | **Should the `ottoyard-OTTO-Q` repo be renamed?** It is OrchestrAV, not the brain, and every reviewer trips on it. | Disruptive but the confusion is real and recurring. |
| Q8 | **Gate B override doctrine sign-off.** Overriding a reservation a vehicle already holds is exactly the in-depot-reassignment case requiring tech approval. | Blocks part of the cuOpt work. |

---

## 4. Things I am confident about but you should still spot-check

Because they are load-bearing, and because this project's history is full of confident-and-wrong.

- **cron job 12 is the run engine.** Verified in `cron.job`. Spot-check before relying on it.
- **`anon` has no USAGE on `ottoq` or `twin`.** Verified by function-count and schema inspection;
  re-verify with `has_schema_privilege('anon','ottoq','USAGE')` if it matters.
- **The 3D car is authored in metres in unit-space.** From the memory record, not from my own reading
  of `Vehicle3D.tsx`. **Read the file before changing it.**
- **The staging sort is backwards in four call sites.** From the memory record. Grep
  `staging_role = 'temp'` to confirm the current count and locations.
- **`ottoq_book_appointment` cannot reserve bays.** Migration 0011 may have changed this. Check
  whether `fwd-bay-reservation-0011` merged.
- **Every number in `10_KNOWN_ISSUES.md`** was measured at some point in the past. Dates matter.

---

## 5. What is deliberately not in this package

- **No secret values.** By design. See `12_CREDENTIALS.md`.
- **No business financials.** See U8.
- **No exhaustive per-function API reference.** ~290 RPCs; the live database is the reference, and
  `otto-q-core/db/baseline/` is the file mirror. I documented the ones you will actually use.
- **No line-by-line code walkthrough.** It would be stale within a week. I documented *shape*,
  *invariants*, and *traps* instead — the things that stay true.
- **No UI screenshots.** They date immediately.

---

## 6. If you find something in this package that is wrong

**Say so, with evidence, and fix the file.** That is not a criticism of the package — it is exactly
how it is supposed to work. Half the value of `memory/` is that it records Claude discovering its own
earlier beliefs were false.

Open a PR against this repository. Put the correction in the doc **and** note the date, so the next
agent can see which claim superseded which.
