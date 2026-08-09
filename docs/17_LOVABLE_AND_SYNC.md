# 17 — Lovable, GitHub sync, and stale clones

**Researched and answered 2026-08-08**, from the repos' own git history plus Lovable's official
documentation. This was an open question in the first draft of this package; it is now closed.

---

## 1. The short answer

**Yes — Lovable commits directly to `main`, and yes — it picks up your merges automatically.**

Lovable and GitHub are wired together **in both directions on exactly one branch**. For all three
OTTOYARD front-end repos, that branch is `main`.

- Somebody edits in the Lovable chat UI → **Lovable commits it straight to `main` on GitHub.**
- Somebody merges a PR into `main` on GitHub → **Lovable pulls that change back into the project.**

Lovable's own wording: *"Changes made in Lovable sync to GitHub"* and *"Changes pushed to the active
GitHub branch sync back into Lovable."* It also states plainly: **"Lovable only edits and syncs one
branch at a time."**

## 2. This is not theoretical — it has already happened here

Lovable's commits are authored by a bot. In these repos it appears as
**`gpt-engineer-app[bot]`** — Lovable's original name was GPT Engineer, and older projects keep the
legacy app identity. (Newer Lovable projects show `lovable-dev[bot]`.) The confirmation that this is
Lovable and not something else: `lovable-tagger` is a dependency in every one of these repos'
`package.json`, imported by `vite.config.ts`.

**Measured commit counts by author:**

| Repo | Lovable bot commits | Claude commits | Merge commits by Chase |
|---|---|---|---|
| `ottoyarddepot-sim` | 9 | 134 | 48 |
| `ottoyard-field-ops` | **66** | 12 | 7 |
| `ottoyard-OTTO-Q` | 4 | 1 | 1 |

**And it has pushed on top of Claude's work.** In `ottoyarddepot-sim`, the two most recent commits at
the time of writing were:

```
d265eff 2026-08-05 gpt-engineer-app[bot] — "Verified Supabase backend"   (a MERGE commit)
4dc9326 2026-08-05 gpt-engineer-app[bot] — "Work in progress"
```

Lovable branched from an older commit, made a change, and **merged it into `main` itself.** Both
touched only `bun.lock` (14 lines), so the practical damage was nil — **but the mechanism is live and
it will do it again.**

## 3. What this means for you, practically

**It does NOT mean your work can be silently overwritten.** Lovable documents a guard: if it cannot
rebase onto someone else's change, it *"pushed it to a new `lovable-sync-<timestamp>` branch
instead"* rather than clobbering. So the failure mode is a **surprise branch and a divergence**, not
lost commits.

**It DOES mean five things you must plan around:**

1. **`main` is not yours alone.** Always `git fetch` and rebase before pushing a branch that has been
   open for more than a few hours.
2. **A PR you open can go stale underneath you** if Lovable commits to `main` while it is open.
3. **Merging your PR changes what Chase sees in Lovable**, immediately and without him doing
   anything. If your change breaks the Lovable preview, he finds out by looking at a broken app.
4. **Lockfile churn is normal noise.** `bun.lock` / `package-lock.json` diffs authored by the bot are
   not a signal of anything.
5. **If you ever see a `lovable-sync-<timestamp>` branch appear, that is a divergence that needs
   resolving** — do not delete it, and tell Chase.

## 4. What to say to Chase, in one sentence

> *"When you change something in Lovable, it saves straight into GitHub on the main branch. And when
> you merge one of my pull requests, Lovable picks it up automatically. So we're both writing to the
> same place — it works, but it means I'll always pull the latest before I push, and if you're doing
> something in Lovable at the same time as I'm working, tell me so we don't collide."*

## 5. One thing that is still not confirmed

Lovable's public documentation says nothing about **deployment** — whether every commit republishes
the live site, or whether publishing requires an explicit action. The evidence in the repos hints at
explicit publishing (there is a bot commit literally titled *"Update site info for publish"*), but
that is inference, not documentation.

⚠️ There is also one historical symptom on record worth remembering: an error Chase reported
**reproduced only on the deployed Lovable site**, never locally.

**If you are about to do anything that could break the live site, ask Chase first.**

## 6. Can this be made safer?

