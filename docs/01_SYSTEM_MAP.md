# 01 — System Map: the five systems and where everything lives

This is the "where do I find X" file. Verified against GitHub, the live Supabase org, and the local
Desktop clones on **2026-08-08**.

---

## 1. The five systems

| System | What it is | Decides? | Owns world state? |
|---|---|---|---|
| **OTTO-Q** | The orchestration **brain**. Consumes world state, produces decisions. This is the product and the IP. Ships to real depots. | **Yes — only this** | No |
| **OTTO-TWIN** | The **world generator** / digital twin. Manufactures maximally real, maximally random inputs (AV telemetry, OCPP, grid, weather, faults, wear) to stress-test the brain, then plays out OTTO-Q's orders. | Never | **Yes** |
| **OrchestrAV** | The **fleet-owner cockpit**. A fleet operator (e.g. Waymo Nashville) logs in and sees only their vehicles. | No | No |
| **OTTO-PULSE** | The **depot-ops cockpit**. Depot staff see everything, all owners. Also called "field-ops". | No | No |
| **OTTOW** | Tow / roadside dispatch. Small, largely dormant. `ottow_missions/drivers/notifications`. | No | No |

**The law:** *OTTO-Q decides. OTTO-TWIN executes and owns world state. The renderer only draws.*

**The swap test** is the investor and OEM pitch in one sentence: *unplug the twin, plug in real
depots, and OTTO-Q cannot tell the difference.* Everything the twin produces crosses a seam shaped
like a real-world feed contract. If you ever find yourself building a code path that only works
because it is a simulation, you have broken the pitch.

⚠️ **A naming trap that has confused every reviewer:** the GitHub repo named
**`ottoyard-OTTO-Q` is NOT the OTTO-Q brain.** It is **OrchestrAV**, the fleet cockpit. The brain
lives in `otto-q-core`. Read `memory/project_architecture_separation.md` before touching repos.

---

## 2. Databases (Supabase, org `gdqqjpxbgpperhgyfkes` "OTTOYARD")

| Project | Ref | Region | Status | What it is |
|---|---|---|---|---|
| **otto-q-core** | `gxdrcyphqjzjsuhxuqtg` | us-east-1 | ACTIVE | **The fused core.** OTTO-Q + OTTO-TWIN + OTTOW in one Postgres 17.6 database. ~130 tables, ~400 functions, 27 edge functions, 5 cron jobs. **This is where essentially all engineering happens.** |
| **OTTOYARD MVP** | `ycsisvozzgmisboumfqc` | us-east-2 | ACTIVE | **OrchestrAV's own older database.** A separate, simpler fleet SaaS: billing/Stripe, profiles, user_roles, an `intelligence_events` table with ~1.35M rows, its own mock simulator, ~35 edge functions. **Not connected to otto-q-core.** |
| **OTTOYARD Fleet Dashboard** | `sovyxwtrqfmizelrammm` | us-east-2 | **INACTIVE** | Paused/abandoned (created 2025-08-16). Candidate to archive. |
| `hfjaofyfxsyniohdfacg` | — | — | **DOES NOT EXIST** | A dead project id still referenced in some `config.toml`/`.env` files. If you see it, it is a stale reference — the real target is `gxdrc`. |

⚠️ **The `ottoq_` naming collision.** Both `ycsis` and `gxdrc` have tables named `ottoq_depots`,
`ottoq_vehicles`, `ottoq_resources` etc. **They are different systems with the same names.** Always
confirm which project ref a client points at before reasoning about a table.

> 🚨 **AND THE BIGGER TRAP: every `supabase/config.toml` in every repo points at the WRONG project.**
> Measured across all clones, the `project_id` values found are `hfjaofyfxsyniohdfacg` (**does not
> exist**), `odhpbdhnpcrjeaxvbrzd` (**dead**), `ycsisvozzgmisboumfqc` (real, but that is OrchestrAV's
> legacy DB), and the placeholder string `OTTO-Q_V1`.
> **`gxdrcyphqjzjsuhxuqtg` — the real core — appears in none of them.** It is hardcoded in client
> code instead (`src/lib/supabase.ts`, `src/lib/ottoTwin.ts`, `src/lib/otto-q-api.ts`).
> ⇒ **Any Supabase CLI command that writes will target the wrong project unless you pass
> `--project-ref gxdrcyphqjzjsuhxuqtg` explicitly.** Do not "fix" the config files without telling
> Chase — the edge functions genuinely deploy to `gxdrc`, so a changed linked ref has deployment
> consequences.

