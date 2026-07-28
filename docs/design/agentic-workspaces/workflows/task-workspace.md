---
title: "Task Workspace Workflow"
status: draft
---

# Task Workspace Workflow

The target workflow, annotated with what `kra` does today. **Bold** marks a step that does not exist
yet; see [gap analysis](../kra/gap-analysis.md) and [roadmap](../kra/roadmap.md).

## Create

```sh
kra ws create TASK-123 --format json
kra ws add-repo --id TASK-123 --repo owner/backend --repo owner/frontend --yes --format json
```

Conceptually:

1. Resolve task/workspace definition — `ws create`, or `ws import jira` / `ws import github-*`.
2. Determine required repositories — `--repo` flags, or `workspace.repo_presets.<name>`.
3. Fetch/update shared bare repositories — done, concurrently, from
   `$KRA_HOME/repo-pool/`; **not yet bounded**.
4. Attach worktrees at deterministic paths — `workspaces/<id>/repos/<alias>/`, branch from
   `workspace.branch.template`.
5. **Run blocking post-attach bootstrap per repository.**
6. **Run workspace-level bootstrap.**
7. Create/reuse the terminal runtime workspace — `ws open` (cmux today).
8. **Start optional background services.**
9. **Mark workspace ready.**

Steps 5, 6, 8, and 9 are the whole of [roadmap phase 1](../kra/roadmap.md#phase-1--make-the-substrate-extensible).
Without them, "created" and "usable" are the same state by assumption rather than by check.

## Layout

```text
KRA_ROOT/
└── workspaces/
    └── TASK-123/
        ├── workspace.md          # task source of truth
        ├── notes/
        ├── artifacts/
        ├── .kra.meta.json        # canonical workspace state
        ├── .cmux/dock.json       # runtime scaffold (template-managed)
        └── repos/
            ├── backend/
            └── frontend/
```

Operational state lives at root level, not per workspace:

```text
KRA_ROOT/.kra/state/
├── cmux-workspaces.json          # runtime mapping (non-canonical)
├── cmux-sessions.json
├── root-repos.json
└── operations/ws-close/<id>.json # lifecycle journal
```

The original design notes sketched a per-workspace `.state/`
directory. `kra` instead keeps canonical state in `.kra.meta.json` beside the work and operational
state at the root — a better split, because the archived workspace stays self-describing while
transient operational files never travel with it.

## Enter

```sh
kra ws open --id TASK-123 [--command "claude"]
```

Resolves the workspace path, creates or reuses the runtime workspace and notifies rather than
stealing focus, and falls back to directory-open (with parent-shell `cd` via
`kra shell init`) when the runtime is unavailable.

`--command` sends a startup command to the workspace terminal. Useful, but it is not bootstrap: it
requires a terminal, is not per-repository, and its failure is invisible to `kra`.

## Status

```sh
kra ws dashboard --format json --workspace TASK-123    # today
kra ws status --format json --id TASK-123              # wanted
```

Should expose:

| Field | Today |
|---|---|
| workspace lifecycle state | yes (`.kra.meta.json`, dashboard JSON) |
| work state (`todo`/`in-progress`) | yes |
| repositories | yes (count and risk tree) |
| branches | **no** in JSON |
| dirty state | aggregated risk yes; per-repo detail **no** in JSON |
| ahead/behind | computed internally, **not exposed** in JSON |
| output coverage | yes (`empty`/`notes-only`/`artifacts-only`/`documented`) |
| bootstrap state | **does not exist** |
| runtime state | **not exposed** |
| background processes | **does not exist** |

Git-derived facts are recomputed from Git on every read, which is correct — `kra` does not cache
risk into metadata. Non-Git facts (bootstrap completion, runtime mapping, ports, PIDs) should be
recorded rather than re-derived.

## Close

```sh
kra ws close --dry-run --format json --id TASK-123   # preflight
kra ws close --id TASK-123
```

Archive is not delete. `ws close`:

1. classifies live repo risk and prompts if anything is not clean,
2. checks output coverage against `workspace.close.empty_record_policy`,
3. opens a lifecycle journal so an interrupted close is resumable,
4. commits a `close-pre: <id>` snapshot with a staging allowlist,
5. removes worktrees, moves the workspace to `archive/<id>/`,
6. closes mapped runtime workspaces best-effort.

**Missing:** stopping tracked background processes, because none are tracked.

## Reopen

```sh
kra ws reopen --id TASK-123 --format json
```

Moves the workspace back from `archive/`, refetches bare repos, recreates branches (tracking
`origin/<branch>` when present, otherwise branching from the recorded `base_ref`), and re-adds
worktrees from `repos_restore[]`.

**Missing:** rerunning only the required setup — there is no setup to rerun and no record of what
had completed.

## Purge

```sh
kra ws purge --id TASK-123
```

Always prompts; refuses non-interactive execution without an explicit force flag; re-checks risk on
active workspaces; honors `protection.purge_guard` in `.kra.meta.json`; commits a `purge-pre: <id>`
snapshot; then removes worktrees, deletes the branches from the pool, and deletes the workspace and
archive directories.

**Missing:** trash before delete. Once the confirmations are past, there is no undo window
([roadmap 3.2](../kra/roadmap.md#32-workspace-trash-and-gc-upstreamable)).

## Health

```sh
kra doctor --format json
kra doctor --fix --plan     # then --apply
```

Detects broken workspace↔worktree links, stale locks, registry drift, missing baselines, legacy
state files, and unfinished close journals — classifying each as resume-ready, reset-ready, or
manual. This is the recovery path the design notes wanted for interrupted operations, and it already
exists.
