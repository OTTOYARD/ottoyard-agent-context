# 14 — Glossary

## Systems and products

| Term | Meaning |
|---|---|
| **OTTOYARD** | The company. AV/EV depot infrastructure. Founder: Chase Ballenger. Flagship: Nashville. |
| **OTTO-Q** | The orchestration **brain**. The product and the IP. The only thing that decides. |
| **OTTO-TWIN** | The digital twin / **world generator**. Manufactures realistic unscripted inputs and plays out OTTO-Q's orders. Never decides. |
| **OrchestrAV** | The **fleet-owner** cockpit. Repo `ottoyard-OTTO-Q` ⚠️ (misleading name). |
| **OTTO-PULSE** | The **depot-ops** cockpit. Repo `ottoyard-field-ops`. Also called "field-ops". |
| **OrchestraEV** | A retail EV membership surface inside OrchestrAV. Lower priority. |
| **OTTOW** | Tow / roadside dispatch. Small, largely dormant. |
| **OTTO-Q PRIME** | The Nemotron conductor agent (`ottoq-orchestrator-agent`). |
| **OTTOCOMMAND** | The natural-language command interface. ⚠️ Its model is **Claude**, not Nemotron. |
| **The Black Box** | The founder-side flight recorder — each run's exact executed code plus all raw data, downloadable as a bundle. |
| **The swap test** | The pitch: unplug the twin, plug in real depots, OTTO-Q cannot tell the difference. |

## Architecture

| Term | Meaning |
|---|---|
| **Simplex / Runtime Assurance** | The named avionics safety architecture OTTO-Q implements: a performant layer proposes, a deterministic layer vets. |
| **L1** | The deterministic **safety shield**. 52 rules, 29 active. Non-raising, fail-closed. Has the final word. |
| **L2** | The **performant** layer. Proposes actions. May be cuOpt, Nemotron, a heuristic, or a learned model. |
| **L3** | The **audit / grade** layer. Records decisions, rule evaluations, counterfactuals. |
| **The seam** | Any interface where one system's vocabulary crosses into another's. **Must be a total function.** |
| **The frozen frame** | A captured, sha256-hashed snapshot of world state that a decision is made against, so the world cannot shift mid-decision and the decision can be replayed. |
| **CRN** | Common Random Numbers. Same seed ⇒ same world for every policy arm, so A/B differences are attributable to the policy. |
| **The forward calendar** | `ottoq_stall_bookings` — the forward-time occupancy model of every space. **This is what makes OTTO-Q an orchestrator.** |
| **The metronome** | `ottoq_demo_metronome`, pg_cron job 12, every minute. **The run engine.** Alternates DECIDE and FIRE beats. |
| **The approach band** | A derived read-only zone from the ETA clock: `ottoq_approach_zone` + view `ottoq_approach_band`. **Not** a vehicle state. |
| **Feed mode** | `depots.feed_mode` ∈ `{'sim','external'}`. External switches off every twin fabricator and keeps the brain loop — **the production cutover is a feed swap.** |

## Depot operations

| Term | Meaning |
|---|---|
| **Stall** | Any addressable space: `dcfc`, `l2`, `wash_bay`, `service_bay`, `staging`. ⚠️ **There are ZERO `detail_bay` stalls** — detail shares the 3 wash bays. |
| **DCFC** | Direct-current fast charging. ~150 kW. 10 at the flagship. |
| **L2** | Level-2 AC charging. ~11 kW. ~35 at the flagship. A full charge from 28% is ~5 h on L2 vs ~22 min on DCFC. |
| **Bay** | A wash, detail, or service space (as opposed to a charger or a staging spot). |
| **Temp staging** | `staging_role='temp'`, 24 interior stalls. **Short, transient, mid-workflow.** |
| **Perimeter staging** | `staging_role='long'`, 176 lot-edge stalls. **Longer-term and the overnight hold.** |
| **Atom** | A single unit of required work on a visit (charge, exterior_wash, interior_deep_clean, sensor_calibration, brake_inspection, …). ⚠️ **The JSON key is `'svc'`**, not `'atom'`. |
| **Needs card** | `ottoq_vehicle_needs_card` — a 66-column view, one row per vehicle, holding everything OTTO-Q knows about what that vehicle requires. |
| **Manifest** | The set of service atoms generated for a visit. |
| **Itinerary / leg** | The timed plan for a visit. `ottoq_vehicle_itineraries` / `ottoq_itinerary_legs`. **Twin-owned.** |
| **Visit** | One depot stay. **Atomic** — full charge plus every required service before redeploy. |
| **The wave** | The overnight return: vehicles arrive 22:00–00:00 in timed waves, ~1% stay out until 03:00, redeploy 04:30–05:00. |
| **The deploy floor** | The minimum SoC to redeploy. **80%**, and it should be a per-operator SLA. |
| **The rush valve** | `deploy_release_per_tick_cap` — OTTO-Q's self-imposed release throttle. The baselines have no equivalent. |
| **Gate backlog** | Vehicles queued at the gate. ⚠️ **A gate queue is a doctrine violation**, not a tunable. Vehicles must never queue at a gate. |
| **Unsafe deploy** | Deploying below the stranding floor. The count that must always be zero. |
| **The exception triad** | Charger-fault auto-reroute (S3a) · vehicle-fault tech-approval (S3b) · bay-fault auto-reroute (S3c). |

