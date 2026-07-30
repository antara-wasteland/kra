---
title: "Monorepo and Polyrepo"
status: draft
---

# Monorepo and Polyrepo

## Monorepo

A Git worktree checks out the whole repository.

```text
TASK-123/
└── platform/
    ├── apps/web/
    ├── services/api/
    ├── packages/shared/
    └── infra/
```

There is no reason to create one Git worktree per package. Git's unit is the repository.

The workspace layer can still provide package-aware hooks and commands.

For example:

```text
task = TASK-123
repo = platform
focus = services/api
```

`focus` is navigation/execution metadata, not another worktree.

## Polyrepo

A task may span several repositories:

```text
TASK-456/
├── frontend/
├── backend/
└── infra/
```

Each child can be a worktree backed by its repository's shared Git store.

This is where a task-first system is structurally stronger than a repository-first worktree manager: the workspace, not the repository, is the container.

## Unified model

```text
Workspace
├── task key
├── metadata
├── artifacts
└── repositories[]
    ├── source
    ├── branch/ref
    ├── path
    └── lifecycle status
```

A monorepo task has `repositories.length == 1`.

A polyrepo task has `repositories.length > 1`.

No special architecture is required.

## Parallel operations

Polyrepo creation should not serially:

```text
fetch repo A
then fetch repo B
then fetch repo C
```

A Go implementation can naturally coordinate bounded parallel operations:

```text
fetch A ─┐
fetch B ─┼─> attach worktrees -> bootstrap
fetch C ─┘
```

The important part is bounded concurrency, cancellation, clear errors, and idempotent retry, not merely "use goroutines everywhere because apparently computers enjoy chaos."
