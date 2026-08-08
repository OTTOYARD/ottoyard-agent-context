---
name: sim-visual-and-vehicle-flow
description: UE depot = future visual layer of the simulator; realistic intra-depot vehicle movement/flow + congestion-awareness is a KEY OTTO-Q demo. Claude = point person.
metadata: 
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

Two linked future-phase items (keep in back of mind; build at the twin/visual phase, AFTER the brain unification + UI work):

**1) VISUAL LAYER = the Unreal Engine depot Chase built** (Claude helped build it). Eventually pull it over as the VISUAL aspect of the simulator/twin. It needs more hyper-realism (the pass we had started). This is the photoreal stage the twin renders on.

**2) VEHICLE MOVEMENT & FLOW** — a HUGE key + investor/OEM proof. Claude is the MAIN point person for this; design it collaboratively with Chase. Goal: show OTTO-Q is spatially + temporally COGNIZANT of every vehicle so it routes movement WITHOUT congestion:
- Realistic movement + spacing: arrival INTO the depot → to the OTTO-Q-decided stall → moving between stalls around the depot, believable paths + spacing.
- Never funnel two vehicles into the same station/queue at the same time (no bottleneck/congestion).
- Model the TIME a move takes: when a vehicle begins leaving a station, where it's headed, and where every other vehicle is stationed/moving around it.
- Do NOT queue many vehicles to move in the same direction simultaneously — they'd back up / cause inter-depot congestion. Movement must be SEQUENCED + SPACED (a movement/path scheduler, congestion-aware).

This is the visible proof that OTTO-Q orchestrates real spatial reality, not just a schedule — ties directly to the scorekeeper story. When we reach it, scout external movement/traffic/pathfinding tools too (see [[scout-external-tools]]). Relates to [[otto-q-otto-twin-status]].