## Energy

| Term | Meaning |
|---|---|
| **BESS** | Battery Energy Storage System. The depot's stationary battery. |
| **Demand charge** | A utility charge on the **15-minute peak kW**, ~$10/kW. **30–74% of a commercial/DCFC bill** — the single strongest quantified prize. |
| **Peak shave** | Reducing that 15-minute peak, principally with the battery. |
| **TOU** | Time-of-use energy pricing. |
| **LMP** | Locational Marginal Price — the wholesale grid price signal. |
| **DR** | Demand Response — a utility event asking for load reduction. |
| **MPC** | Model Predictive Control. The rolling-horizon MILP that schedules BESS and charging. |
| **Ratchet** | The billing mechanic where a peak sets a floor for future months. |
| **EcoStruxure** | Schneider's energy management system — the external EMS OTTO-Q **signals** but never actuates. |

## Protocols and integration

| Term | Meaning |
|---|---|
| **OCPP** | Open Charge Point Protocol. Charger ↔ management system. |
| **MQTT** | The de-facto connected-vehicle transport. Topics: `fleet/{veh}/telemetry/bsm`, `/cmd`, `/cmd/ack`, `/assist/req`, `/assist/resp`. |
| **SAE J2735 BSM** | Basic Safety Message — the telemetry payload shape, plus EV and AV extensions. |
| **ISO 20078** | Extended Vehicle (ExVe) — the message envelope. HTTPS/REST + OAuth2/OIDC. |
| **SAE J3016 / BSI PAS 1886** | Remote assistance / teleoperation roles (monitor / assist / drive). |
| **ISO 15118 / V2G** | The **charger-cable** protocol. ⚠️ **Not** the telemetry path — that side is modelled via OCPP. |
| **ICD** | Interface Control Document — the OEM-facing contract artefact. |

## NVIDIA and AI

| Term | Meaning |
|---|---|
| **cuOpt** | NVIDIA's GPU optimization service. Used via **hosted API** — no GPU purchase. |
| **Nemotron** | NVIDIA's LLM family. `nvidia/nemotron-3-ultra-550b-a55b` via hosted NIM. |
| **NIM** | NVIDIA Inference Microservice. |
| **Isaac Sim** | NVIDIA's robotics simulator on Omniverse. Runs the photoreal Tier-B twin. |
| **Omniverse / Kit / USD** | The rendering platform, its app framework, and the OpenUSD scene format. |
| **CIL** | Continuous Improvement Loop — `ottoq_cil_propose` / `ottoq_cil_tick`. Hill-climbs policy parameters, A/B's each candidate forward through the twin, adopts only strictly-better winners under a 0-unsafe hard gate. |
| **LAP** | Linear Assignment Problem. **Polynomial** — greedy is provably near-optimal, so an optimizer can only tie. |
| **HiGHS** | The open-source LP/MILP solver behind the energy MPC. |
| **CP-SAT** | Google OR-Tools' constraint solver. A candidate rival to cuOpt for the scheduling regime. |

## Certification and measurement

| Term | Meaning |
|---|---|
| **Arm** | One policy's run in an A/B (otto_q / greedy / fifo / manual). |
| **CRN-paired** | Arms sharing a seed so differences are attributable to the policy. |
| **Cert / receipt** | A certification run and its written evidence document. |
| **Vacuous guard** | A check that cannot fail. Multiple have existed here; **prove a guard has fired at least once**. |
| **Phantom capacity** | The defect where a stall reads free when it is occupied. Twice a source of invalid baselines. |
| **Orphan deploy** | A deploy logged with no matching dispatch. |
| **Denominator** | ⚠️ **Always state it.** 98.8% and 68.1% were both true of the same run. |

## Metrics — safe and unsafe to quote

| ✅ Quote these | ❌ Never quote these |
|---|---|
| `trips_completed` | `vehicles_turned_around` |
| `vehicles_cycled` | `fleet_ready_pct` |
| `productive_deploys` | `gate_backlog` |
| grid peak kW (run-scoped) | `deviation_s` (corrupt baseline) |
| unsafe deploys (always 0) | `charge_sessions` from any run scored before 2026-07-26 |
| bookings by state, clipped to the run window | any booking-derived capacity number from before 2026-08-01 |

## People and process

| Term | Meaning |
|---|---|
| **Chase** | Chase Ballenger, founder/CEO. Non-engineer, deeply technical on product and operations. **The only person who merges.** |
| **Claude Code** | The reviewing senior engineer agent. Built most of this system. Deepest context. |
| **Hermes** | The autonomous implementation engineer agent. You, if you are reading this. |
| **Doctrine** | A founder ruling on how the system must behave. **Binding.** Violating one is a defect. |
| **The gap register** | `docs/10_KNOWN_ISSUES.md`. Chase should never be the first to find a gap. |
