# 04 — OrchestrAV, OTTO-PULSE, and the ecosystem tie-in mandate

## 1. The two cockpits

| | **OrchestrAV** | **OTTO-PULSE** |
|---|---|---|
| Repo | `ottoyard-OTTO-Q` (public) ⚠️ misleading name | `ottoyard-field-ops` (private) |
| Who uses it | **Fleet owners** — Waymo Nashville, Tesla Robotaxi TN, Zoox Southeast | **Depot operations staff** |
| Scope | **Only the logged-in operator's vehicles** | **Everything, all owners** |
| Mental model | "Where are *my* cars and when do I get them back?" | "What is happening in *my depot* right now?" |
| Stack | Vite + React + shadcn + Tailwind | Vite + React + shadcn + Tailwind + TanStack Query |
| Local clone | `~/Desktop/OTTOYARD/ottoyard-OTTO-Q` | `~/Desktop/OTTOYARD/ottoyard-field-ops` |

There is also **OrchestraEV** (`src/pages/OrchestraEV.tsx` in the OrchestrAV repo) — a retail EV
membership surface. Lower priority; it exists.

## 2. Where each reads its data (get this wrong and nothing works)

### OTTO-PULSE
- **Use `src/lib/supabase.ts`** — hardcoded to `gxdrcyphqjzjsuhxuqtg`.
- **NEVER use `src/integrations/supabase/client.ts`** — it points at a dead project.
- `DEPOT_ID` hardcoded `11111111-1111-1111-1111-111111111111` in ~6 files. Accepted convention.
- Access pattern: **RPC + `refetchInterval ~10s`. Polling only, no realtime.**
- Hook pattern to copy: `src/hooks/use-ottoq-frontier.ts`.
- Tabs: `src/components/tabs/{Overview,Depot,Vehicles,Energy,Waves,Incidents,Comms,OttoQ}Tab.tsx`.
- Card style: hand-rolled `div.rounded-lg.border.bg-card`; status-chip and timeline-dot patterns
  live in `VehiclesTab.tsx`.
- **Honest empty states. No mock fallback.** Follow the PrimeOps pattern. Several older hooks
  (vehicles/stalls/incidents) still fall back to mock arrays on API error — **that is a bug to fix,
  not a pattern to copy.**

### OrchestrAV
- `src/lib/otto-q-api.ts` → `ottoqRpc` (PostgREST RPC against `gxdrc`) and `ottoqInvoke` (edge
  functions). Use these; do not add a new client.
- The **Fleet tab already reads real `gxdrc` vehicle UUIDs** via `ottoqInvoke("ottoq-fleet-vehicles")`,
  and that edge function **already accepts a `fleet_operator_id` filter** — no ID mapping needed.
- Vehicle card: inline JSX in `src/components/OTTOQFleetView.tsx` (~lines 627–774). Sequence-render
  precedent: `VisitReportCard.tsx`.
- ⚠️ **Auth lives on the `ycsis` project (legacy).** `useOTTOQRealtime` subscribes to `ycsis` tables,
  which are **dead for `gxdrc` views — poll instead.**
- ⚠️ **The user → fleet-operator binding is client-side only** (anon key, no JWT). It is spoofable.
  Fine for a demo; real tenancy is a known security gate.

### The shared read surface
- **Gateway:** `/functions/v1/otto-q-api/api/v1/...` — `/fleet/summary`, `/energy/history?range=24h&depot_id=`,
  `/ai/depot-projection?depot_id=`, `/fleet/schedule-intelligence?horizon=`. `ottoQFetch` unwraps
  `{data, meta}`.
- **Direct PostgREST tables:** `site_energy_snapshots`, `ottoq_energy_commands`, `waves`.
  ⚠️ `ottoq_bess_units` is RLS-blocked to anon and returns `[]` — get battery SoC from the
  `ottoq_nl_status_brief` RPC instead.