Yes, and it is worth raising with Chase rather than doing unilaterally.

**Lovable can be pointed at a branch other than `main`** — the branch picker lives in the project's
GitHub settings, and Lovable's docs describe both switching branches and creating one.

**The safer arrangement would be:** Lovable works on its own branch (say `lovable`), agents work on
`main`, and the two are reconciled by deliberate PRs rather than by continuous two-way sync.

**The cost:** Chase loses the immediacy of "I changed it in Lovable, it is now in the repo." For a
non-programmer founder who uses Lovable as his direct hands-on surface, **that immediacy may be worth
more than the safety.** It is his call, not yours. Do not change this setting.

---

## 7. ⭐ THE WORKING-COPY RULE — clone fresh, always

**Decided 2026-08-08 by the founder, and it is now how this project works.**

> **GitHub is the only source of truth. There are no persistent local working copies.**

**How to work:**
```bash
git clone https://github.com/OTTOYARD/<repo>.git   # fresh, into your own temp dir
# ... build, test, verify ...
git push origin hermes/<lane>/<slug>                       # push the branch
# ... open the PR, then delete the clone ...
```

**Why this is the rule.** The founder's own words: *"I don't like code being in two places and
missing commits because of that."* He is right, and it had already cost real work:

- The Desktop clones drifted **up to 77 commits behind**. The first draft of this entire onboarding
  package was written from them and was wrong about migration state, branch state, and several
  "open" defects that were already fixed.
- Two agent sessions sharing one folder share **one working tree** and clobber each other. That has
  happened on this project.
- A stale clone **looks authoritative and isn't.** That is the worst property a file can have.

**What was actually done on 2026-08-08:** the persistent Desktop clones of `ottoyarddepot-sim`,
`ottoyard-field-ops`, `ottoyard-OTTO-Q`, `ottoq-intelligence`, and the non-git scratch copy
`depot-sim-motion` were **deleted** (~930 MB), after verifying that every commit in them was
preserved on GitHub. Six branches that existed **only** on that Desktop — including 198 lines of
evidence written that same day — were pushed to GitHub first as **`salvage/*`** branches.

**If you find a `salvage/*` branch**, that is what it is: work rescued from a local-only branch
before the folder was removed. Review it, merge or discard it, then delete the branch.

**Corollary:** never write anything you care about to a path that is not inside a git repo you are
going to push. And never leave a branch unpushed at the end of a session.

## 8. ⚠️ Why this rule exists: the clones were stale

Discovered while researching the above, and it is arguably more dangerous than the sync itself.

**Every local clone under `~/Desktop/OTTOYARD/` is behind its remote**, measured 2026-08-08:

| Repo | Commits behind `origin/main` |
|---|---|
| `otto-q-core` | **77** |
| `ottoyarddepot-sim` | 11 |
| `ottoyard-field-ops` | 10 |
| `ottoyard-OTTO-Q` | 5 |

**Consequence:** anything reasoned from a Desktop clone without fetching first is potentially months
out of date. **The first draft of this very package was written against those stale clones, and it
was wrong about migration state and branch state as a result** — see `docs/15_UNCERTAINTIES.md` §3c.

**The rule: `git fetch origin` and compare against `origin/main`, never against local `main`.**

```bash
git fetch origin --prune
git log --oneline main..origin/main        # what you are missing
git rev-list --count main..origin/main     # how far behind
```

**And to check whether a branch is actually merged** — do not trust a branch's continued existence,
and do not trust a memory file:

```bash
git rev-list --count origin/main..origin/<branch>   # 0 means fully merged
```

## 8. GitHub identity for commits

⚠️ **GitHub rejects pushes authored as `chase@ottoyard.com`** (email privacy is enabled on the
account). Commit as a noreply identity. Existing repos are already configured; check
`git config user.email` in any repo you commit from.

The commit identities you will see in history, and what they mean:

| Author | Who |
|---|---|
| `gpt-engineer-app[bot]` | **Lovable**, committing directly to `main` |
| `Claude (OTTO-Q)`, `Claude`, `Claude Opus 4.8` | Claude Code sessions |
| `OTTOYARD <230774767+OTTOYARD@users.noreply.github.com>` | Chase, almost always a PR merge |
| `hermes/...` branches | you |
