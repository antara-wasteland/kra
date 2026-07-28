---
title: "Core Insights"
status: draft
---

# Core Insights

## 1. The task is the unit of work

The most important architectural realization was that the repository is not always the correct top-level abstraction.

Repository-centric model:

```text
repository
└── branch
    └── worktree
```

Task-centric model:

```text
task
├── backend worktree
├── frontend worktree
├── infra worktree
├── notes
├── artifacts
└── agent/process sessions
```

A monorepo task happens to contain one repository worktree. A polyrepo task may contain several.

This allows one mental model for both.

## 2. Worktrees should remain a Git primitive

Do not invent a second VCS abstraction.

A workspace manager should compose native Git worktrees and expose better lifecycle semantics around them.

Git remains authoritative for:

- commits,
- branches,
- refs,
- merges,
- rebases,
- remotes,
- object storage.

## 3. Workspace lifecycle is different from Git workflow

Useful lifecycle events include:

```text
pre-create
post-create
pre-enter
post-enter
pre-remove
post-remove
```

Potentially:

```text
pre-archive
post-archive
```

These are orthogonal to whether the project uses squash merges, merge commits, stacked PRs, or some future agent workflow.

## 4. Repository-owned bootstrap is the stable contract

Instead of encoding project setup deeply into a particular worktree manager:

```text
scripts/worktree-bootstrap
scripts/worktree-teardown
scripts/verify
```

The manager invokes these contracts.

This keeps the manager replaceable.

## 5. Task state and terminal state should be separate

A task workspace can exist without a terminal.

A terminal workspace can be recreated from task state.

Therefore:

```text
task workspace manager (kra) → task/filesystem/worktree ownership
terminal runtime (cmux/Herdr) → terminal/pane/process ownership
```

The terminal should not become the authority for Git worktree lifecycle.

`kra` already holds this line on the storage side: `.kra.meta.json` is canonical workspace state and
the cmux mapping files under `.kra/state/` are explicitly non-canonical runtime data that commands
must degrade past when stale (`docs/dev/spec/concepts/state-store.md`). The remaining coupling is at
the code level, where the runtime is cmux by name rather than by interface — see
[gap analysis](kra/gap-analysis.md#8-runtime-adapter-abstraction--partial).

## 6. Declarative inventory and ephemeral execution are separate

Mani solves long-lived repository inventory/bootstrap.

Task workspaces solve ephemeral execution.

```text
Mani
= what repositories belong to the workstation

Task workspace manager
= what repositories/branches belong to this task right now
```

They can coexist without one replacing the other.
