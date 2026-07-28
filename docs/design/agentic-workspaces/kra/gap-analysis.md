---
title: "kra Gap Analysis"
status: draft
---

# kra Gap Analysis

Source-level audit of this repository against the tool-neutral
[requirement baseline](../03-requirements.md).

The [exploration log](../exploration-log.md) ends by naming the next step: compare candidate
substrates *at source level* around hook extensibility, parallel execution, task state model,
archive/reopen, cleanup, machine-readable APIs, and runtime integration boundaries. This page is the
`kra` half of that comparison. It is deliberately evidence-first: every claim points at a file, a
spec, or the absence of both.

Audited at commit `1fdcd0b` (`fix(repo): honor gh clone protocol in discover`).

**This is an audit of upstream.** At the time of writing, `rishwanth-th/kra` `main` is identical to
`tasuku43/kra` `main` at `1fdcd0b` — no commits in either direction. Every finding below therefore
describes upstream `kra`, not fork-local behavior, and every gap is an upstream contribution
opportunity rather than a divergence to maintain. See
[fork strategy](../decisions/fork-strategy.md).

Two facts from upstream's own process documents frame the results:

- **The backlog is complete.** `docs/dev/backlog/` tracks 17 epic files, and every ticket in every
  file is checked. There is no stated pending direction to collide with.
- **Nothing hook-shaped was ever proposed.** Searching the whole backlog for hooks or bootstrap
  returns only shell-integration bootstrap (`UX-CORE-002`), config scaffolding (`CONFIG-003`,
  `CONFIG-006`), and template scaffolding (`TEMPLATE-WS-002`). Lifecycle hooks were not considered
  and rejected — they were never raised. That is the difference between a philosophical objection
  and an unfilled gap, and it is what makes the roadmap upstreamable.

`kra` is also strictly layered — `cli` → `app` → `domain`, with `infra`/`ui` adapters — and the
layering is enforced by tests rather than convention (`internal/archguard/layering_test.go` asserts
dependency direction and holds an explicit allowlist for direct infra imports under `cli`, so it
cannot expand silently). Contributions have to respect that structure; the specs under
`docs/dev/spec/` are written before implementation, per `AGENTS.md`.

## Scorecard