- **Card RPCs (already built and live):**
  - `ottoq_depot_cards(p_depot_id, p_fleet_operator_id DEFAULT NULL)` — one call feeds a whole
    cockpit; the operator filter is OrchestrAV's scoping.
  - `ottoq_vehicle_card(p_vehicle_id)` — a single vehicle's work order.
  - Verified over the cockpit REST path: ~290 KB for 116 vehicles; an operator-scoped call returned
    46/46 Waymo-only.

**Honesty rules baked into the card RPCs — preserve them:**
- **Prune `skipped` legs** (55% are speculative blocks that never ran).
- **Scope to the current visit** — newest itinerary this run; last 3 done + active + next 5 planned.
  Do not let a card balloon to 235 legs.
- **Omit `deviation_s`** — the baseline is corrupt. Show planned and actual times instead.
- **Atoms use the key `'svc'`.** The contract doc says `'atom'`; the doc is stale.

**Live fleet operators on `gxdrc`:** Waymo Nashville (88 vehicles), Tesla Robotaxi TN (65), Zoox
Southeast (63), Local EV Fleet Co (0).

---

## 3. ⭐ THE ECOSYSTEM TIE-IN MANDATE

**This is a direct founder instruction and it is one of your two headline lanes.**

> *"The complete ecosystem tie-in between the Twin, OrchestrAV and OTTO-PULSE so that they all
> depict the same information — and OTTO-Q being the orchestrator and sorter of all of them."*

### The problem today

The three surfaces have grown independently. The twin cockpit shows the depot moving. Pulse shows
depot operations. OrchestrAV shows a fleet owner's vehicles. **They do not currently present one
coherent, consistent picture of the same live run**, and in some places they read different
projects, different tables, or fall back to mock data.

That is the single biggest thing standing between "three demos" and "one product."

### What "the same information" has to mean

Not "similar dashboards." Specifically:

1. **One run, one clock.** When a run is live, all three surfaces show *that* run, at *that* sim
   clock, with the same tick. When no run is live, all three say so honestly and identically — not
   one showing an empty lot, one showing stale numbers, and one showing mock data.
2. **One vehicle, one truth, three lenses.** Vehicle `X` at 09:14 sim-time is in DCFC stall 12,
   87% SoC, 14 minutes from a wash bay booking, owned by Waymo Nashville, with a reason for every
   one of those facts. The twin **draws** it. Pulse shows it in the depot's work queue. OrchestrAV
   shows it in Waymo's fleet list. **The numbers must match to the digit**, because they come from
   the same RPC.
3. **Every surface can answer "why."** The differentiator is not that the car is in stall 12; it is
   that OTTO-Q can say *why stall 12, why now, what it displaced, and what happens next*. The
   `ottoq_decisions` ledger, the `ottoq_rule_evaluations` trail, and the itinerary legs already hold
   this. It is largely not surfaced.
4. **OTTO-Q is visibly the orchestrator.** All three surfaces should make it obvious that a single
   brain is sorting everything — a shared "OTTO-Q is thinking / decided / enacted" signal, the same
   decision feed, the same approval queue.

### The architectural shape to build toward

```
                         ┌──────────────────────────┐
                         │        OTTO-TWIN         │   world state, physics, motion
                         │  (twin schema, gxdrc)    │
                         └────────────┬─────────────┘
                                      │ world state
                                      ▼
                         ┌──────────────────────────┐
                         │         OTTO-Q           │   decides · sorts · reserves · explains
                         │  (ottoq schema, gxdrc)   │
                         └────────────┬─────────────┘
                                      │ ONE read contract
             ┌────────────────────────┼────────────────────────┐
             ▼                        ▼                        ▼
     ┌───────────────┐      ┌──────────────────┐      ┌────────────────┐
     │  Twin cockpit │      │   OTTO-PULSE     │      │   OrchestrAV   │
     │ depot-sim     │      │  depot ops, all  │      │ one operator's │
     │ 2D / 3D / RTX │      │  owners          │      │ fleet          │
     └───────────────┘      └──────────────────┘      └────────────────┘
```

**The design principle: one read contract, three projections.** Each cockpit is a *filter and a
presentation* over the same OTTO-Q read surface — never its own reimplementation of depot state.

