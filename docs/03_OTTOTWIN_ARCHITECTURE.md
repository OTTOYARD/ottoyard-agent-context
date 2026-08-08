# 03 — OTTO-TWIN: the world model

> **This is your first major work lane. Read it twice.**

## 1. What the twin is *for*

OTTO-TWIN is not a demo animation. It is a **world generator** whose job is to manufacture
maximally real and maximally *random* conditions so that OTTO-Q can be proven to handle them.

Chase's framing, and it should govern how you build:

> The twin is meant to be **almost a video game — a 4K, hyper-realistic world model**. The realism
> is not decoration. A photoreal, physically-plausible, continuously-moving depot is the *proof
> surface* for the OTTO-Q intelligence layer. If a viewer can watch vehicles arrive from the city,
> queue, take an assigned lane, pull into a specific stall, dwell for a physically sensible time,
> move to a wash bay, and leave — and every one of those movements traces back to a decision OTTO-Q
> made and can justify — then the intelligence is *visible*, not asserted.

Two audiences, one artefact:
- **Investors** see a spectacle that makes the abstraction concrete.
- **OEM engineers** see a system whose world model is calibrated to real data, whose decisions are
  logged, and whose safety layer is deterministic — which is what makes them willing to greenlight
  a telemetry pilot.

**The swap test is the pitch:** unplug the twin, plug in a real depot's telemetry, and OTTO-Q cannot
tell the difference. Everything the twin emits crosses a seam shaped like a real feed contract.

## 2. The non-negotiable realism rules

**Unscripted, always.** The twin must field realistic **unscripted** demand. Nothing pre-programmed.
Vehicles arrive with *varying* service manifests. A demo that only works because the scenario was
rigged is worth nothing.

**Needs are a probability draw, not accumulated wear.** At every run start,
`ottoq_run_boot_draw(run)` → `ottoq_seed_vehicle_need_profiles(run)` draws ~24 per-vehicle condition
variables — tire tread mm, brake wear %, sensor health %, soil index, cadence counters — via
`ottoq_sim_seeded_random(seed, 'vnp:x:'||id)`. Set-based, order-independent, PK'd per vehicle.

**Measured density against Chase's stated 1-in-5-to-10 band — already in band:**
pm due 25.9% · calibration 33.6% · wash 29.3% · deep-clean 15.5% · fault ≤sev3 9.5% ·
tire <4 mm 17.2% · brake ≥65% 21.6%.

> 🚨 **DO NOT BUILD A NEW NEEDS DRAW. IT EXISTS AND IT WORKS.** Claude once asserted it didn't and
> was wrong. Full detail: `memory/project_needs_draw_already_exists.md`. Wear *modelling* — vehicles
> slowly accumulating damage over time — is an explicit **non-requirement**.

**Calibration, not replay.** The twin's randomness is fitted to **seven real corpora**, ingested
2026-05-22..28 into `ottoq_calibration_datasets/distributions/profiles` (16 distributions, 8
profiles, **only 1 fitted correlation**):

| Corpus | Content | Feeds |
|---|---|---|
| `acn_data` | Caltech ACN-Data, 130k+ real EV charging sessions | charge_duration, energy_delivered, dwell_time, SoC curves |
| `ca_dmv_av` | CA DMV AV disengagement + collision reports, 20k | incident_rate, incident_type_mix, edge_case_library |
| `eia_grid` | EIA Hourly Grid Monitor, TVA region (Nashville) | grid demand profile, carbon intensity, tariff windows |
| `nyc_tlc` | NYC TLC trip records, 3.5M trips | arrival rate by hour/day-of-week, trip duration |
| `noaa_nws` | NOAA NWS, Nashville KBNA | ambient temp, weather severity, precipitation, wind |
| `nrel_fleet` | NREL Fleet DNA, 15k | daily energy kWh, duty cycle, service interval |
| `charger_reliability` | UC Berkeley / JD Power / ChargerHelp composite, 40k | charger fault rate, fault mode mix, MTBF |

