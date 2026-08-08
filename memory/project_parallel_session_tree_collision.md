---
name: project_parallel_session_tree_collision
description: "🚨2026-08-06 MY ERROR: I told Chase parallel chats were safe because 'only START is exclusive.' WRONG — two sessions in the same repo folder share ONE working tree, and my workflow checked out a branch under another session's uncommitted work. Rule: one folder per session, use git worktree."
metadata:
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-06T04:18:59.103Z
---

# 🚨 THE PARALLEL-SESSION RULE I GOT WRONG

## WHAT I SAID (wrong)
> *"Only START is exclusive. Reading is always safe. Starting a run wipes the previous one, but
> reading the database costs nothing and collides with nothing."*

True for the **database**. **Completely missed the filesystem.** Both sessions worked in
`~/Desktop/OTTOYARD/ottoyarddepot-sim`, which is **one working tree**. My layout workflow ran
`git checkout` in the same directory a motion session was actively editing.

## WHAT HAPPENED
The motion session reported: *"the repo switched branches under me mid-task — from `main` to
`layout-unify-importer` — while the other session was editing `scripts/buildLayoutSeed.mjs` and
`unreal/layoutSeed.*`. My changes are uncommitted in that shared tree."*
**~4,000 lines of measured, tested work sat uncommitted on someone else's branch.** It survived by
luck. One more `git checkout`/`git stash` and it was gone with no error. Recovered by backing the
files out to scratchpad, then committing to `motion-lane-endpoint-fix` off main and pushing.

## ⭐ THE CORRECTED RULE
**Two sessions must never share a working tree.** The DB and the GitHub repo can be shared; the
**folder on disk cannot**.
```bash
git -C ~/Desktop/OTTOYARD/<repo> worktree add ~/Desktop/OTTOYARD/<repo>-<task> main
```
Same repo, same remote, separate files, zero collisions. Point each parallel chat at its own path.

## THE THREE EXCLUSIVE RESOURCES (complete list — the first one was missing before)
1. **The working tree / folder on disk** ← the one I missed
2. **The running simulation** — one at a time; START purges the prior run and its evidence
3. **Migrations** — never two applying to the same database at once

**Read-only DB queries remain genuinely safe and unlimited.** That part was right.

## SUB-RULES EARNED
- Never `git checkout` a branch that would discard another session's uncommitted work. Commit or
  stash it to **its own ref** first, and back the files out before touching git at all.
- A workflow prompt that touches a shared repo must name the branches it may NOT disturb.
- **Verify a stored "established fact" before handing it to another session as a premise** — I passed
  the stale off-map claim to the motion session as settled and it was already fixed. See
  [[project_twin_render_offmap_bug]].

Links: [[reference_parallel_sessions_and_graphs]], [[project_twin_render_offmap_bug]],
[[project_depot_layout_unification]].