`ottoq_depot_cards(depot_id, fleet_operator_id)` is already exactly this shape: **one call feeds a
whole cockpit, and the operator filter is the only difference between Pulse and OrchestrAV.** Extend
that pattern rather than inventing a parallel one.

### Concrete work this implies

- Build a shared **run-context** read (`ottoq_twin_run_context` already exists) that every surface
  consumes for run id, status, sim clock, tick, scenario, seed, speed — so the three cockpits can
  never disagree about *which* run they are showing.
- Kill every mock fallback in Pulse and OrchestrAV. An honest "no live run" beats a plausible lie.
- Reconcile OrchestrAV's `ycsis` dependency: auth is there, data is on `gxdrc`. The realtime
  subscription against `ycsis` is dead code for `gxdrc` views and should be removed or repointed.
- Surface the **decision feed** (`ottoq_decisions`, with `resolved_action_context`, source, and
  rationale) in all three, scoped appropriately.
- Surface the **forward calendar** — the thing that makes OTTO-Q an orchestrator — as a real
  timeline in Pulse and as "your car's plan" in OrchestrAV.
- Make the twin cockpit's operator console the single place a run is started/paused/stopped, and
  have the other two reflect that state rather than each having their own controls.

### Vehicle cards — the pattern already agreed

Chase approved this shape (2026-07-16) and it should be the template for the tie-in:

> Card at rest = owner badge (Pulse only) + current step + progress bar + next step.
> Click to expand = the full sequence: done ✓ / current ▶ / upcoming, with times.
> **Streamlined, not bulky.**

Pulse shows **all owners**. OrchestrAV shows **only the logged-in operator's**, inside each
vehicle's Fleet-tab card. Both read `ottoq_depot_cards` / `ottoq_vehicle_card`.

Branches already carrying this work (unmerged as of 2026-08-08):
- `ottoyard-field-ops`: `claude/ottoq-vehicle-cards` (base `claude/prime-ops-panel` — merge that
  first or together), `claude/prime-ops-panel`.
- `ottoyard-OTTO-Q`: `claude/ottoq-sequence-cards`.

**Check whether these merged before rebuilding any of it.**

### The depot's robots in the three apps (added 2026-11-09)

Chase, 2026-11-09: *"the robotics aspects of the depot need to be reflected in the UI's respectively. Pulse should
have depot/operator level info on the functionality of the machines (not necessarily CAD-level, but functionality and
facts). Orchestra needs just user/asset/vehicle owner detail from a stall or stage perspective. ... I think just the
twin needs the CAD level detail."*

The twin depot has three machines: a charge arm at each of the 30 fast chargers, a cleaning arm over each of them,
and a wash machine in each of the 3 wash bays. Each app shows them at its own level:

| App | Who reads it | What it shows |
|---|---|---|
| OTTO-TWIN, Robotics tab | engineering | the CAD: every part, the clash check, the service lines, the cycle on the model |
| OTTO-PULSE, Robots tab | depot crew, every owner's cars | what each machine does (steps, facts, safety), what each arm and wash bay is doing now, the run's results |
| OrchestrAV, each car's card | one fleet owner, its own cars | the machines at work on the car, the visit stage by stage with the station and the machine that does each stage, what each machine does to the car |

**What a machine IS has one home: the twin.** `ottoyarddepot-sim/src/lib/robotics/machineFacts.ts` builds
`public/cad/robot-facts.json` from the Robotics tab's own sources (the parts lists, the phase tables, the robot
specs), and a test fails when the file is stale. Both cockpits carry a byte copy at `src/lib/twin/robotFacts.json`.
**Change it in the twin, then copy it to both.** Never edit a cockpit's copy.

**What a machine is DOING is read from the feeds, never from that file** (`src/lib/twin/robots.ts`, identical in
both cockpits):

- charge arm: the snapshot's `stalls_status` (`tethered`, `tether_direction`, `tether_phase`), `arm.cycles` (open
  cycles, with retries), `arm.timings`, `arm.accuracy`;