---

## 3. GitHub repositories (account `OTTOYARD`)

All under `https://github.com/OTTOYARD/`. Default branch is `main` on all of them.

⚠️ **`OTTOYARD` is a personal GitHub account, not an organization.** Everyone calls it "the org" in
conversation, which is harmless in prose and a 404 in an API call — use `/user/repos`, never
`/orgs/OTTOYARD/repos`.

| Repo | Private | What it is | Local clone |
|---|---|---|---|
| **`otto-q-core`** | yes | **The brain, in files.** Migrations, DB baseline DDL, edge-function source. The source of truth for OTTO-Q. | `~/Desktop/OTTOYARD/otto-q-core` |
| **`ottoyarddepot-sim`** | yes | **OTTO-TWIN's cockpit + renderer.** The Three.js/React depot simulator, site plan, motion engine, Omniverse RTX embed, operator console. | `~/Desktop/OTTOYARD/ottoyarddepot-sim` |
| **`ottoyard-field-ops`** | yes | **OTTO-PULSE** — the depot-ops cockpit. | `~/Desktop/OTTOYARD/ottoyard-field-ops` |
| **`ottoyard-OTTO-Q`** | **public** | **OrchestrAV** — the fleet-owner cockpit. (Misleading name — see the trap above.) | `~/Desktop/OTTOYARD/ottoyard-OTTO-Q` |
| **`ottoq-intelligence`** | **public** | **The frontier stack** — a Python FastAPI service for optimizers and ML that cannot live in Postgres (energy MILP/MPC, forecasting, cuOpt assignment, Nemotron orchestration). | `~/Desktop/OTTOYARD/ottoq-intelligence` |
| **`otto-q-core-snapshot`** | yes | Read-only schema snapshots of the live DB, refreshed periodically. Useful for diffing. | `~/Desktop/OTTOYARD/otto-q-core-export` |
| **`otto-q-workspace`** | yes | The founder's original working sandbox: ~90 design/spec/certification markdown docs, the original TypeScript engine scaffold, seed SQL. **Documentation gold, code mostly superseded.** | `~/Desktop/OTTO-Q V1` |
| **`OTTOYARD-SITE`** | yes | Marketing website. | — |
| **`OTTOYARD`** | public | Org profile repo. | — |

**Also on disk, not a git repo:** `~/Desktop/OTTOYARD/depot-sim-motion` — a working copy of the
depot-sim used for motion experiments. Treat as scratch.

⚠️ **Two chats sharing one repo folder share one working tree.** If you and Claude both work in
`~/Desktop/OTTOYARD/otto-q-core`, you will clobber each other. **Use `git worktree` per session, or
clone into your own directory.** See `memory/project_parallel_session_tree_collision.md`.

---

## 4. Where each thing actually lives

### The decision brain (OTTO-Q)

**Primary home: the live database `gxdrcyphqjzjsuhxuqtg`.** Almost all brain logic is PL/pgSQL
functions. The `otto-q-core` repo mirrors them.

- `otto-q-core/db/migrations/0001..0010_*.sql` — the numbered migration series (the *new*,
  file-first regime). `0010_unify_depot_layout.sql` is **authored but NOT applied** — see
  `docs/10_KNOWN_ISSUES.md`.
- `otto-q-core/db/baseline/` — dumped DDL: `functions_public.sql`, `functions_ottoq.sql`,
  `functions_twin.sql`, `tables.sql`, `rls_policies.sql`, `cron_jobs.sql`. **Read these to
  understand the brain without hitting the DB.**
- `otto-q-core/db/checks/depot_geometry_guard.sql` — the layout validity guard.
- `otto-q-core/edge-functions/` — edge function source, plus `_MANIFEST.md`.
- `otto-q-core/MIGRATION_LOG.md` — which migration landed at which DB version.