| # | Requirement | Status | Where |
|---|---|---|---|
| 1 | Post-create bootstrap hooks | **Missing** | no hook code or spec exists |
| 2 | Ergonomic CLI (create/open/list/status/exec/remove/cleanup) | **Mostly met** | `internal/cli/cli.go`; no `exec` |
| 3 | Safe cleanup with risk classification | **Met** | `internal/core/workspacerisk/workspacerisk.go` |
| 4 | Monorepo and polyrepo in one model | **Met** (no `focus`) | `internal/cli/ws_add_repo.go`, `.kra.meta.json` |
| 5 | Efficient storage (shared object store) | **Met** | `internal/paths/paths.go:13`, `internal/gitutil/gitutil.go:124` |
| 6 | Machine interface (JSON, exit codes, non-interactive) | **Mostly met** | `internal/cli/json_output.go`, `docs/dev/spec/concepts/output-contract.md` |
| 7 | Hook trust/approval | **Missing** (blocked on #1) | — |
| 8 | Runtime adapter abstraction | **Partial** | `internal/app/cmux/usecase.go:17` |
| 9 | Blocking and background hooks | **Missing** (blocked on #1) | — |
| 10 | Archive / reopen | **Met** | `internal/cli/ws_close.go`, `ws_reopen.go` |
| 11 | Trash before delete | **Missing** | `docs/dev/spec/commands/ws/purge.md` deletes directly |
| 12 | Doctor / repair | **Met** | `internal/cli/doctor.go`, `docs/dev/spec/commands/doctor.md` |
| 13 | Path / branch templates | **Partial** | branch templates only |
| 14 | Custom subcommands | **Missing** | fixed dispatch in `internal/cli/cli.go` |
| 15 | Shell navigation protocol | **Met** | `docs/dev/spec/commands/shell/init.md` |
| 16 | Bounded parallel operations | **Partial** | bounded pool exists (`internal/app/cmux/usecase.go:145`), not shared |

Headline: `kra` is strong exactly where the baseline calls for *state discipline* — task identity,
canonical metadata, risk classification, recoverable close, shared bare pool, machine output — and
absent exactly where the baseline calls for *extensibility*. There is no way for a repository to say
"after you attach my worktree, run this."

---

## 1. Post-create bootstrap — Missing

The baseline calls this mandatory, and the [exploration log](../exploration-log.md) calls it the
non-negotiable feature: a newly created worktree must be able to run `uv sync`,
`pnpm install --frozen-lockfile`, `git submodule update --init --recursive`, `direnv allow`, or
`./scripts/worktree-bootstrap` with `TASK_ID` / `WORKSPACE_PATH` / `REPO_NAME` / `WORKTREE_PATH` /
`BRANCH` / `INDEX` in the environment.

None of this exists. Searching the Go sources for `hook` matches only an unrelated shell test.
There is no hook spec under `docs/dev/spec/`, and no config key reserves the namespace
(`docs/dev/spec/concepts/config.md`).

The closest existing mechanisms are all inert with respect to setup:

- **Workspace templates** (`docs/dev/spec/concepts/workspace-template.md`) copy
  `templates/<name>/` into the new workspace, and the spec is explicit that "copy is static (no
  variable interpolation / placeholder expansion)". A template can ship a bootstrap *script*; nothing
  ever runs it.
- **`ws open --command "<cmd>"`** sends a startup command to the workspace terminal. That is
  runtime convenience, not a lifecycle invariant: it needs a terminal, it is not per-repository, its
  failure is invisible to `kra`, and it does not run at `ws add-repo` time — which is when a worktree
  actually appears.
- **`ws create` / `ws add-repo`** finish at "directory exists, worktree attached". Nothing in the
  attach path (`internal/cli/ws_add_repo.go`) executes repository-owned setup.

Consequence for agent use: every agent that lands in a fresh workspace must rediscover and rerun
project setup itself, and `kra` cannot distinguish "workspace created" from "workspace usable".
There is no `bootstrap-failed` state to report because there is no bootstrap.

This is the single change that most increases `kra`'s value as an agentic substrate, and it is a
clean fit for the existing architecture — `internal/procexec/` already centralizes external command
execution (`docs/dev/spec/concepts/external-command-execution.md`), and the lifecycle journal
(`docs/dev/spec/concepts/lifecycle-journal.md`) already demonstrates how to persist multi-phase
progress recoverably.

## 2. Ergonomic CLI — Mostly met

`create`, `open`, `list`, `status`, `close`, `reopen`, `purge`, `add-repo`, `remove-repo`, `lock`,
`doc`, `task`, `dashboard`, `log`, plus `init`, `context`, `root`, `repo`, `template`, `shell`,
`doctor`, `serve`, `agent` (`internal/cli/cli.go:65-160`). Interactive selection is available through
`--select` on the action commands, and every action command also takes `--id`, satisfying the
baseline's "interactive is useful, but everything must be scriptable".

Gap: no `exec`. The baseline wants one command to run something across a workspace's repositories —
the natural companion to bootstrap hooks and the thing polyrepo work needs constantly
(`kra ws exec --id T-1 -- git status -sb`). Today that is a shell loop over `repos/*`.

`kra agent prompt` (`internal/cli/agent.go`) is a genuinely good idea the baseline did not think to
ask for: a machine-readable description of the CLI's own safe/destructive/JSON-first command
classes, so an agent can be told how to drive `kra` without scraping help text.

## 3. Safe cleanup — Met

`internal/core/workspacerisk/workspacerisk.go` classifies each repo as `clean`, `dirty`, `unpushed`,
`diverged`, or `unknown`, with `unknown` covering detached HEAD, missing HEAD, absent upstream, and
status errors — then aggregates repo states into a workspace-level risk with a deliberate precedence
order. `ws close` and `ws purge` both compute this live and prompt before continuing
(`docs/dev/spec/commands/ws/close.md`, `.../purge.md`); `ws purge` requires an extra confirmation and
refuses to run non-interactively without an explicit force flag.

That covers five of the baseline's seven states. The remaining two are handled elsewhere rather than
missing: `stale` is `doctor`'s territory (broken repo links, stale locks, unfinished journals), and
`archived` is workspace lifecycle status in `.kra.meta.json`, not a risk state. That split is
correct — risk is a Git fact, staleness is an operational fact.

Beyond the baseline: `ws close` writes a lifecycle journal under
`.kra/state/operations/ws-close/<id>.json` and refuses to start a second close over an unfinished
one, so an interrupted close is resumable rather than a half-state. The baseline asked for
workspaces that can represent `created` / `bootstrap-failed` / `ready` / `remove-pending` instead of
pretending multi-command operations are atomic; `kra` already built that discipline for close.
Bootstrap should reuse it rather than invent a second mechanism.

## 4. Monorepo and polyrepo — Met, without `focus`

One workspace holds N repos under `repos/<alias>/`, each an independent worktree recorded in
`.kra.meta.json`'s `repos_restore[]` with `repo_uid`, `repo_key`, `remote_url`, `alias`, `branch`,
`base_ref` (`docs/dev/spec/concepts/workspace-meta-json.md`). A monorepo task is the N=1 case. No
special-casing, exactly as the baseline wanted.

Gap: no `focus` path. A task touching one package inside a monorepo cannot record that fact, so
navigation, scoped bootstrap, agent cwd, and status presentation all operate at repository
granularity. This is cheap to add — one optional string per repo entry in `repos_restore[]` — and it
must not create extra worktrees. Listed as an open question in the original notes; it is really a
small feature request, and it becomes more valuable once hooks exist (scoped bootstrap).

## 5. Efficient storage — Met

`$KRA_HOME/repo-pool/<host>/<owner>/<repo>.git` holds one bare repository per remote
(`internal/paths/paths.go:13`). `gitutil.EnsureBareRepoFetched` (`internal/gitutil/gitutil.go:124`)
clones `--bare` once, pins `remote.origin.fetch`, resolves `origin/HEAD`, and thereafter only
`fetch --prune`s. Worktrees are added from the bare repo (`internal/cli/ws_reopen.go:447`), so N
tasks against one repository share one object database. No full clone per task — the baseline's
requirement, met structurally rather than by convention.

`repo gc` (`docs/dev/spec/commands/repo/gc.md`) reclaims pool space behind four safety gates: no
current-root registration, no workspace metadata reference, no live worktrees, and no reference from
any other known root in the root registry. Cross-root awareness is a detail most implementations
would get wrong.

Nothing is claimed or attempted for the expensive half — `node_modules/`, `.venv/`, `target/`,
build outputs. The baseline's position is that these should be rebuilt from lockfiles against shared
content-addressed caches (pnpm store, uv cache, Cargo registry, Go module cache) rather than shared
mutably across worktrees. That is a hook concern, which lands back on gap #1: `kra` cannot implement
the cheap-and-correct strategy because it cannot run the rebuild step.

## 6. Machine interface — Mostly met

- One shared JSON envelope — `ok`, `action`, `workspace_id`, `result`, `warnings`, `error{code,message}` —
  in `internal/cli/json_output.go`, specified in `docs/dev/spec/concepts/output-contract.md`, with a
  stable error-code vocabulary that clients are told to branch on instead of parsing messages
  (`invalid_argument`, `not_found`, `conflict`, `internal_error` implemented; `permission_denied`
  specified but not yet emitted).
- Stable exit codes: `0` ok, `2` usage, `3` error (`internal/cli/cli.go:12-15`), used consistently
  (e.g. `doctor` returns `3` on any error finding).
- `--format json` on ~30 command files, including every lifecycle command.
- `--dry-run --format json` preflight on `ws close`, `ws reopen`, `ws purge`
  (`docs/dev/spec/commands/ws/dry-run.md`) — the baseline never asked for this and it is the single
  best affordance for an agent about to do something destructive.
- Deterministic paths and a canonical on-disk state file an external tool can read directly.

Gap: `kra ws status` — the command whose name matches the baseline's `task-status TASK-123 --json` —
has no `--format` flag at all (`internal/cli/cli.go:146` → `runWSStatus`); it renders the
`workspace.md` task view for humans and the cmux Dock. The nearest machine equivalent is
`ws dashboard --format json --workspace <id>`, which returns risk, repo count, coverage, and
context — but not per-repo branch names, ahead/behind counts, bootstrap state, runtime state, or
background processes. So the aggregate is exposed and the detail is not.

## 7. Hook trust — Missing, blocked on #1

Checked-in hook definitions are executable code that arrives with `git pull`. The baseline wants
first-execution approval plus re-approval when the definition changes, with a different policy for
user-local config. There is nothing to approve yet.

Worth noting for the design: `kra` already has a trust-shaped mechanism in
`internal/cli/git_allowlist.go` (staging allowlists so lifecycle commits cannot sweep up unrelated
changes), and `docs/dev/spec/concepts/external-command-execution.md` centralizes how external
processes are run. A hook trust store belongs next to those, not bolted onto the hook runner.

## 8. Runtime adapter abstraction — Partial

The boundary is deliberate, not accidental. There is an interface — `cmux.Client`
(`internal/app/cmux/usecase.go:17`); the state-store spec treats runtime mapping as non-canonical,
degradable data; and the architecture spec states the containment rule outright: "`cmux` is exposed
only via workspace entrypoints (`kra ws open`); no dedicated top-level command group exists"
(`docs/dev/spec/concepts/architecture.md`). Upstream has already decided the runtime must not become
a peer of the workspace model.

But the abstraction is named for its only implementation. `cmux` appears in 27 non-test source files,
including `internal/cli/init.go`, `template_common.go`, `root_migrate.go`, `doctor.go`,
`ws_close.go`, `ws_task.go`, `ws_doc.go`, and dedicated packages `internal/cmuxmap/` and
`internal/cmuxdocs/`. The default workspace template ships `.cmux/dock.json`; `root migrate` exists
to add it to older roots. Runtime state files are named `cmux-workspaces.json` /
`cmux-sessions.json`.

So the coupling is not "cmux is wired into open"; it is "cmux is the vocabulary". Introducing a
second runtime (Herdr) means renaming the port to something runtime-neutral, moving Dock/docs-pane
concepts behind capability queries rather than assuming them, and generalizing the mapping store's
schema and filenames. `ws open` already degrades to directory-open when cmux is unavailable, which
is effectively a third (no-op) adapter and a useful shape to generalize from.

Doing this generically is also what makes it upstreamable rather than fork-only — see
[fork strategy](../decisions/fork-strategy.md).

## 9. Blocking vs background hooks — Missing, blocked on #1

The baseline distinguishes blocking hooks that establish invariants (`uv sync`, migrations) from
background hooks that start disposable activity (dev servers, watchers), and wants the background
ones tracked and cancellable rather than making creation hang.

Neither exists. `kra` does not supervise processes at all — no PID tracking, no allocated-port
record, nothing under `.kra/state/` for running work. The state-store spec's list of runtime files
is mapping data only. Any long-running process today is the terminal runtime's business, which means
it is invisible to `ws status`, survives nothing, and is not stopped on close.

## 10. Archive / reopen — Met

`ws close` archives to `archive/<id>/` preserving `notes/` and `artifacts/` while removing worktrees;
`.kra.meta.json`'s `repos_restore[]` carries enough to reconstruct them; `ws reopen` refetches the
bare repos, recreates branches (tracking `origin/<branch>` when it exists, otherwise branching from
`base_ref`), and re-adds worktrees (`internal/cli/ws_reopen.go:408-447`).

This answers one of the original open questions outright — see
[open questions](../decisions/open-questions.md). Archive detaches worktrees and preserves branches;
task artifacts stay; identity survives. The remaining nuance is that `repos_restore[]` records the
branch, not the exact commit, so reopen restores *the branch as it now is*.

`ws close` also commits a `close-pre: <id>` snapshot of the workspace directory before archiving
(the root is itself Git-managed), with a staging allowlist so unrelated staged work is preserved.

## 11. Trash before delete — Missing

`ws purge` removes worktrees, deletes the corresponding local branches from the pool bare repo, and
deletes the workspace and archive directories outright (`docs/dev/spec/commands/ws/purge.md`). The
guards in front of it are good — always prompts, refuses non-interactive without force, re-checks
risk, and honors a `protection.purge_guard` flag in `.kra.meta.json` — but once past them there is
no undo window and no GC phase. Searching the sources for `trash` returns nothing.

The baseline wants disposable state moved to a trash location first, with permanent deletion
deferred to GC. `repo gc` already establishes the vocabulary and the safety-gate pattern for
pool-side reclamation; a workspace-side equivalent would follow the same shape.

## 12. Doctor / repair — Met

`kra doctor [--format human|json] [--fix --plan|--apply]` covers root layout, metadata readability,
broken workspace↔worktree links in both directions, stale lock files whose owner PID is dead,
root-registry drift, missing baselines, legacy state-file migration, unfinished `ws-close` journals
classified as `resume_ready` / `reset_ready` / `manual_required`, and root hygiene
(`docs/dev/spec/commands/doctor.md`). Exit `3` on any error finding; JSON uses the shared envelope.

This exceeds the baseline. The plan/apply split on `--fix` is the same discipline the baseline
elsewhere calls dry-run preflight.

## 13. Path and branch templates — Partial

Branch names are templated: `workspace.branch.template` with `{{workspace_id}}`, `{{repo_key}}`,
`{{repo_name}}`, precedence `--branch` > template > workspace ID, validated through
`git check-ref-format`, deterministic rendering with no shell/env interpolation
(`docs/dev/spec/concepts/branch-naming-policy.md`). Exactly the right design.

Workspace and worktree *paths* are not templated: `workspaces/<id>/repos/<alias>` is structural
(`docs/dev/spec/concepts/layout.md`). Fixed layout is arguably a feature — it is what makes
filesystem-first root detection and deterministic agent paths work — so this is a gap only if
integrating with an existing directory convention. Low priority; note the trade-off before changing it.

## 14. Custom subcommands — Missing

Dispatch is a fixed `switch` in `internal/cli/cli.go`. There is no `kra <custom>` extension point,
no `kra-*` PATH-lookup convention, and no config-defined command aliases. The baseline lists this
under "strongly desired" as the escape valve that keeps workstation-specific behavior *out* of the
tool — which is the same argument [fork strategy](../decisions/fork-strategy.md) makes for keeping
OpenSpec/AXIS-specific behavior in a higher layer. Without an extension point, that behavior has
nowhere to go but into the fork.

## 15. Shell navigation protocol — Met

`kra shell init [<shell>] [--with-completion]` emits an eval-able wrapper that sets
`KRA_SHELL_ACTION_FILE` and applies post-exec actions such as `cd '<path>'` to the parent shell, for
zsh/bash/sh/fish (`docs/dev/spec/commands/shell/init.md`). This is what keeps `ws close` from
stranding you in a deleted directory, and what makes directory-fallback `ws open` actually navigate.

## 16. Bounded parallelism — Partial (not generalized)

A correct bounded worker pool already exists. `cmux.Service.openConcurrent`
(`internal/app/cmux/usecase.go:145`) runs a fixed worker count over a job channel, aggregates results
into their original order, and reports partial failures per target. It is exposed as
`kra ws open --multi-select --concurrency <n>`, defaults to `min(targets, max(2, GOMAXPROCS))` when
unspecified, and is pinned to `1` for `--all` (`docs/dev/spec/commands/ws/open.md`). It shipped as a
deliberate backlog item — `CMUX-010: bounded parallel execution with result aggregation and
partial-failure reporting`.

So the baseline's requirement — bounded concurrency, cancellation, clear errors, "not merely 'use
goroutines everywhere because apparently computers enjoy chaos'" — is already understood and
implemented here.

Gap: it is not a shared primitive. The other fan-outs do not use it. `ws add-repo` prefetches
repositories with one goroutine per item and a bare `sync.WaitGroup`
(`internal/cli/ws_add_repo.go:530-570`), as do `repo pool add` and `ws import all` — no semaphore, no
`--concurrency` flag, no shared policy. Adding twenty repositories launches twenty simultaneous
`git fetch`es. Context cancellation threads through; the bound and retry policy do not.

This reframes the fix: extract the proven pool from `internal/app/cmux/` into a layer-appropriate
shared package and retrofit the other call sites — not design a new one. Worth doing before hooks
land, since bootstrap is heavier than fetch.

---

## Reading of the result

Against the baseline `kra` is a strong state-and-safety substrate with no extensibility surface.
The things that are hard to retrofit — canonical filesystem-first state, a task identity that
survives archive/reopen, live risk classification, recoverable multi-phase operations, a shared bare
pool with cross-root GC, a consistent JSON envelope with dry-run preflight — are already built and
specified. The things that are missing are, with the exception of the runtime rename, *additive*:
hooks, hook trust, a bounded worker pool, `ws exec`, workspace trash/GC, `--format json` on
`ws status`, a `focus` field.

That is the favorable shape. The fork does not need to fight the architecture to get what the
baseline asks for.

Sequencing is in [roadmap.md](roadmap.md).