- cleaning arm: the car's visit atom `interior_tidy` with `performed_by: 'stall_cabin_robot'`, `robot_stall_id`,
  `started_at`, `ends_at`, `done_at` (otto-q-core 0758);
- wash machine: the car's `bay_seat {since, ends_at}` while its state is `in_wash_bay` or `in_detail_bay` (0762);
- a car's stages: `ottoq_depot_cards` 1.5 `steps`, each with `to_stall_code` and `to_stall_kind` (0592);
- run totals: `ottoq_twin_kpi_board` (`service.by_service`) and `ottoq_twin_events_window.by_type`.

**Traps, each measured on run a0001fd6 (2026-11-07):**

- **The card feed holds ONE step under way** (the first active leg). A cabin clean or a sensor inspection that runs
  during the charge is not in `steps` until it is done. The cockpits add work under way from the visit's needs.
- **A leg does not say who did it.** The need does (`performed_by`). 158 of 201 done interior-tidy legs were the
  cleaning arm's. 6 itineraries hold a planned tidy leg the engine closed with no duration beside the arm's own leg:
  show one stage, the one that ran.
- **Every done wash and detail leg went to a wash bay** (59 of 59), so those stages are the wash machine's.
- **Through a charge, `tether_until` is a lease the engine renews** (120 s at a time), not the charge's end.
- **A car pulling out no longer points at its stall.** The arm's open cycle and the stall's tether still name it.

**Links both ways.** A cockpit opens the twin on one machine: `/?tab=robotics&robot=<charge_arm|clean_arm|wash_gantry>`.
The twin's Robotics tab opens PULSE on its Robots tab: `/?tab=robots` (with `source=twin&run=` while a run is live).

### One set of words, and no vendor product on any screen (added 2026-11-10)

Chase, 2026-10-07: *"I want to remove all specific tool naming from our descriptions. So for instance, I don't want to
specifically name which Nvidia tools or specifically googles CP – SAT."* A model or a solver is named by what it does:
the agent, the lexicographic planner, the GPU planner. The twin held this from October. Three PRs of 2026-11-10 bring
the cockpits to it: ottoyarddepot-sim #160, ottoyard-field-ops #31 and ottoyard-OTTO-Q #37.

**The words have one home: the twin.** Both cockpits carry these files in `src/lib/twin/`:

| File | In a cockpit it is |
|---|---|
| `publicNames.ts`, `plainWords.ts`, `eventFeed.ts`, `secondLoop.ts` | a byte copy of the twin's `src/lib/` file (`cmp` proves it) |
| `decisionText.ts` | the twin's, with its one row-type import changed |
| their tests | the twin's, with import paths adjusted, in `__tests__/` |

**Change the twin's copy first, then copy it to both.** Never edit a cockpit's copy. `decisionText.ts` imports
`publicNames.ts`: carry them together.

**The scan, and what it cannot see.** `publicNames.test.ts` reads every string an app writes (literals, templates,
JSX text) and fails on a product name with its file and line. It runs in all three apps. It cannot see a name that
arrives as **data**. Measured on otto-q-core, 2026-11-10:

- **The activity feed's `engine` column is the model's id on an agent decision**
  (`nvidia/nemotron-3-ultra-550b-a55b -> cpsat_service`): 5,304 of 9,360 agent decisions. Both cockpits drew it as
  hover text. **Never draw `engine`, `agent_model` or a provider key as it comes.** `proposedBy()` names who proposed
  a decision and never draws a model's id.
- **The agent's own words can name a product or a file of ours.** It quoted `CLAUDE.md` by name on 10 of 9,360
  agent decisions. Its summary, its directives and its reason go through `publicText()`. So does an upstream error.
- A decision's reason, verb, action and source keys named no product on 62,788 decisions of two runs.

**OrchestrAV's settings** had an "AI Model" picker that named three vendors' models and saved a value nothing read.
It is removed, and `profiles.ai_preferences_jsonb.model` is still saved as it was. Payment and sign-in brands stay
where a buyer needs them (checkout, billing, the sign-in button).

