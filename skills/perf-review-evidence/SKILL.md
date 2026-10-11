---
name: perf-review-evidence
description: Gather GitHub evidence for a manager's performance review of one engineer over a time period, and write it up as a single Markdown report - PR activity metrics, key docs and their outcomes, operational contributions, features delivered with customer impact, and technical initiatives. Produces evidence only; never writes the review itself. Use when the user wants to prep for a perf review, collect contribution data for a report, or asks for "perf review evidence" or "review artifacts" for someone.
disable-model-invocation: true
---

# Perf Review Evidence

Collect and organise evidence about one engineer's work so the manager can write their
performance review. **This skill produces evidence, not judgement.** It never rates the
engineer, never writes review prose ("strong performer", "needs to improve"), and never
recommends a rating. It reports what the repos show, with links.

## Inputs

Gather all of these before starting. Ask for anything missing.

1. **Repo catalog file**: a free-form Markdown file the user writes, listing each local
   repo (a path the user has already cloned/pulled) and what it is and what kind of
   information it holds (e.g. "design docs", "backend service", "runbooks", "CI
   tooling"). Read it as prose. If an entry is ambiguous or a path doesn't exist or
   isn't a git repo, list it back to the user and ask; don't guess.
2. **Employee**: their name, and optionally their GitHub username.
3. **Time period**: a start and end date. Resolve relative phrases ("last half",
   "FY26 H1") to explicit dates and confirm them with the user.
4. **Hints file** (optional): a free-form Markdown file with the user's own notes on
   outcomes and customers (e.g. "the retry design doc led to the v2 rollout; Acme and
   Globex adopted it"). Treat it as a trusted source, and cite it as "hints file" in
   the report so the reader can tell it from repo evidence.

## Process

### 1. Resolve identity

Build the set of identifiers that count as "the employee": GitHub username, commit
author names and emails, and co-authored-by trailers.

- If a username was given, use it, then look up its emails via `git log --author` and
  `gh api users/<username>`.
- If not, search by name: `gh search users "<name>"` and `git log --format='%an <%ae>'`
  across the catalogued repos, filtered by name.
- **Always show the user the resolved username and every name/email alias, and wait for
  confirmation** before continuing. A wrong match produces a wrong report.

### 2. Validate the repos

For each catalogued repo, confirm the path is a git repo, find its GitHub `owner/name`
from the remote, and confirm `gh` can reach it. Fail here, not inside the sub-agents.
Report any repo that can't be used and ask whether to continue without it.

### 3. Spawn one sub-agent per repo, in parallel

Each sub-agent mines a single repo and returns structured findings, so raw git and PR
output stays out of the main context. Give each sub-agent:

- The repo path, its `owner/name`, and its catalog description.
- The confirmed identity set and the time period.
- The classification rules below.
- The hints file contents, if any.
- The brief to return (see the schema below), and to **cite a link for every claim**
  and to write "not found in repo" instead of guessing.

Each sub-agent gathers:

- **PRs authored**: counts of opened, merged, closed unmerged, and still open in the
  period (`gh pr list --author`, `gh search prs`), with a per-month breakdown by
  created date.
- **PRs reviewed**: count of PRs reviewed or commented on (excluding their own), the
  number of distinct authors helped, approvals versus change requests, and a per-month
  breakdown.
- **Areas touched**: the top directories or components by changed files in their merged
  PRs and commits.
- **Docs authored**: Markdown and other doc files they added or substantially changed
  (`git log --diff-filter=A` / `--author` on doc paths, plus design docs, ADRs, RFCs and
  runbooks), with the PR or commit each came in with.
- **Candidate work items** for features, operational contributions and initiatives:
  merged PRs worth classifying, each with title, link, linked issues and a one-line
  summary. Group related PRs into one item, because the report lists work items, not
  individual PRs.
- **Outcome and customer signals**: for each doc and feature, look for evidence of
  what happened next. Use the PR description and linked issues, `gh issue view`,
  discussions, release notes, changelogs, and doc status fields. For customers, look
  for customer names or reported-by references in linked issues and release notes.

### 4. Classify work items

Classification uses PR evidence. Where a PR fits more than one category, put it in the
best fit and flag the item as ambiguous. The hints file can override any classification.

- **Feature delivered**: a user- or customer-facing change: `feat:` commits or labels,
  a linked product issue, new endpoints, UI or CLI behaviour.
- **Operational contribution**: on-call and incident fixes, bug fixes, CI and release
  work, dependency and security work, **test automation, canaries, and writing runbooks**.
- **Technical initiative**: self-started or cross-cutting work with no assigned ticket,
  such as refactors, tooling, migrations, design docs and platform improvements.

### 5. Merge and write the report

Merge the sub-agent findings. Sum the PR metrics across repos and keep the per-repo
rows. Deduplicate work items that appear in more than one repo. Write the report to
`./perf-review/<employee-slug>-<start>_<end>.md` in the current directory, creating the
folder if needed. Overwrite a previous report with the same name.

Then tell the user the path, and list what you couldn't find (see "Gaps" below).

## Report format

A single Markdown file with these sections, in this order.

### Header

Employee, resolved GitHub username and aliases, time period, repos included (with the
catalog description of each), and the date generated.

### 1. PR activity

Activity metrics, not a PR list.

- **Totals**: PRs authored (opened / merged / closed unmerged / open) and PRs reviewed
  (count, distinct authors helped, approvals versus change requests).
- **Per-repo table**: the same numbers for each repo, with the repo's role from the
  catalog.
- **Monthly trend**: a table of authored and reviewed counts per month, so consistency
  of activity is visible.
- **Areas of contribution**: the repos and top directories or components they worked
  in, tied to the repo roles from the catalog.

Do not include per-PR lists, time-to-merge, or code-size metrics.

### 2. Key docs authored

| Doc | Repo / path | Date | Role | What it proposed | Outcome achieved | Evidence |
|---|---|---|---|---|---|---|

- **Role**: sole author or co-author.
- **Outcome achieved**: what shipped, was adopted or was decided as a result. "Not
  found" when there's no evidence.
- **Evidence**: links to the PRs, issues or commits that back the outcome. Mark
  outcomes that come from the hints file.

### 3. Key operational contributions

Grouped work items, each with a one-line description, the category detail (incident,
canary, runbook, test automation, CI, and so on), the date or range, and links.

### 4. Key features delivered

For each feature: what it is, **how it helps customers**, the employee's role, links,
and **customers who used it**. Customers come only from linked issues, release notes
and the hints file, and each name is cited. If none are found, write "No customer
information found in repos or hints."

### 5. Technical initiatives

Self-started or cross-cutting work: what the initiative was, why it was taken on (if
the repo evidence says), what it changed or enabled, and links.

### Gaps and ambiguities

Anything the reader should check: repos that couldn't be read, items classified as
ambiguous, outcomes and customers not found, and catalog or hints entries that couldn't
be interpreted.

## Rules

- **Every claim needs a link** (PR, issue, commit, doc, release) or a "hints file"
  citation. If you can't cite it, don't write it.
- **Never invent outcomes or customers.** Missing evidence is reported as missing.
- **Evidence only.** No ratings, adjectives about performance, or review text.
- Counts must be reproducible from the stated period and identity set. State the date
  field used (created or merged) the first time a count appears.
- Keep customer names exactly as the source gives them. Don't normalise or infer.
