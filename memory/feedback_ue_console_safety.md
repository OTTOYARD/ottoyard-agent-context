---
name: feedback-ue-console-safety
description: NEVER use cmd+a+Delete to clear the UE Python console — it deletes all level actors if focus slips
metadata: 
  node_type: memory
  type: feedback
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

When driving the Unreal Editor Python console via computer-use, **never use `cmd+a` then `Delete` to clear the console input field.** If keyboard focus is on the viewport/Outliner instead of the console (which happens constantly — taking a screenshot or any focus change pops the macOS "Claude" desktop app to frontmost), `cmd+a` = Select All Actors and `Delete` = delete them all. This emptied the Depot level (1039 → 0 actors) once; recovered only because the dusk script had already `save_current_level()`'d to disk and a single editor `cmd+z` ("Undo Delete Elements") restored it.

**Why:** The Mac focus repeatedly slips off UnrealEditor between batches, so you can't assume the console has keyboard focus when a batch starts.

**How to apply:**
- Before any UE console batch, call `open_application(app="UnrealEditor")` to force it frontmost.
- To clear/replace the console input, use `triple_click` on the input field (selects the single line, then type to replace) — NOT `cmd+a`+`Delete`. If focus is wrong, triple_click at worst selects one actor (harmless); cmd+a+Delete nukes the level.
- The console input is the bottom status-bar field (~x=590, y=872 at full-screen layout); there's a second one in the Output Log panel (~y=845).
- If actors ever vanish (Outliner "0 actors", title "Depot*"), DO NOT SAVE — click the viewport and `cmd+z` to undo; the on-disk Depot.umap holds the last `save_current_level()`.
- Recovery proof pattern: after undo, confirm Outliner actor count is back before doing anything else.

Related: [[project_ottoq]]. Pipeline order if build changes: rbuild → VERIFY counts → rd → rb (+ rdusk for night).