**The twin's old Intelligence, Orchestration and Events tabs are removed (2026-11-10, in ottoyarddepot-sim #160).**
They left navigation on 2026-09-30 and sat in the tree unmounted, and the Intelligence tab drew the model's id and raw
stack keys from data. Two things in the twin now look dead and are not: nothing there imports `src/lib/eventFeed.ts`
(both cockpits draw it), and `src/lib/intelligenceStack.ts` is types only. To find code an app cannot reach, walk the
imports from its entry points. A search for a file's name counts a mention in a comment as a use: it found four of the
six unreachable tab components.

---

## 4. Start-of-run variable selection (the other named lane)

Chase specifically wants **variable selection at the start of a new run** to be a real, visible
control — not a hidden config. Today:

- `ottoq_start_demo_run(scenario, speed, days, seed)` is the entry point.
- Six featured scenario **quick-launch chips** exist in the depot-sim OperatorConsole Run Control:
  Normal Day / Heat Wave / Winter Storm / Demand Charge / Charger Outage / Peak Turnover. The full
  deck list stays in a dropdown, and the running scenario is highlighted.
- Eight scenario decks exist in `ottoq_sim_scenarios` (see `03_OTTOTWIN_ARCHITECTURE.md` §3).

**The gap:** the operator cannot currently choose the *variables* — fleet size, arrival rate, needs
density, charger fault rate, staffing, weather severity, grid stress — at run start, only a
preset deck. And the deck override JSON does not cover the `_rates` knobs at all.

**What good looks like:** the operator console offers a preset **and** an "advanced" panel exposing
the real variability knobs, writes them into the run's variability profile, and the twin's boot
manifest (`ottoq_twin_boot_manifest`) reports back exactly what was drawn — so the run is both
configurable and reproducible from its seed.

⚠️ **Before building this, resolve the seed pin.** Every recent run uses seed `424242`, so runs are
currently identical regardless of what the UI offers. The pin has not been located.

---

## 5. Branding — the standing UI mandate

Chase has a standing want: **a full UI audit of OrchestrAV and OTTO-PULSE to match OTTOYARD
branding. No lazy UI.** Both currently use generic shadcn themes.

**Design tokens**, extracted from `~/Desktop/OTTOYARD/OTTOYARD-Brand-Guide.html`:

**Theme: dark, high-contrast, premium, techy, red-accented.**

```
Backgrounds   --bg-0  #06070A   (near-black base)
              --bg-1  #0A0B0E
              --bg-2  #111317   (cards / panels)
Ink           --ink       #E7EAF0   (near-white)
              --ink-dim   #8A8F99   (secondary)
              --ink-faint #4A4E57   (muted)
Brand red     --red      #C8102E   (primary accent)
              --red-hot  #E8293F   (hover / active)
              --red-deep #8E0B20   (pressed / deep)
Lines         rgba(255,255,255,0.06) subtle · rgba(255,255,255,0.12) strong
Accents       mint #C9E0D4 · pink #E8C9DC · blue-white #EAF0FF · gray #D8DCE2
              (status/category only, never primary)
```

**Type:**
- **Chakra Petch** — display, headings, labels (techy, geometric)
- **Inter Tight** — body and UI text
- **JetBrains Mono** — data, metrics, codes, sequences (ideal for stall codes and OTTO-Q sequences)

**How to apply:** map these into each app's `index.css` `:root` (`--background`, `--foreground`,
`--primary` = red, `--card`, `--border`, `--muted-foreground`) and `tailwind.config.ts`
`fontFamily` (sans = Inter Tight, mono = JetBrains Mono, display = Chakra Petch); load the three
Google Fonts. **Keep dark mode as the default.** OTTO-Q's engine and agent surfaces should lean on
red + JetBrains Mono for a command-console feel.

⚠️ `ResponsiveGuard` in the depot-sim requires ≥1200px width. Test at 1440×900 and above.

---

## 6. A hard-won operational warning

**The Mapbox incident.** A leaked Mapbox token cost $1–2k. **Policy: MapLibre with free tiles, never
metered raster, never a committed token.** If any map work happens in these cockpits, that policy is
binding.