> **The rule the repo states in its own README, in bold:** *If it isn't a committed file, it didn't
> happen.* Every brain change is written as a migration file, committed, and **then** applied from
> that file — in that order. This rule exists because the live DB once had **621 applied migrations
> while the founder's working folder had 80 files with zero overlap**.

### The world model (OTTO-TWIN)

- **Engine:** PL/pgSQL functions in the `twin` schema of `gxdrc` (~70 relocated functions,
  `twin.ottoq_sim_*`, `twin.ottoq_world_advance`). Mirrored in
  `otto-q-core/db/baseline/functions_twin.sql`.
- **Cockpit + renderer:** `ottoyarddepot-sim`.
  - `src/lib/sitePlan.ts` — **the depot's geometry and lane network** (~349 lines, user-locked
    2026-06-10). Logical 2D coords → 3D world. Canopies, gap lanes, collectors, aisles, ingress/
    egress. This is the canonical lot.
  - `src/engine/SimulationEngine.ts`, `src/engine/TwinMotionDriver.ts` — motion. The driver
    interpolates along real lanes using Yuka steering; the twin owns the discrete truth.
  - `src/hooks/useTwinSceneBridge.ts` — the scene bridge (twin state → renderer).
  - `src/lib/ottoTwin.ts` — twin client, calls edge fn `otto-twin-control`. **Hardcoded to
    `gxdrcyphqjzjsuhxuqtg`.**
  - `src/components/photoreal/OmniverseViewer.tsx` — the RTX/WebRTC embed.
  - `src/components/tabs/Twin*.tsx` — Orchestration, Scorekeeper, SwapTest, Copilot, KPIs, Alerts,
    History, WorldContract.
  - `unreal/` — layout seed generation (`layoutSeed.sql`, `layoutSeed.json`) + USD assets.
  - `scripts/exportSitePlan.mjs`, `buildLayoutSeed.mjs`, `checkLayoutGeometry.mjs`.

### The cockpits

**OTTO-PULSE** (`ottoyard-field-ops`) — Vite + React + shadcn + TanStack Query.
- Tabs: `src/components/tabs/{Overview,Depot,Vehicles,Energy,Waves,Incidents,Comms,OttoQ}Tab.tsx`
- ⚠️ **Use `src/lib/supabase.ts`** (hardcoded to `gxdrc`). **NEVER**
  `src/integrations/supabase/client.ts` — it points at a dead project.
- `DEPOT_ID` is hardcoded as `11111111-...` in ~6 files. Accepted convention.
- Data access pattern: RPC + `refetchInterval ~10s`. **Polling only, no realtime.**
- Card style: hand-rolled `div.rounded-lg.border.bg-card`; status-chip + timeline-dot patterns live
  in `VehiclesTab.tsx`.
- **Honest empty states, no mock fallback.** Follow the PrimeOps pattern.

**OrchestrAV** (`ottoyard-OTTO-Q`) — Vite + React + shadcn.
- Pages: `src/pages/{Index,FleetCommandOttoQ,OrchestraEV,Incidents,Auth,AdminUsers,...}.tsx`
- `src/lib/otto-q-api.ts` — `ottoqRpc` (PostgREST rpc against `gxdrc`) and `ottoqInvoke`.
- Vehicle card: inline JSX in `src/components/OTTOQFleetView.tsx` (~lines 627–774).
- ⚠️ **Auth is on the `ycsis` project (legacy).** `useOTTOQRealtime` subscribes to `ycsis` tables,
  which are dead for `gxdrc` views — **poll instead.**
- The Fleet tab already reads real `gxdrc` vehicle UUIDs via `ottoq-fleet-vehicles`, and that edge
  function already accepts a `fleet_operator_id` filter.

### The frontier / ML stack

`ottoq-intelligence` — Python FastAPI, deployed to EC2 (`deploy/AWS_PLAYBOOK.md`,
`deploy/DEPLOY_EC2.md`, `Dockerfile`).
- `app/optimizers/` — energy MILP/MPC using HiGHS. **Live and tested.**
- `app/forecasters/` — probabilistic arrivals/load/SoC. Stub.
- Doctrine stated in its own README: *model proposes → optimizer disposes → shield guarantees →
  loop learns.* Every output is **advisory** to the deterministic shield.