The decision was **hybrid**: calibrate randomized generators *to* these corpora (real parameters and
logic, still perturbable for novel stress) rather than verbatim replay, because replay cannot be
perturbed.

> ⭐ **The single biggest realism gap, and it is a great piece of work:** there is only **one fitted
> cross-variable correlation.** Independent random knobs are not realistic stress. Reality
> **co-moves** — a heat wave means AC load up *and* charge rate down *and* grid price up *and*
> arrival pattern shifted. Correlated variability is what turns the twin from a random-number
> generator into a statistical clone, and it is what makes any OTTO-Q-beats-baseline claim credible.

## 3. The twin engine (server side)

Lives in schema `twin` on `gxdrcyphqjzjsuhxuqtg` — **71 functions**, mirrored in
`otto-q-core/db/baseline/functions_twin.sql`.

- `twin.ottoq_world_advance()` — the production physics pass.
- `twin.ottoq_sim_advance_tick_world(run)` — the demo physics pass. Derives elapsed from
  `clock_timestamp() - last_tick_at`, clamped by a 10-minute anti-teleport ceiling, and anchors
  `last_tick_at` at tick **start** by design.
- `twin.ottoq_sim_advance_service_flow`, `ottoq_sim_advance_charge_sessions`,
  `ottoq_sim_auto_dispatch_tick`, `ottoq_sim_vehicle_exception_handler`,
  `ottoq_sim_generate_service_manifest`, `ottoq_sim_seed_fleet`, `ottoq_sim_confirm_commands`.
- Variability: `ottoq_twin_deal(...)`, `ottoq_sample_calibrated`, `ottoq_sample_shaped`,
  `ottoq_variability_instantiate`, `ottoq_twin_climate_stress`, `ottoq_twin_incident_weather_mult`.
- Feeds: `ottoq_telemetry_packets`, `ottoq_ocpp_messages`, `ottoq_weather_snapshots`,
  `ottoq_grid_snapshots`, `ottoq_solar_output`, `ottoq_oem_webhook_log`, `ottoq_comms_messages`.
- Read API for the renderer: `ottoq_twin_snapshot(run)`, `ottoq_twin_depot_layout(depot)`,
  `ottoq_twin_run_context(run)`, `ottoq_twin_fleet_condition(run)`, `ottoq_twin_boot_manifest(run)`,
  `ottoq_twin_events_window(run)`, `ottoq_twin_appointments(run)`.

**Scenarios** live in `ottoq_sim_scenarios`. Eight already exist — do not rebuild them:
`normal_day`, `busy_day`, `heat_wave`, `winter_storm`, `dr_event_cascade` (the demand-charge
headline), `charger_outage_morning_rush`, `grid_brownout_at_peak`,
`solar_underperformance_partly_cloudy`, `aggressive_fleet_turnover`.

Deck override schema: `weather_overrides{ambient_peak_c, ambient_floor_c, cloud_max_pct,
humidity_floor_pct}` · `grid_overrides{lmp_multiplier, dr_required_cap_kw, force_dr_call_hour,
force_bess_starting_soc_pct}` · `fleet_overrides{dispatch_rate_multiplier,
target_deployed_fraction}`.

⚠️ The engine *also* exposes `_rates` knobs (arrival/dispatch rate, ETA delay, DR likelihood,
charger fault, incident, staffing) via the variability profile — **but those are not in the deck
override JSON.** Building a "traffic surge" or "mass incident" deck requires first working out how
a scenario seeds `_rates`. That is an open question, not a solved one.

⚠️ **Density stacking trap.** `busy_day` applies `pm_km_scale 0.010` and `calib_h_scale 0.02`,
crushing `pm_interval_km` 8000 → **80 km** and `calib_interval_h` 250 → **5 h**. *That* is what
produces today's high mechanical-PM rate — not any probability. Adding a new draw on top would
double-count. If you touch either path, return the scenario scales to 1.0 in the same change or
explicitly subordinate one path.

