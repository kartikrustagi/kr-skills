---
name: sync-repos
description: Recursively find every git repository reachable from the current working directory (including the repo that contains the cwd itself) and bring each one onto its main branch, fast-forwarded to match its remote. Use this whenever the user asks to "sync repos", "update repos", "pull latest", "sync my repos", "refresh all repos", "get latest from remote", or wants to make sure every repo under here is up to date before starting work. If a repo is not on its main branch, is dirty, or has diverged from its remote, this skill flags it and asks the user how to proceed instead of forcing a resolution.
---

# Sync Repos

Bring every git repository reachable from the current working directory onto its main
branch and fast-forward it to match the remote. This is meant to be run at the start of
a work session so all repos under here start from a clean, up-to-date baseline.

## 1. Find every reachable repo

"Reachable from here" means two things, both in scope:

- **The repo containing the cwd itself**, if any — resolve it with
  `git -C . rev-parse --show-toplevel`. If cwd isn't inside a repo, there's nothing to
  add from this part.
- **Every repo at or below cwd**, found by walking the directory tree.

Walk the tree depth-first from cwd:

- At each directory, first check whether it's a git repo root (`.git` present, or
  `git -C "$dir" rev-parse --show-toplevel` resolves to `$dir` itself). If it is, record
  it and **stop descending into it** — don't walk into a repo's internals looking for
  more repos nested inside (submodules, vendored checkouts, etc. are intentionally left
  alone).
- Otherwise, follow symlinked directories as well as real ones (a repo reachable only
  through a symlink should still count, same as a real directory). Guard against cycles:
  track the resolved real path (`readlink -f`) of every directory you enter, and never
  enter one you've already resolved — this catches circular symlinks without relying on
  depth limits.
- Skip descending into common noise directories without checking them for repos first:
  `node_modules`, `vendor`, build/dist output directories, and hidden dot-directories in
  general (other than resolving a repo's own `.git`). These are never where a real
  working repo lives and would otherwise slow the walk down or surface stray nested
  `.git` dirs from installed dependencies.

Collect every resolved repo root and **deduplicate** — the upward lookup and the
downward walk can both resolve to the same repo (e.g. cwd is the repo root itself), and
two different symlinks can resolve into the same repo too. Sync each distinct repo once.

If nothing resolves (no enclosing repo and nothing found below cwd), tell the user
there are no git repos reachable from here and stop.

## 2. For each repo, in order

Run each step from inside the repo (`git -C <repo>` or `cd`). Do this one repo at a
time and keep track of results for the final summary — don't stop the whole run just
because one repo needs attention; flag it and continue to the next repo.

1. **Fetch first**, so branch comparisons and the main-branch check reflect the
   remote's current state: `git fetch --prune`. If fetch fails (no network, no
   remote, auth error), record the repo as failed with the error and move on.

2. **Determine the repo's main branch.** Don't hardcode `main` — repos may default to
   `master` or something else. Prefer, in order:
   - `git symbolic-ref refs/remotes/origin/HEAD` (strip the `origin/` prefix)
   - if that's unset, `git remote show origin | grep 'HEAD branch'`
   - if there's no remote at all, fall back to checking whether `main` or `master`
     exists locally
   Treat whatever this resolves to as "the main branch" for this repo.

3. **Check the current branch**: `git branch --show-current` (empty output means
   detached HEAD — treat that like "not on main").

4. **If the current branch is NOT the main branch** (or HEAD is detached): do **not**
   check out main or pull automatically. Record this repo as needing input, including
   its current branch/HEAD state and the detected main branch name. Leave the repo
   untouched and move on to the next one.

5. **If already on the main branch**: check for uncommitted changes to *tracked*
   files — `git status --porcelain --untracked-files=no`. Untracked files don't block
   a pull (git only refuses if the incoming changes would collide with one, which is
   rare and surfaces as its own clear error during the pull step below), so ignore
   them here; don't count a repo as dirty just because it has untracked files.

   Watch out for a false-positive specific to symlinked tracked files (e.g. CRDs
   symlinked into a Helm chart dir) in sandboxes where the repo lives on a
   virtiofs/`sbx mount`-ed directory: git's status fast-path can misreport an
   unchanged symlink as modified due to stat/mtime caching quirks over that mount,
   and the same file can flip between "modified" and "clean" across consecutive
   `git status` calls with no writes in between. Before flagging a repo dirty solely
   because of symlink entries, verify each one is a real change: compare
   `git cat-file -p HEAD:<path>` against `readlink <path>` byte-for-byte. If they
   match, treat that entry as clean (it's a caching artifact, not a real edit) —
   only count genuinely differing content as dirty.

   If there are real tracked modifications/staged changes after that check, record
   the repo as needing input (they could conflict with a pull) rather than guessing
   whether to stash — move on.

6. **If on main and clean**: pull with `git pull --ff-only`. This intentionally avoids
   creating merge commits — if it fails (e.g. history has diverged from the remote),
   record the repo as needing input with the error, rather than forcing a merge or
   rebase.

## 3. Resolve anything that needs input

After going through every repo, if one or more repos were flagged (wrong branch,
detached HEAD, dirty tree, diverged history, or fetch/pull failure), summarize them
clearly to the user — repo path, what's wrong, current state — and **ask what to do**
before taking any further action on those repos. Do not guess or force a resolution
(e.g. don't auto-checkout main, don't auto-stash, don't force-pull). Apply whatever
the user decides per repo.

## 4. Final summary

Report, per repo:
- ✅ already on main and pulled cleanly (note if it was already up to date vs. fetched
  new commits)
- ⚠️ needs input (with the specific reason)
- ❌ failed (fetch/network/auth error, with the error)
- 🔗 skipped — broken/looping symlink, or didn't resolve into a git repo

Keep the summary concise — one line per repo is enough unless the user asks for more
detail.
