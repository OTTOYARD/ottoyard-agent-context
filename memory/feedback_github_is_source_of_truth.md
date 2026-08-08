---
name: feedback_github_is_source_of_truth
description: "⭐STANDING (Chase 2026-08-06): GitHub is the ONLY source of truth. Chase's sole git action is clicking Merge on a PR. Desktop folders are Claude's workspace — never ask Chase to run git commands, sync folders, or manage local checkouts."
metadata:
  node_type: memory
  type: feedback
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-08-06T12:41:54.096Z
---

# ⭐ GITHUB IS THE SOURCE OF TRUTH — CHASE ONLY CLICKS MERGE

> *"It's a little frustrating having to manage desktop folders and GitHub folders. I'm much more
> comfortable with GitHub just being the source of truth. I don't mind there being file folders on
> the desktop that are irrelevant, but I cannot keep updating both folders or both sources — GitHub
> and the desktop — every time there's a new PR."*
> — Chase, 2026-08-06

**Why:** Chase is the founder, not the operator. Every git command handed to him is work offloaded
onto the person least equipped to do it and most expensive to interrupt. It also creates a second
place for state to drift, which is the same class of bug as the two-depot-layout problem —
[[project_depot_layout_unification]].

## HOW TO APPLY
- **GitHub is authoritative. Always.** Desktop clones are Claude's scratch workspace and are
  DISPOSABLE. If a local folder drifts, Claude re-syncs it silently before working — never reports it
  as a thing Chase must resolve.
- **Chase's ONLY git action is clicking Merge on a PR.** Give him the PR link and nothing else.
- **NEVER hand Chase a git command to run.** Not `checkout`, not `pull`, not `worktree`, not `clone`.
  If isolation is needed for a parallel session, Claude creates it (`git worktree add`) and simply
  tells Chase which folder that session should use — one sentence, no command.
- Before ANY work in a local repo: `git fetch` + confirm the branch is current with origin. Stale
  local state is Claude's bug, never Chase's task.
- Do not report local-vs-remote divergence as news unless it changes what Chase must decide.
- ⚠️ Corollary from [[project_parallel_session_tree_collision]]: one folder per session. Claude sets
  each one up. Chase is told the path, not the mechanism.

Links: [[feedback_plain_language]], [[project_parallel_session_tree_collision]],
[[feedback_rebuild_not_benchmark]], [[project_architecture_separation]].
