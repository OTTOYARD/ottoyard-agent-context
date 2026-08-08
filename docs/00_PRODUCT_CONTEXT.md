# 00 — Product Context: what OTTOYARD is and why this software exists

## 1. The company

**OTTOYARD** is an autonomous-vehicle and EV **depot infrastructure** company founded by Chase
Ballenger. The physical thesis: robotaxi fleets are scaling faster than the ground infrastructure
that keeps them running. A Waymo, Zoox, or Tesla robotaxi does not go home at night — it needs a
place to charge, get cleaned inside and out, get its sensors calibrated, get inspected, and get
parked. Today that is a leased lot with extension cords and a spreadsheet.

OTTOYARD builds and operates those depots. Nashville is the flagship location. Depot classes are
referred to internally as **Mini / Max / Ultra** by site size.

**The flagship depot spec** (founder, 2026-06-21) — this is the physical target the software models:
- 1.5–2 acres minimum (the rendered/canonical lot is **452.13 × 313.98 ft ≈ 3.3 acres**)
- **10 DCFC + 20–30 L2 chargers**
- **~100+ vehicles** served
- **A hard constraint that is also a differentiator: one dedicated parking/staging spot per vehicle
  on-site**, generally around the perimeter. In a safety recall or fleet-wide event, every vehicle
  must be able to return and park safely without relying on third-party parking. **OTTO-Q must
  never admit or keep more vehicles than the depot can park *and* service.**

## 2. The software thesis

Real estate and chargers are commodity. The defensible IP is the **orchestration layer**: the
software that decides, continuously, which vehicle goes where, when, in what order, and why — and
can prove the reasoning to an OEM auditor.

That layer is **OTTO-Q**.