## 4. The renderer (client side) — `ottoyarddepot-sim`

### 4.1 The geometry, and the yardstick you must never mix up

**1 plan unit = 0.4785 m = 1.57 ft.** The 2D plan (`src/lib/sitePlan.ts`, viewBox `0 0 300 220`) is
drawn to real dimensions, and the 3D frame is **1:1 with plan units** — `coordUtils.toWorld` is
`x3d = x2d - 150; z3d = 110 - y2d`, **no scale factor**.

Confirmed by geometry nobody tuned for it: `PARK_RUNS dy: 5.7` = 8.95 ft stall width; staging
stripes at ±2.9 = 9.11 ft; stall depth 10.5u = 16.5 ft. A 9.1 × 16.5 ft bay is textbook.

> ⚠️ **A different yardstick exists in the site-plan export: 1.5699 ft/unit.** Mixing the two caused
> a 1.57× outage. Always check which space you are in. See `memory/reference_depot_plan_scale.md`.

### 4.2 The lane network (RAILS)

`src/engine/motion/LaneGraph.ts` → `buildDepotLanes()` is a genuine **one-way directed road
network**:

- A **two-way divided ring**: south blvd y=172, north blvd y=74, west ave x=30, east ave x=275.
  Both directions exist as directed edges on one centreline, and `rightOffset = 2.4` shifts each to
  its own side, so opposing streams pass beside each other and **never head-on**.
- **One-way northbound** gap lanes through the canopy at `GAP_LANES x = [80, 126.5, 173.5, 220]`.
  Chargers face the wash/service bays.
- **One-way eastbound rear apron** (R0..R6 at `REAR_LANE_Y = 16`) draining the pull-through bays
  east — deliberately *not* extended west, so the graph can never route a car into the fenced BESS
  yard.
- Gates: `ingress → S_in` (**EAST**, x=200); `S_eg → egress` (**WEST**, x=100).

**Chase's confirmed flow doctrine (2026-07-18 — he chose all three, so do not "improve" them):**
1. Charging lanes stay **all northbound**, not alternating. Every charging car faces the same way;
   zero head-on risk; the ring loop-around is accepted.
2. Perimeter avenues stay **two-way divided**, not a one-way loop. Shorter routes; opposing traffic
   separated to its own side.
3. Gates stay **enter east / exit west**. Arrivals and departures fully separated; entry lands next
   to the temp staging block.

**Lane paint is generated, never hand-drawn.** `src/engine/motion/lanePaint.ts` derives every
marking (drive line, chevrons, stop bars, centre stripes) from the LaneGraph itself, rendered by
`LaneOverlay.tsx` (2D) and `three/Lanes3D.tsx` (3D). **Because every marking is derived from the
graph, painted right-of-way can never drift from routed motion.** Both renderers previously carried
hand-drawn arrows that *contradicted the actual rules* — the picture was lying about the traffic
rules. Do not reintroduce hand-drawn markings.

### 4.3 The lane budget, and the car that is four different sizes

`rightOffset = 2.4`, and `addRoad()` puts both directions on one centreline each offset right ⇒
**two opposing drive-lines are 4.8 units (2.30 m) apart.** `LANE_PAINT_WIDTH = 4.8` matches by
construction. **The paint has always been correct.**

The 2D car body was once `5.0 × 8.0` units — **wider than the 4.8-unit lane separation** — so two
cars passing in opposite directions overlapped by 0.20u of *body*. Clearance was **negative**. They
tracked their lanes perfectly and still collided. That is Chase's *"sometimes it looks like they're
heading right for each other."* Now **4.2 × 10.2 units (2.01 × 4.88 m)**, giving ~0.6u of daylight.

