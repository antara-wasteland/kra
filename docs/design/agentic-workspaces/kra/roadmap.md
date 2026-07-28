---
title: "kra Roadmap"
status: draft
---

# kra Roadmap

Sequenced from the [gap analysis](gap-analysis.md). Ordering is by dependency and by how much each
item unblocks, not by implementation cost.

Each item notes whether it is **upstreamable** (generic enough to raise against `tasuku43/kra`) or
**fork-local** under the rules in [fork strategy](../decisions/fork-strategy.md).

## Phase 1 — make the substrate extensible

### 1.1 Bounded worker pool (upstreamable)

A shared internal primitive for fan-out: configurable concurrency limit, context cancellation,
per-item error aggregation, streamed progress. Retrofit `ws add-repo`, `repo pool add`, and
`ws import all` onto it.

Deliberately first. Hooks are heavier than fetches, and adding an unbounded hook runner then
bounding it later means writing the same code twice.

### 1.2 Lifecycle hooks (upstreamable — the headline gap)

Events, minimally:

```text
pre-create / post-create        (workspace level)
pre-attach / post-attach        (per repository, ws add-repo)
pre-close / post-close
pre-reopen / post-reopen
pre-purge / post-purge
```

Contract:

- hooks are repository-owned or root-owned scripts, not `kra` config that inlines shell,
- environment carries `KRA_TASK_ID`, `KRA_WORKSPACE_PATH`, `KRA_REPO_KEY`, `KRA_REPO_ALIAS`,
  `KRA_WORKTREE_PATH`, `KRA_BRANCH`, `KRA_BASE_REF`, `KRA_HOOK_EVENT`, `KRA_INDEX`,
- run through `internal/procexec/` so `docs/dev/spec/concepts/external-command-execution.md` keeps
  covering every external process,
- blocking by default; failure produces a recorded `bootstrap-failed` state, not a silent partial
  workspace,
- idempotent by contract — reruns must be safe, and there must be a way to rerun
  (`kra ws bootstrap --id <id>`),
- progress and failures surface in both human and JSON output.

Persist hook results the way `ws close` persists phases: reuse
`docs/dev/spec/concepts/lifecycle-journal.md` rather than inventing a second progress mechanism.

Write the spec under `docs/dev/spec/concepts/` before the code; this repo's convention is
spec-then-implementation, and hooks are a compatibility surface once released.

### 1.3 Hook trust (upstreamable)

First-execution approval, re-approval when the hook's content hash changes, per-root trust store
under `.kra/state/`, non-interactive mode refuses untrusted hooks rather than prompting or silently
running them. Different policy tier for user-local hooks vs repository-checked-in hooks.

Ship with 1.2 or immediately after — a hook runner without a trust model is a `git pull` that
executes code.

### 1.4 Background hooks and process tracking (upstreamable)

Long-running work (dev servers, watchers) started detached, with PID/port/handle recorded under
`.kra/state/`, reported by `ws status`, and stopped on `ws close`. Depends on 1.2 and on the state
model from 3.1.

## Phase 2 — close the machine-interface gaps

### 2.1 `ws status --format json` (upstreamable)

Per-workspace detail the way `ws dashboard --format json` gives per-root summary: lifecycle state,
work state, per-repo branch, ahead/behind counts, repo risk state, worktree path, bootstrap state,
runtime mapping, background processes. Same envelope, `action=ws.status`.

The requirement baseline's `task-status TASK-123 --json` is the one mandatory machine-interface item
still unmet.

### 2.2 `ws exec` (upstreamable)

`kra ws exec --id <id> [--repo <alias>...] -- <cmd>` across a workspace's repositories, on the
bounded pool from 1.1, with per-repo exit codes aggregated into the JSON envelope. The natural
companion to hooks and the thing polyrepo work needs daily.

### 2.3 Repo `focus` (upstreamable)

Optional `focus` path per entry in `repos_restore[]`, used for navigation, agent cwd, scoped hook
execution, and status presentation. Must not create extra worktrees.

## Phase 3 — safety and recovery

### 3.1 Explicit workspace readiness state (upstreamable)

`.kra.meta.json` currently carries lifecycle status (`active`/`archived`) and work state
(`todo`/`in-progress`). Hooks need a third axis — something like `created` / `bootstrapping` /
`ready` / `bootstrap-failed` — so "the directory exists" stops being confused with "the workspace is
usable". Requires a `schema_version` bump; see `docs/dev/spec/core/COMPATIBILITY.md`.

### 3.2 Workspace trash and GC (upstreamable)

`ws purge` moves disposable workspace state to a trash location and detaches Git state, with
permanent deletion deferred to a `ws gc` pass behind safety gates modelled on `repo gc`. Gives an
undo window and makes purge feel instant.

## Phase 4 — runtime neutrality

### 4.1 Rename the runtime port (upstreamable, larger)

Generalize `cmux.Client` to a runtime-neutral port with capability queries (notify, open workspace,
send command, docs pane, dock/status control), keep cmux as the reference adapter, and keep the
existing directory-open fallback as the no-op adapter. Touches ~27 files, the mapping store schema,
and the state filenames — mechanical but broad, and best raised upstream as one intentional change
rather than accumulated in a fork.

### 4.2 Herdr adapter (fork-local until upstream accepts 4.1)

Implement against the port from 4.1. Not before it — a second runtime wired in by name is exactly
the "scattered special cases" outcome the design notes warn about.

### 4.3 Custom subcommand extension point (upstreamable)

`kra <name>` resolving to `kra-<name>` on PATH, or root-config-declared commands. This is what lets
OpenSpec/AXIS-specific and workstation-specific behavior live outside the tool instead of inside
the fork.

## Explicitly out of scope

Unchanged from the [requirement baseline](../03-requirements.md): no LLM-generated commit messages,
no model API keys, no task decomposition, no autonomous merging, no PR or CI policy, no
specification ownership. Those belong above the workspace substrate.

`kra serve`, `ws insight`, and the GitHub import paths already present in this fork sit at the edge
of that line. They are fine as long as the substrate below them stays usable without them.