### The documentation archive

`~/Desktop/OTTO-Q V1` (repo `otto-q-workspace`) holds ~90 markdown specs. The ones worth reading:

| Doc | Why |
|---|---|
| `RESUME_HERE.md` | Rolling status log with dated updates. Read the top. |
| `OTTOYARD_ARCHITECTURE_TRUTH_AND_SEPARATION_PLAN.md` | The canonical as-built map (2026-07-13) and the P0–P6 separation plan. |
| `OTTOQ_PHASE1A_ARCHITECTURE_MASTER.md` | 169 KB. The locked architecture spec: 8 blockers, 35 open decisions, 6-tier build order. |
| `OTTOQ_SEAM_MAP.md` | The bidirectional TWIN ⇄ OTTO-Q ⇄ {cuOpt, Nemotron} seam map. |
| `OTTOQ_TABLE_OWNERSHIP_MANIFEST.md` | Which product owns which table. |
| `OTTOQ_RETURN_TRIGGER_SPEC.md` (65 KB), `OTTOQ_APPOINTMENT_BUILDOUT.md` (67 KB) | The appointment/return-signal design. |
| `OTTOQ_VISUAL_LAYER_PLAN.md` | The Isaac Sim / Omniverse plan. |
| `OTTOQ_OEM_INTEGRATION_CONTRACT.md`, `OTTOQ_OEM_READINESS*.md` | The OEM-facing interface control document and readiness scorecard. |
| `OTTOQ_SERVICE_OPS_RESEARCH.md` | Fact-checked depot servicing: 4 categories; cadences/durations are tunable **assumptions**, not facts. |
| `OTTOQ_HONEST_CERT_2026-07-26.md`, `OTTOQ_BASELINE_FAIRNESS_RECERT_2026-07-26.md` | The certification receipts. Read before quoting numbers. |
| `OTTOQ_GAPS_REGISTER.md` | The founder-facing gap register. |
| `OTTOQ_DEPOT_TRAFFIC_SPEC.md`, `OTTOQ_TIMING_CALIBRATION.md` | Motion and timing. |

⚠️ Many of these predate the current architecture. **Cross-check dates against `memory/`.**

### Infrastructure outside the repos

- **AWS EC2 `g6e.2xlarge`** (NVIDIA L40S 48 GB) — runs the NVIDIA Isaac Sim container
  (`nvcr.io/nvidia/isaac-sim:6.0.1`) headless with WebRTC streaming, which the depot-sim RTX tab
  embeds. Relaunch script `~/ottoq/launch_isaac.sh` on that box. **Stop it when idle (~$2–4/hr).**
  The public IP changes on stop/start; the cockpit overrides it via
  `localStorage('ottoq_omniverse_ip')`.
- **NVIDIA cuOpt** — hosted API at `optimize.api.nvidia.com/v1/nvidia/cuopt`. **No local GPU
  needed.**
- **NVIDIA Nemotron** — hosted NIM, model `nvidia/nemotron-3-ultra-550b-a55b`.
- **Lovable** — both cockpits and the depot-sim were originally scaffolded in Lovable and keep
  full GitHub sync. Platform decision (2026-06-27): **stay on Lovable, no migration.** Complex
  visual code is hand-authored in the repo via GitHub sync, because Lovable's generation degrades
  above ~300-line components.

---

## 5. Quick reference: the ids you will need constantly

| Thing | Value |
|---|---|
| Core Supabase project ref | `gxdrcyphqjzjsuhxuqtg` |
| Core Supabase URL | `https://gxdrcyphqjzjsuhxuqtg.supabase.co` |
| Production/flagship depot id | `11111111-1111-1111-1111-111111111111` |
| Benchmark/cert depot id | `22222222-2222-2222-2222-222222222222` |
| API gateway | `/functions/v1/otto-q-api/api/v1/...` |
| The pinned demo seed | `424242` (**see the known-issues file — this pin is a defect**) |
| The main demo scenario | `busy_day` |
| Fleet size on the flagship | ~116–220 vehicles depending on scenario |
| Stalls | ~300–320 rows; 10 DCFC, ~35 L2, 3 wash bays, 2 service bays, **0 detail bays**, ~100+ staging |