> 🔴 **STILL INCONSISTENT — four different cars exist. This is real, open work.**
> - 2D body: `4.2 × 10.2 u` — fixed and correct.
> - 3D `Vehicle3D`: `BoxGeometry(2.2, 0.85, 4.9)` — **authored in metres and dropped into
>   unit-space**, so it renders at ~48%: a toy car. Cheap fix is a uniform group scale ≈ 2.0.
> - Physics `traffic.ts CAR_LENGTH = 9` (4.31 m) — a third length, used by the IDM car-following
>   model.
>
> Changing `rightOffset` or `CAR_LENGTH` moves routed motion, so those need a **certified pass**,
> not a drive-by edit.

### 4.4 Motion

- `src/engine/TwinMotionDriver.ts` — keeps a **stable per-vehicle stall assignment** and, on each
  backend state change, routes the vehicle along the real lanes (`sitePlan.routeToStall` /
  `routeToEgress`) while a requestAnimationFrame loop interpolates. The prior bridge **teleported**
  vehicles to stalls on every 1.5 s poll, which is where the jitter came from.
- Yuka steering: `FollowPathBehavior` + `SeparationBehavior`. **Separation weight must stay low
  (0.35).** At its original 2.2 it was *stronger than* path-following and shoved cars sideways off
  the lanes — that was the "slide / don't turn / drag over railings and beams" complaint.
- `Vehicle3D.tsx` already rotates to face travel (`atan2`) and lerps at `delta·2·simSpeed`.
- **Clean split:** the twin owns discrete truth (which stall, which state); the renderer owns
  **only** interpolation. Zero world logic client-side.

### 4.5 Motion bugs: what was real, what was not

> ⚠️ **The "off-map render" bug does NOT reproduce.** It was recorded as an established fact and
> handed to a fresh session as a premise, costing it a wasted starting hypothesis. Verified against
> a captured real run (`busy_day`, 116 vehicles, 31 snapshots, 465 geometry samples): **0 off-map
> sightings.** *Re-verify a stored "established fact" before building on it.*

**The real bug was `LaneGraph.route()` displacing its own endpoints.** `route()` applied the
drive-on-the-right lane offset to the **whole polyline including both endpoints** — but `from` is
where the car physically *is* and `to` is the exact point it must reach; only interior road vertices
are centrelines. Measured drift: **3.20 units on both endpoints, against a 5.7-unit stall pitch.**

One bug, both symptoms:
1. A car starting a route **teleported 3.2u sideways into its neighbour's stall** ⇒ "piling into
   each other."
2. From that displaced start its path ran *inside* the parked neighbour, so the car-following model
   read a **negative gap and pinned speed to 0**. Every wedged car sat at `s=0`, and the 45 s
   watchdog rebuilt the identical route from the identical spot ⇒ "never drains."

Fixed by restoring the two endpoints after the shift. Measured on the fixture: body-overlap samples
**543 → 221**; wedged-car samples **297 → 130**; overlapping pairs 110 → 90. **Not zero — do not
claim it is.** Separately, the gate queue spawned cars inside each other (8u spacing for a 10.2u
car); fixed with pitch 11 plus a queue bound (4 per poll, 5th deferred, never dropped).

**🔴 Open: the east-avenue right-of-way conflict.** `EAST_AISLE_X = 275` with lane offset 3.2 ⇒ the
northbound lane body spans x 276.1–280.3, while an E-column car parked nosing east spans
279.4–289.6 ⇒ **0.9u of overlap on every northbound pass.** This is the dominant residual hotspot.
It needs a **layout decision**, not a motion fix: either move the carport, or drop the lane offset
to ≤2.3 (it was deliberately raised for passing clearance, and the paint tracks it). West avenue has
4.1u clearance and no hotspot — **the principled fix is to mirror the west.**

### 4.6 Method rules earned the hard way

- **Measure oriented body overlap** — the 4.2 × 10.2 box actually drawn — **never centre distance.**
  Perimeter stalls are pitched 5.7u apart, so a distance test flags every pair of parked neighbours.
  The first pass reported **31 phantom collisions** that way.
