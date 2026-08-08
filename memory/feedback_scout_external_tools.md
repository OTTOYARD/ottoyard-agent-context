---
name: scout-external-tools
description: Chase wants Claude to proactively scout external tools / GitHub repos / APIs / simulators at implementation time — not rely solely on its own building strength.
metadata: 
  node_type: memory
  type: feedback
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

When implementing a non-trivial capability (especially on the simulator/twin side — vehicle movement, variability, physics, visuals, energy/grid feeds), proactively SEARCH for existing external tools, GitHub repos, simulators, or APIs that could provide or strengthen it, and surface/propose them before or while building.

**Why:** Claude's own building is valued and is "a huge factor," but it shouldn't be the ONLY path — the frontier build benefits from best-in-class external pieces. Chase explicitly said he's always open to integrating third-party APIs/tools (Unreal Engine, the NVIDIA stack, open-source repos, real-data APIs) and doesn't want me to rely solely on internal strength.
**How to apply:** at implementation kickoff for a meaningful feature, do a quick scan of what's available (web / GitHub search) and present options alongside the build-it-myself path. Relates to [[ask-clarify-early]], [[sim-visual-and-vehicle-flow]], [[otto-q-otto-twin-status]].