> **The foundational message (Chase's words, 2026-07-20):** *"Our foundational message could be
> overall orchestration and reservation/servicing. Truly solving autonomous infrastructure so that
> AVs are seamlessly managed around our depots without issues. Ultimately, we don't have to hyper
> focus on just one layer of the orchestration when it's the whole ecosystem combined."*

Read that carefully, because it determines how you build:

- OTTO-Q is the **operating system for autonomous depot infrastructure**. The headline is
  **seamless integrated management**, not one optimised number.
- **Do not over-index on a single metric.** The charger-ratio throughput number, the energy-shave
  dollars, the cuOpt-vs-greedy delta — each is one facet, and any single facet under adversarial
  audit can look modest. The value compounds across the combined ecosystem: the reservation
  handshake **and** the servicing sequence **and** the energy orchestration **and** the
  deterministic safety shield, all interlocking.
- **"Without issues" is the bar.** Seamless means: no gridlock, no stranded cars, no unserviced
  vehicle deployed, no unsafe deploy, no human firefighting. Every gap that makes an OEM engineer
  see *an issue* undercuts the whole-ecosystem story more than any single number helps it.

## 3. Who the customer is

Two buyers, one product:

1. **AV fleet operators / OEMs** — Waymo, Tesla Robotaxi, Zoox, Motional and similar. They send
   vehicles to the depot and need a contract-grade guarantee that vehicles come back charged,
   clean, serviced, and safe, on time, with an audit trail. In the twin's seeded data these appear
   as real fleet operators: *Waymo Nashville* (~88 vehicles), *Tesla Robotaxi TN* (~65), *Zoox
   Southeast* (~63), *Local EV Fleet Co*.
2. **Investors** — the near-term commercial goal is a pilot raise. The demo has to survive a senior
   engineer from Alphabet, Tesla, or Amazon reading it adversarially.

**The audit bar is the design constraint.** Chase repeatedly asks for code that is *audit-ready for
senior engineers at Alphabet, Tesla, and Amazon*. That is why the system records every decision,
every rule evaluation, and every state transition, and why the safety layer is deterministic rather
than learned.

## 4. What is actually provable (read before writing any marketing-shaped claim)

A 2026-06-21 deep-research pass (104 agents, 20 claims verified / 5 refuted) reordered the pitch.
Full doc: `~/Desktop/OTTO-Q V1/OTTOQ_AV_DEPOT_RESEARCH.md`.

**Strongest provable edge: energy economics.** Demand charges are **30–74% of a commercial/DCFC
bill** (~$10/kW on the 15-minute peak) — the single strongest quantified prize. Peak-shaving via
forecast + BESS + charge scheduling is where the money demonstrably is.

**Second: charger-reliability-aware dispatch.** Public DCFC has been field-tested at ~72.5%
functional. Routing around faulted hardware is real value.

**Third: guaranteed-SoC-by-deployment / zero unsafe deploys.** The safety record.

**Then:** utilisation and packing (a 45–82× $/kWh swing between low- and high-utilisation sites).

**Verified savings are modest (3–13%). Do NOT headline large throughput or turnaround multipliers.**

**The density pitch is a design target, not a citation.** "100+ vehicles per depot vs 50–80" has no
external evidence. Frame it as *demonstrated in our calibrated twin*, never as an industry stat —
it will not survive diligence. The honest anchor: Waymo's 201 Toland site runs roughly 38 DCFC
ports (~60 kW dual-port) for 100+ vehicles ≈ **one charger per three vehicles**. Chargers are the
scarce binding constraint — and that scarcity is precisely *why* orchestration matters.

**Do not cite (refuted):** the Paren Reliability Index trajectory; the "80% fast-charge knee";
"Waymo charges at 30 kW". Arrival curves, dwell, cleaning cadence, and returns-per-day are not
publicly evidenced — they are calibrated from our own ingested corpora (see
`memory/reference_calibration_corpora.md`).

## 5. The real depot operating model (what the software must reflect)

From Chase, 2026-06-21. The twin and OTTO-Q must match **this**, not a generic queue.

**Daytime = relentless 24/7 robotaxi demand.** Vehicles cycle fast: arrive (arrivals are largely
predictable because OEM telemetry gives advance notice), get charged/serviced/cleaned as scheduled,
and **redeploy as soon as possible** — they should be out earning, not parked. Demand is effectively
unlimited during the day.

**Overnight = demand near zero** until the early-morning commute. Vehicles arrive in **waves (~5–10
every few minutes)**, go through full service/charge/clean, and are then **held in a dedicated
parking spot per vehicle** until the morning commute deployment.

The deploy target is therefore a real **operating curve** by hour (`ottoq_deploy_target_fraction`):
roughly 0.10 overnight rising to 0.90–1.0 in daytime. A static "hold 45% as a buffer" policy was
wrong and has been removed — a readiness buffer is an **overnight pre-positioning** concept, never
a daytime one.

**The four demo proof dimensions Chase wants, all of them:**
1. Most vehicles safely turned around (throughput **and** zero unsafe deploys)
2. Fastest turnaround time per vehicle
3. Highest fleet availability (overnight pre-positioned and ready for the commute)
4. Lowest energy cost (charge into cheap/clean hours, peak-shave, solar + BESS)

**Why the first benchmark misled, and what an honest test looks like:** a one-shot wave over a long
window is too easy — every policy clears it. The honest test is **sustained relentless demand** (the
"dinner rush"), where manual ops drown, reckless FIFO makes unsafe calls, and OTTO-Q safely turns
around the most.

## 6. Robotics and staffing direction

Staffing is modelled as a **fixed slider**, and the service-resource model is deliberately
**resource-agnostic** — a "service resource" can be a human technician or a robot. This matters
architecturally: do not hard-code human assumptions (shift breaks, one-task-at-a-time humans) into
the service model in a way that blocks swapping in automation later. Chase's thesis is that depot
service capacity eventually becomes robotic, and the software should already be indifferent.

## 7. Where the business documents live

Non-code business material is on the Desktop, outside any repo, and is **not** in this repository:

- `~/Desktop/OTTOYARD Company Brief - August 2026.pdf` — current company brief
- `~/Desktop/OTTOYARD/OTTOYARD Pitch Deck .pdf`, `OTTOYARD · Seed Deck · v1.pdf`
- `~/Desktop/OTTOYARD/OTTOYARD Fleet Depot Economics 12.7.25.pdf` — cost + revenue models
- `~/Desktop/OTTOYARD/OTTOYARD X Waymo .pdf`, `OTTOYARD Partnership Memo 11.17.25.pdf`
- `~/Desktop/OTTOYARD/Depot Pilot - Site Features & Specs.pdf`
- `~/Desktop/OTTOYARD/OTTOYARD-Brand-Guide.html` — the brand guide the design tokens come from
- `~/Desktop/OTTOYARD DATA ROOM/` — investor data room

If you are running in a cloud environment you will not have these. **Ask Chase for any specific one
you need; do not infer business numbers from the code.**

## 8. The deployment bar

"Production" here means **the most realistic hardware-integration path for an OEM** — the system
should be architected so that a real depot's chargers, a real fleet API, and a real BESS plug into
exactly the seams the twin currently occupies. Security hardening (full RLS, real tenancy, auth) is
consciously **deferred until post-funding**, with the exception of holes already found and closed.
That is a founder decision, made knowingly. Do not spend your autonomy budget hardening security
unless asked — but also never *weaken* what is already there, and always flag anything new you
introduce that would need hardening later.