- **Sample during motion, not after the scene settles.**
- **Replay a captured fixture, not a live run.** These bugs are intermittent; a frozen recording
  makes a fix falsifiable. Fixture: `src/engine/__fixtures__/twinRun.busyday.json` + `replay.ts`.

### 4.7 Measured worse — do not re-propose

- **Routing cars onto the parking access aisle before departing:** 305 → **728** overlap samples. It
  funnels cars onto the aisle centreline into oncoming traffic, and the rail stepper deliberately
  ignores oncoming cars as leaders, so they drive *through* each other.
- **Correcting the parked heading** so temp-block cars nose away from their aisle: 305 → **378**.
- **Do NOT enable `separationSteer`** (`traffic.ts:74`). It is dead code, and its `radius = 5.5` is
  *larger* than the 4.8u opposing-lane separation, so every oncoming car would trigger up to 0.3 rad
  of steer — it would **manufacture** the head-on swerve.
- No A*, no Dubins paths were needed. The real causes were dull.

## 5. The visual layer — two tiers

**Tier A — web motion.** Three.js in the depot-sim cockpit, driven off the backend twin. $0, no GPU.
Lane paths, spacing, move times, congestion, variability. **This is where day-to-day realism work
happens.**

**Tier B — photoreal.** NVIDIA **Isaac Sim on Omniverse**, streamed into the cockpit over WebRTC.

Tier B is **built and working**:
- AWS EC2 `g6e.2xlarge` (L40S 48 GB, driver 595 open-kernel) runs
  `nvcr.io/nvidia/isaac-sim:6.0.1` headless with `omni.kit.livestream.webrtc`.
- Ports: **49100/TCP signaling, 47998/UDP media**, 8210/TCP.
- The depot USD auto-loads via `--exec /depot/open_depot.py` (waits 60 frames → `open_stage` → waits
  180 → adds a `UsdLux.DomeLight` only if none → computes the stage bbox and frames the camera).
- Relaunch is one command on the box: `bash ~/ottoq/launch_isaac.sh` (auto-detects the public IP via
  IMDSv2, chmods, runs the container; depot auto-loads ~6 min after start).
- The cockpit embed is `src/components/photoreal/OmniverseViewer.tsx` using
  `@nvidia/omniverse-webrtc-streaming-library@5.6.0` in DIRECT mode. The package is
  NVIDIA-proprietary and **not on public npm** — `.npmrc` scopes `@nvidia` to
  `https://edge.urm.nvidia.com:443/artifactory/api/npm/omniverse-client-npm/` (public, no auth) and
  is committed so CI resolves.
- View toggle in the cockpit: **2D / 3D / RTX**.

**Gotchas that cost days, so they are written down:**
- The Isaac container runs as sandbox user **uid 1234**. Files scp'd in are mode 600 ⇒
  "Permission denied" / "Can't find a file to execute". **`chmod a+r ~/ottoq/*` before launch.**
- The nvidia-container-toolkit graphics manifest hardcodes egl-wayland `1.1.13`/`1.1.20`; the host
  has `1.1.21`. Symlink both, then `ldconfig`.
- RunPod is a **dead end** for this: its containers cannot access the GPU render node
  (`/dev/dri/renderD*` owned `nobody:nogroup`, un-chmod-able inside a user namespace) so Vulkan
  reports "Found no drivers!". A full EC2 VM works because it has real `/dev/dri` access.
- A hand-built `kit-app-template` at 107.3 **segfaults in `librtx.scenedb.plugin.so`** due to
  carb CudaInterop v0.15-vs-v0.17 version skew. The prebuilt Isaac container avoids this entirely —
  **use the container.**
- EC2 public IP changes on stop/start. Override in the cockpit with
  `localStorage.setItem('ottoq_omniverse_ip','<ip>'); location.reload()`. **An Elastic IP would fix
  this permanently and has not been done.**
- **Stop the g6e when idle** (~$2–4/hr).

