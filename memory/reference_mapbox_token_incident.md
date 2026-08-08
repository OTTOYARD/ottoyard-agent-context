---
name: reference_mapbox_token_incident
description: "Mapbox \"fleet command\" token racked up $1-2k raster charges and was DELETED (2026-07-07) — map features are broken until replaced; use MapLibre/free tiles going forward"
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

**Mapbox billing incident (Chase, 2026-07-07):** his Mapbox "fleet command" token accumulated **~$1,000–2,000 of Raster-tile usage "out of nowhere."** Chase **DELETED the token** (from Mapbox and, he believes, from the GitHub repo where it was committed).

**Facts established:** Claude never used Mapbox in the twin/motion work (verified: zero mapbox references in any local Desktop repo incl. depot-sim — it's pure Three.js/SVG; twin is Supabase). Prime suspects: one of the **Lovable-hosted cockpits (OTTO-PULSE / OrchestraAV / field-ops — repos not on this machine)** with a fleet-map view. Likely causes, in order: (1) **token committed to a GitHub repo** → scraped + abused for tile serving (classic pattern for sudden raster spikes); (2) a map component **re-mounting/re-fetching raster tiles on every data poll** (cockpits poll ~1.5s → thousands of tile loads/hour if the map remounts).

**Consequences + standing guidance:**
- Any cockpit map view is **broken until replaced** — expect a blank/erroring map in Pulse/Orchestra fleet views.
- **Replacement policy (Chase: "replace with a different tool most likely"): use MapLibre GL JS + free/self-hostable tiles (OpenFreeMap, Protomaps, or OSM raster with caching)** — no per-tile metering, no token to leak. This ALSO covers the future **city-map arrival feature (Nashville map with vehicles driving around)** — build that on MapLibre/Protomaps, NEVER metered raster tiles.
- If Mapbox is ever used again: **URL-restricted token + hard usage cap + stored as an env secret, never committed.**
- **Action item when next touching Pulse/Orchestra:** find the map component, check for a remount-per-poll tile-fetch loop (fix regardless of provider), swap to MapLibre, and confirm the old token is fully out of git history (a deleted file still lives in history — consider the repo compromised for that token; token deletion at Mapbox is the real kill).