**Remaining Tier-B polish:** hide the Isaac editor chrome for a clean depot-only view; a tighter
cinematic camera; and — the big one — **push OTTO-Q assignments over the WebRTC data channel so
vehicles actually move in the photoreal twin.** Today the RTX view renders the depot but not live
motion.

## 6. The depot exists twice — and the copies disagree

> 🚨 This is a founder-ruled, half-finished piece of work. Read
> `memory/project_depot_layout_unification.md` before touching layout.

Two layouts were never connected: `public.stalls` (a database table) and `sitePlan.ts` (a
hand-authored code file). **A grep of the renderer for the DB's coordinate columns returns zero
hits — it never reads them.** Chase has only ever seen the renderer's.

**Nobody had ever drawn the database's depot.** Drawn for the first time, it fails on seven counts:
**54 overlapping stall pairs (76 of 150 stalls)** · **13 staging stalls inside the wash building**,
4 inside the BESS compound, 2 service bays inside the office · **20 of 45 charging stalls with no
aisle a vehicle can turn into** (12 ft clear; a 90° module needs 64) · 2 of 4 solar canopies shelter
no chargers · the 5 service/wash bays have **NULL** width and depth · `absolute_point` NULL on all
300 rows.

**Root cause: the site is over-programmed.** The DB fence is 1.82 acres holding a program needing
~3.3. The tell: the 5 stalls that make the DB say 35 L2 instead of 30 (`L2-STALL-21..25`) run past
the end of their own canopy and cause 10 of the 54 overlaps. Someone needed the count to be 35 and
appended rows.

**Founder rulings (binding):**
1. **The 3.3-acre rendered lot (452.13 × 313.98 ft) is the real parcel.** Import the renderer
   wholesale.
2. **The 2 service bays inside the office are intentional** (an attached service garage) —
   whitelist, do not "fix".
3. **Aisle standard is two-tier: ≥24 ft two-way, ≥20 ft one-way.** The single 20.5 ft one-way
   perimeter staging lane is accepted (clears the 20 ft fire-apparatus minimum). **Do not raise the
   bar.**

**Architecture:** the **DB stays the source** (everything that decides reads it); the **renderer
supplies the content**. Then renderer *and* the USD generator both read **from** the DB via
`ottoq_twin_depot_layout`. This is cheap because geometry is barely read — only ~4–8 call sites, and
**nothing currently computes a footprint, collision, swept path, or aisle width** server-side.

**Expect these consequences (direction only — we are rebuilding, not benchmarking):** mean pairwise
stall distance **144.64 → ~204 ft**, so travel legs lengthen, dwell rises, throughput falls. That is
a *realism improvement* — the old distances were computed on a lot that cannot physically exist.

## 7. Where the twin is weakest right now

Ranked, honestly:

1. **Only one fitted cross-variable correlation.** The world's variables move independently. This is
   the biggest realism gap in the system.
2. **The depot layout unification (migration 0010) is authored but not applied**, and must not be
   applied as written — it retires 19 stalls per depot, 13 of which carry real booking history, and
   `ottoq_stall_bookings.stall_id` is **ON DELETE CASCADE**. Applying it would silently vaporise
   ledger rows with no error.
3. **Tick cadence is irregular** (8–31 real seconds) while compute is ~1.15 s. Motion smoothness is
   bounded by the pg_cron metronome, not the engine.
4. **The 3D car renders at ~48% scale** and three different car lengths coexist.
5. **The east-avenue lane/stall overlap** (0.9u every northbound pass).
6. **The photoreal tier renders the depot but not live motion.**
7. **Seed 424242 is pinned**, so "completely new probabilities each run" is false today. The pin has
   not been located.
8. **The wash night gate** discards due washes on daytime runs — 34 of 116 vehicles were due a wash
   and **0 `exterior_wash` atoms were emitted** because the gate requires night. All three wash bays
   sat idle *by construction*.
